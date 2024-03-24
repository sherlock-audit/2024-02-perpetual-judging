Wobbly Cerulean Blackbird

high

# Trader loss on liquidation is completely taken by the trader leading to excessive bad debt and incorrect liquidation incentives

## Summary
Bad debt is completely taken by the trader and the liquidator and protocol always get the full fee.

## Vulnerability Detail
On liquidation, the offset notional is the worth of the size of the position, which will be compared against the liquidatee notional and the trader realizes this loss most likely as bad debt, as can be seen in `ClearingHouse::liquidate()`, [`vault.settlePosition()`](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L183-L192). The liquidator always receives the full liquidation fee based on the trader's `openNotional`, regardless of bad debt and receives the merged position and notional at current prices of the liquidatee. This means that the liquidator will not take any loss, as it receives the notional at current prices, putting all the loss in the trader's size, decreasing the incentive to liquidate the trader as fast as possible.

## Impact
Considerable bad debt creation as the taker realizes the full loss and the liquidator gets the full fee.

## Code Snippet
In `ClearingHouse::liquidate()`, settling the liquidatee and liquidator positions:
```solidity
vault.settlePosition(
    IVault.SettlePositionParams({
        marketId: params.marketId,
        taker: taker,
        maker: params.maker,
        takerPositionSize: result.base,
        takerOpenNotional: result.quote,
        reason: reason
    })
);
```
And in `LibLiquidation::getPenalty()`:
```solidity
uint256 openNotionalAbs = self.openNotional.abs();
uint256 liquidatedNotionalMulWad = openNotionalAbs * liquidatedPositionSizeDelta;
uint256 penalty = liquidatedNotionalMulWad.mulWad(self.liquidationPenaltyRatio) / self.positionSize.abs();
uint256 liquidationFeeToLiquidator = penalty.mulWad(self.liquidationFeeRatio);
uint256 liquidationFeeToProtocol = penalty - liquidationFeeToLiquidator;
```

## Tool used
Vscode
Manual Review

## Recommendation
Reduce the fees if the trader has bad debt resulting from the liquidation. This will also create a greater incentive to act as soon as possible, reducing bad debt.