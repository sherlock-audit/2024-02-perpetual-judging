Wobbly Cerulean Blackbird

medium

# Traders can open trades or liquidate with INITIAL margin instead of MAINTENANCE due to abusing `LibPosition.isReduceOnly()`

## Summary
Traders can open/close trades or liquidate always using `INITIAL` margin instead of `MAINTENANCE` by opening a position of 1 wei in the opposite side and their intended position in the other.


## Vulnerability Detail
The margin requirement in `_openPositionFor()` , `_closePositionFor()` or `liquidate()` is `INITIAL` if the trader is not reducing the position and `MAINTENANCE` otherwise. A trader may game the system by opening a 1 wei trade before in the opposite intended direction and open with a margin of `MAINTENANCE` instead of `INITIAL`.

## Impact
Traders opening trades with excessive profit or the creation of excessive bad debt due to instant liquidations.

## Code Snippet
[`LibPosition::isReduceOnly()`](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/LibPosition.sol#L67):
```solidity

function isReduceOnly(int256 positionSizeBefore, int256 positionSizeAfter) internal pure returns (bool) {
    if (positionSizeAfter != 0 && positionSizeBefore ^ positionSizeAfter < 0) {
        return false;
    }
    return positionSizeBefore.abs() > positionSizeAfter.abs();
}
```
[`ClearingHouse::_checkMarginRequirement()`](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L365),
```solidity
if (params.isReducing) {
    // Reducing, Closing positions can share the same logic for now.
    int256 freeCollateralForReducingPosition = vault.getFreeCollateralForTrade(
        marketId,
        trader,
        price,
        MarginRequirementType.MAINTENANCE
    );
    if (freeCollateralForReducingPosition < 0) {
        revert LibError.NotEnoughFreeCollateral(marketId, trader);
    }
    return;
}
// else is INITIAL
```
The logic is similar for close trade and liquidations.

## Tool used
Vscode
Manual Review

## Recommendation
Understanding the real intentions of the trader is unclear, so it would be safer to always use the `INITIAL` ratio when changing positions and `MAINTENANCE` when liquidating.