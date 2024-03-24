Loud Steel Pelican

high

# Mass liquidations of any market generates sell pressure on the same UniV3 pool

## Summary

A liquidation allows a liquidator to steal an unhealthy position from a trader, receiving some liquidation fee as a bonus. For the liquidator to make a profit, they should be able to close a position right after liquidating, to prevent risk exposure.

There are currently two paths to closing a position. We show that both paths holds various level of risks that will discourage liquidators from operating.

## Vulnerability Detail

For the liquidator to make a profit, they should be able to close a position right after liquidating, realizing a PnL of zero and keeping the liquidation fee as a bonus. The PnL when closing against a maker may not necessarily be zero due to some price impacts, but liquidator bots are generally happy to take a liquidation, as long as profit is made.

There are currently two paths to closing a position, corresponding to two makers: Oracle Maker and Spot Hedge Based Maker. We show that both paths holds various level of risks that will discourage liquidators.

### Oracle Maker

Suppose a liquidation were to happen and closed under the Oracle Maker:
- Liquidator forcefully takes the trader's position and liquidation fee.
- Liquidator has to send a tx to the relayer.
- The relayer has to update the price and send a tx.

This is the more risky path to take because:
- The liquidator has to use their own margin, which prevents liquidating large positions.
- Between the time when the position is taken over, versus when the close-position tx is relayed, there could be a price movement that results in a negative PnL that outweights the liquidation fee.
  - The extra relayer fee and gas fees for an additional tx adds to the risk of losing.
  - The Oracle Maker's price spread also adds to the risk, the price impact between the two txs could be large.

Overall, liquidators will be discouraged from closing a liquidation against the Oracle Maker because of the potential risk for a net loss resulting from the liquidation.

### Spot Hedge Based Maker

Suppose a liquidated position is closed under the Spot Hedge Based Maker, the flow could go as follow:
- Liquidator flashloans external funds, deposits them as collateral.
- Liquidator forcefully takes the trader's position and liquidation fee.
- Liquidator immediately closes the position against the Spot Hedge Based Maker.
- Liquidator returns the flashloan amount and realizes profit, or intentionally revert the tx if this turns out unprofitable.

This is the less risky path to take, and is more likely to be chosen by liquidators because:
- The liquidator has the chain's flashloanable TVL at their disposal. This allows liquidation arbitrary large positions without needing initial capital.
- All steps can be done in one tx, which does not allow price movements.
- The loss from price impacts can be controlled - the liquidation can't ever make a loss, they can deny losses by reverting the tx.

Since this path does not require initial capital and has significantly less risk, this will be the method chosen by most, if not all, liquidators.

Now, the problem with this approach is that, UniV3 liquidity on Optimism (in fact, most chains) [revolves around WETH](https://app.uniswap.org/explore/pools/optimism): most pools are of the form WETH/TOKEN. Creating a position against the Spot Hedge Maker involves a swap from the base asset TOKEN to the quote token, which is the collateral token (USDC or USDT, for simplicity let's say USDT). Then a path will always take the form of TOKEN --> WETH --> USDT, or the other way around depending on whether the position was short or long. Note that the pool WETH/USDT is always present, regardless of what TOKEN is.
- An exception is that there is an OP/USDT and OP/USDC pool, so there is another route of TOKEN --> WETH --> OP --> USDT. However this route's liquidity is comparably weaker, and the Spot Maker's route is fixed as an admin-supplied route, not a user-supplied route.

In fact, liquidation of any assets of the same direction (long/short) will also force the swap in the same direction. Liquidation of **any** asset will always generate market pressure on WETH/USDT pool, therefore all it takes is a sharp general market movement (not a single-asset crash/pump) for the WETH/USDT pool to take the pressure from all of the liquidations. This is an extremely common occurence in cryptocurrency, for the general market to experience a sudden fast price movement.

The resulting price impact from the liquidation pressure may make it impossible for the Spot Hedge Maker to swap between the liquidated asset and USDT when a mass liquidation occurs, therefore timely liquidation may not be profitable until positions become underwater.

Note that if a position becomes underwater (margin does not cover the loss) before it is liquidated, it will still get fully liquidated anyway at some point if profitable, leaving the position with zero margin but positive debt.

## Impact

Liquidators are discouraged to operate in fast market movements in either liquidation routes because:
- One route is not guaranteed to profit and may even cause a loss, which is especially true in fast market movements.
- One route guarantees profit, and is desired. However liquidation for any market will put pressure on one specific pool, which may not be able to handle the liquidation pressure from all of the markets at once.

Without liquidations, positions will become insolvent, and the protocol will accrue bad debt.

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L28

## Tool used

Manual review

## Recommendation

A possible fix would be to allow closing a position against Oracle Maker right away if the equivalent position has been used in a liquidation, without needing to go through a relayer (for example by bundle such functionality in `ClearingHouse.liquidate()`), encouraging liquidator bots for being able to guarantee a profit situation.
- The Oracle Maker can receive a portion of the liquidation fee in such a case, if so desired.
