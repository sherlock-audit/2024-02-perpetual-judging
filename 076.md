Loud Steel Pelican

medium

# Global deposit cap cannot reflect per-asset liquidity

## Summary

While there is a cap for total collateral deposited to the Vault, there is no cap for total absolute position size for each market. The total Vault deposit cap cannot reflect the external liquidity of all markets at once.

A market that has a total position size of too large will bring difficulties and may discourage liquidations.

## Vulnerability Detail

While there is a deposit cap for the Vault (and thus caps the total margin amount), there is no cap for how much positions can be opened per market. That simply means the entire market (or the majority of it) can choose to open positions against the most illiquid supported asset on the chain.

This alone does not pose a problem yet for a perpetual exchange, but due to the protocol's reliant on external liquidity, we show why liquidations may be discouraged on illiquid assets.

### On liquidation

The key that incentivizes liquidating is the guaranteed profit without risk exposure: The liquidator must be able to immediately close their position.
- If the liquidator keeps their position, their margin is exposed to that asset's price. Considering that the price movement was already unfavorable, pushing the position to being liquidated to begin with, then holding the position in their place is a very risky move.

Currently, the only way to close a position right away is through opening an opposite position on the SpotHedgeBasedMaker (orders towards Oracle Maker must go through the relayer). However, Different assets on UniV3 has different liquidity/TVL. Since the SpotHedgeBasedMaker performs a swap in response to an order, illiquid assets will incur more price impact when swapped.

Sending a closing order to the relayer for the Oracle Maker ASAP is also an option. However the latency between when the liquidation tx is confirmed, when the relayer receives an order, and when the tx actually goes through, may cause the PnL to outweight the liquidation bonus. The added layer of risk will discourage liquidators from using this method.
  - In case of a mass liquidation (e.g. a sharp price drop), the Oracle Maker's quoted price may have a significant spread, which will further discourage liquidators due to the added risk due to the potential loss.

Another factor to consider is that this issue fortifies itself. Since there is no cap per market, traders are actually encouraged to open positions on assets with less liquidity, since it will likely more difficult to get liquidated in time, in case of a sharp market movement.
- When the market cannot handle the liquidations, one can actually attack the protocol by opening a position against themselves on another account (p2p positions), with one side making a profit, the other side making a loss but won't get liquidated on time anyway.

To sum up, the total Vault deposit cap cannot reflect external liquidity for all supported assets. The majority of margins may be used up against more illiquid assets (e.g. LINK) rather than more liquid assets (e.g. WETH), which will cause problems when a typical sharp price movement occurs.

## Impact

Position size per market is uncapped, which will leave certain markets in a risky state during sharp market movements. When the external market fails to handle the liquidation pressure, bad debt occurs.

## Code Snippet

There is no position cap as a market-specific risk param:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/config/Config.sol#L27-L32

## Tool used

Manual Review

## Recommendation

Introduce a cap for absolute total position sizes per market, with the cap being a risk param, reflecting their volatility and on-chain liquidity. 
- Since the total signed position sizes (short and long) of a market is always zero, just checking that one side does not exceed the cap is sufficient.
