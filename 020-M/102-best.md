Brilliant Aegean Dinosaur

medium

# There may be excess funds in the PnL pool or bad debt due to the funding fee.

## Summary
There are two types of `makers`: `OracleMaker` and `SpotHedgeBaseMaker`, where `LPs` can deposit funds.
`Traders` can then execute their `orders` against these `makers`.
To incentivize `LPs`, several mechanisms exist for these `makers` to profit.
One is the `borrowing fee`, which both `makers` can benefit from.
Another is the `funding fee`, which specifically benefits `OracleMaker`.
The `funding fee` incentivizes users to maintain `positions` with the same direction of `OracleMaker`.
However, due to the `funding fee`, there may be excess funds in the `PnL pool` or occurrences of `bad debt`.
## Vulnerability Detail
Typically, in most `protocols`, the generated `yields` are totally distributed to users and the `protocol` itself.
In the `Perpetual` protocol, all `borrowing fees` from `payers` are solely distributed to `receivers`, which are whitelisted by the `protocol`.
However, not all `funding fees` are distributed, or there may be a lack of `funding fees` available for distribution.
The current `funding rate` is determined based on the current `position` of the `base pool`(`OracleMaker`).
```solidity
function getCurrentFundingRate(uint256 marketId) public view returns (int256) {
    uint256 fundingRateAbs = FixedPointMathLib.fullMulDiv(
        fundingConfig.fundingFactor,
        FixedPointMathLib
            .powWad(openNotionalAbs.toInt256(), fundingConfig.fundingExponentFactor.toInt256())
            .toUint256(),
        maxCapacity
    );
}
```
Holders of `positions` with the same direction of the `position` of the `OracleMaker` receive `funding fees`, while those with `positions` in the opposite direction are required to pay `funding fees`.
The amount of `funding fees` generated per second is calculated as the product of the `funding rate` and the sum of `openNotionals` of `positions` with the opposite direction.
Conversely, the amount of `funding fees` distributed per second is calculated as the product of the `funding rate` and the sum of `openNotionls` of `positions` with the same direction of the `position` of the `OracleMaker`.
```solidity
function getPendingFee(uint256 marketId, address trader) public view returns (int256) {
    int256 fundingRate = getCurrentFundingRate(marketId);
    int256 fundingGrowthLongIndex = _getFundingFeeStorage().fundingGrowthLongIndexMap[marketId] +
        (fundingRate * int256(block.timestamp - _getFundingFeeStorage().lastUpdatedTimestampMap[marketId]));
    int256 openNotional = _getVault().getOpenNotional(marketId, trader);
    int256 fundingFee = 0;
    if (openNotional != 0) {
        fundingFee = _calcFundingFee(
            openNotional,
            fundingGrowthLongIndex - _getFundingFeeStorage().lastFundingGrowthLongIndexMap[marketId][trader]
        );   // @audit, here
    }
    return fundingFee;
}
```
All `orders` are settled against `makers`, meaning for every `long position`, there should be an equivalent `short position`.
While we might expect the sum of `openNotionals` of `long positions` to be equal to the `openNotionals` of `short positions`, in reality, they may differ.

Suppose there are two `long positions` with `openNotional` values of `S` and `S/2`.
Then there should be two `short positions` with `openNotianal` values of `-S` and `-S/2`.
If the holder of the first `long position` cancels his `order` against the second `short position` with `-S/2`, the `openNotional` of the `long position` becomes `0`, and the second `short position` becomes a `long position`.
However, we can not be certain that the `openNotional` of the new `long position` is exactly `S/2`.
```solidity
function add(Position storage self, int256 positionSizeDelta, int256 openNotionalDelta) internal returns (int256) {
    int256 openNotional = self.openNotional;
    int256 positionSize = self.positionSize;

    bool isLong = positionSizeDelta > 0;
    int256 realizedPnl = 0;

    // new or increase position
    if (positionSize == 0 || (positionSize > 0 && isLong) || (positionSize < 0 && !isLong)) {
        // no old pos size = new position
        // direction is same as old pos = increase position
    } else {
        // openNotionalDelta and oldOpenNotional have different signs = reduce, close or reverse position
        // check if it's reduce or close by comparing absolute position size
        // if reduce
        // realizedPnl = oldOpenNotional * closedRatio + openNotionalDelta
        // closedRatio = positionSizeDeltaAbs / positionSizeAbs
        // if close and increase reverse position
        // realizedPnl = oldOpenNotional + openNotionalDelta * closedPositionSize / positionSizeDelta
        uint256 positionSizeDeltaAbs = positionSizeDelta.abs();
        uint256 positionSizeAbs = positionSize.abs();

        if (positionSizeAbs >= positionSizeDeltaAbs) {
            // reduce or close position
            int256 reducedOpenNotional = (openNotional * positionSizeDeltaAbs.toInt256()) /
                positionSizeAbs.toInt256();
            realizedPnl = reducedOpenNotional + openNotionalDelta;
        } else {
            // open reverse position
            realizedPnl =
                openNotional +
                (openNotionalDelta * positionSizeAbs.toInt256()) /
                positionSizeDeltaAbs.toInt256();
        }
    }

    self.positionSize += positionSizeDelta;
    self.openNotional += openNotionalDelta - realizedPnl;

    return realizedPnl;
}
```
Indeed, the `openNotional` of the new `long position` is determined by the current `price`.
Consequently, while the `position size` of this new `long position` will be the same with the old second `long position` with an `openNotional` value of `S/2`, the `openNotional` of the new `long position` can indeed vary from `S/2`.
As a result, the sum of `openNotionals` of `short positions` can differ from the sum of `long positions`.
There are numerous other scenarios where the sums of `openNotionals` may vary.

I believe that the developers also thought that the `funding fees` are totally used between it's `payers` and `receivers` from the below code.
```solidity
/// @notice positive -> pay funding fee -> fundingFee should round up
/// negative -> receive funding fee -> -fundingFee should round down
function _calcFundingFee(int256 openNotional, int256 deltaGrowthIndex) internal pure returns (int256) {
    if (openNotional * deltaGrowthIndex > 0) {
        return int256(FixedPointMathLib.fullMulDivUp(openNotional.abs(), deltaGrowthIndex.abs(), WAD));
    } else {
        return (openNotional * deltaGrowthIndex) / WAD.toInt256();
    }
}
```
They even took `rounding` into serious consideration to prevent any shortfall of `funding fees` for distribution.
## Impact
Excess `funding fees` in the `PnL pool` can arise when the sum of `openNotionals` of the `payers` exceeds that of the `receivers`.
Conversely, `bad debt` may occur in other cases, leading to a situation where users are unable to receive their `funding fees` due to an insufficient `PnL pool`.
It is worth to note that other `yields`, such as the `borrowing fee`, are entirely utilized between it's `payers` and `receivers`.
Therefore, there are no additional `funding sources` available to address any shortages of `funding fees`.
## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L133-L139
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L89-L102
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/LibPosition.sol#L45-L48
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L183-L191
## Tool used

Manual Review

## Recommendation
We can calculate `funding fees` based on the `position size` because the sum of the `position sizes` of `long positions` will always be equal to the sum of `short positions` in all cases.