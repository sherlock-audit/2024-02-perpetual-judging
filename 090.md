Loud Steel Pelican

high

# If both makers make a loss, closing positions may be impossible, discouraging liquidations

## Summary

In the case that both makers are in a loss position, the `_checkMinMarginRatio()` will trigger most of the time, preventing position closes, which will discourage liquidations.

## Vulnerability Detail

Consider the following scenario:
- Both Makers are in a short position.
  - It follows that the majority of the market are in a long position.
- There are some users who are still bearish, and takes the short position.
- Price goes up, creating profits for long, and losses for short.
- Both Makers incur a loss due to being in a short. Most trades against them will fail due to the `minMarginRatio` safety check.
- Short users' position becomes unhealthy and liquidatable.
- However, no liquidation bots are willing to liquidate, as it is impossible to close out the position due to lack of available makers. Holding a recently liquidated position is a highly risky move, discouraging for liquidators.

Then the short positions (especially high-leverage ones) will likely go underwater before liquidators start to act. Note that the same can be said for the other trading direction as well.

Now, there is no problem yet if no traders can realize their profits or losses due to the lack of makers. However, we present several scenarios where an already-underwater short trader can close their position and realize their loss:
- Additional liquidity is added to either makers, which opens up more trading opportunities against them. Then liquidator bots will be the first to act against such movement, liquidating those underwater short positions and force a bad debt realization.
- Long traders can technically close their position and realize profits by liquidating the short positions manually. However this cannot reliably prevent bad debt (and in fact may force-realize bad debt) because:
  - Regular traders normally don't run bots that monitor *other* users' positions. By the time those traders realize the Maker DoS and are able to find liquidatable positions, those positions might have been already underwater.
  - Traders can choose when to close out their position. They are not obliged and have no incentives to force-close positions for the good of the protocol, and can choose when to realize their profits without any regards to how insolvent the other position is, especially when they are already holding a highly profitable position.

The end result is that, since liquidators are not able to profitably operate, unhealthy positions cannot be liquidated, and bad debt is accrued.

## Impact

If both makers are in a losing position (or any other situation that results in being unable to add positions against them), closing other positions may be impossible, discouraging liquidations.

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L377-L382

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L685-L689

## Tool used

Manual review

## Recommendation

Introduce a maker specifically for closing liquidated positions and not for any other positions, to account for the case when both makers are out of commission. They can take the exact design as one of the current makers, preferrably Oracle Maker (two Spot Hedge Makers on the same market would share the same UniV3 liquidity, which defeats the purpose). 
- An implementation detail could be that, in `ClearingHouse.liquidate()`, the liquidator can choose to force the position to the liquidation maker with a premium (part of liquidation fee, and/or the same base price spread as with an Oracle Maker), instead of having to carry the position themselves.

This is known as a "backstop" pool in other protocols, notably some lending protocols.
