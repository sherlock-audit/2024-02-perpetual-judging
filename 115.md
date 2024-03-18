Amusing Cream Buffalo

medium

# Attackers can create positions that have no incentive to be liquidated

## Summary

There is no incentive to liquidate tiny positions, which may lead to insolvency


## Vulnerability Detail

A well-funded attacker (e.g. a competing exchange) can create millions of positions where each position's total open notional (and thus the liquidation fee given when closing the position) is smaller than the gas cost required to liquidate it if there's a loss.


## Impact

Lots of small losses are equivalent to one large loss, which will lead to bad debt that the exchange will have to cover in order to allow others to withdraw from the PnL pool


## Code Snippet

There is [no](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/config/Config.sol) minimum position size, and the liquidation incentive is based on the total open notional (average cost to open):
```solidity
// File: src/clearingHouse/LibLiquidation.sol : LibLiquidation.getPenalty()   #1

73        /// @notice penalty = liquidatedPositionNotionalDelta * liquidationPenaltyRatio, shared by liquidator and protocol
74        /// liquidationFeeToLiquidator = penalty * liquidation fee ratio. the rest to the protocol
75        function getPenalty(
76            MaintenanceMarginProfile memory self,
77            uint256 liquidatedPositionSizeDelta
78        ) internal view returns (uint256, uint256) {
79            // reduced percentage = toBeLiquidated / oldSize
80            // liquidatedPositionNotionalDelta = oldOpenNotional * percentage = oldOpenNotional * toBeLiquidated / oldSize
81            // penalty = liquidatedPositionNotionalDelta * liquidationPenaltyRatio
82            uint256 openNotionalAbs = self.openNotional.abs();
83 @>         uint256 liquidatedNotionalMulWad = openNotionalAbs * liquidatedPositionSizeDelta;
84            uint256 penalty = liquidatedNotionalMulWad.mulWad(self.liquidationPenaltyRatio) / self.positionSize.abs();
85            uint256 liquidationFeeToLiquidator = penalty.mulWad(self.liquidationFeeRatio);
86            uint256 liquidationFeeToProtocol = penalty - liquidationFeeToLiquidator;
87            return (liquidationFeeToLiquidator, liquidationFeeToProtocol);
88:       }
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/LibLiquidation.sol#L73-L88

Furthermore, even if somehow gas costs were free, the `mulWad()` used to calculate the penalty/fee rounds _down_ the total penalty as well as the portion that the liquidator gets, so one-wei open notionals will have a penalty payment of zero to the liquidator


## Tool used

Manual Review


## Recommendation

Have a minimum total open notional for positions, to ensure there's a large enough fee to overcome liquidation gas costs. Also round up the fee
