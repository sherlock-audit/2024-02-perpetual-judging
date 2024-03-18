Wild Carmine Whale

high

# DoS of SpotHedgeBaseMaker.withdraw() if withdrawnBaseAmount exceeds spotBaseBalance

## Summary
In the `SpotHedgeBaseMaker.withdraw()` there is a check whether `withdrawnBaseAmount ` exceeds `spotBaseBalance`. It is used to calculate how many quote tokens the maker needs to withdraw from vault in terms of base tokens. However, the subsequent check of maker's `positionSize` in the vault would dos all calls to `SpotHedgeBaseMaker.withdraw()` where `withdrawnBaseAmount > spotBaseBalance` due to using the wrong operator, `!=` instead of `==`
## Vulnerability Detail
The reason for revert is the following if-statement:
```solidity
if (withdrawnBaseAmount > spotBaseBalance) {
            if (vault.getPositionSize(_getSpotHedgeBaseMakerStorage().marketId, maker) != 0) { //@audit should be equal to zero otherwise reverts
                revert LibError.NotEnoughSpotBaseTokens(withdrawnBaseAmount, spotBaseBalance);
            } else {
            ...
```
Whenever a maker fills order, it deposits acquired quote tokens from swapping base tokens into vault and transfers them into margin account. This is almost always means that maker's `positionSize` would not be equal to zero, since `fillOrder()` function is a part of `ClearingHouse._openPosition()` which in result settles position for both maker and taker and changes their `positionSize`. This effectively means that any amount that exceeds the spot balance is not withdrawable and the maker's quote tokens would be stuck in the vault unless someone opens a position against a maker with a `isBaseToQuote == false`.
## Impact
DoS
## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L331-L334
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L416-L435

## Tool used

Manual Review

## Recommendation
Change the `!=` to `==`:
```solidity
if (vault.getPositionSize(_getSpotHedgeBaseMakerStorage().marketId, maker) == 0) { 
                revert LibError.NotEnoughSpotBaseTokens(withdrawnBaseAmount, spotBaseBalance);
```