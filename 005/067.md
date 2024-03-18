Massive Blush Wasp

medium

# Funding rate not accounting for unrealized PnL on `OracleMaker` leads to inaccurate funding fee

## Summary

The margin of `OracleMaker` is key to calculating the funding rate. Essentially, a larger margin means a lower funding rate, and a smaller margin results in a higher rate. However, the margin is not taking into account the unrealized PnL of `OracleMaker`, and this will cause a great variance in the funding fee if the maker hasn't settled a position in a while.

## Vulnerability Detail

The purpose of the funding fee is to reward positions that are in the same direction as `OracleMaker` and make pay for positions that are in the opposite direction. The funding rate is calculated as the following:

$$
\begin{aligned}
maxCapacity &:= \frac{OracleMakerMargin}{minMarginRatio}
\\\\\\
imbalanceRatio &:= \frac{abs(OracleMakerOpenNotional)^{exp}}{maxCapacity} 
\\\\\\
fundingRate &:= fundingFactor \times imbalanceRatio
\end{aligned}
$$

In the actual implementation of the funding rate, the margin of `OracleMaker` does not include the pending margin (funding fee and borrowing fee) because it would result in an infinite loop dependency. This is stated in the code comments:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L120-L123
```solidity
// we can only use margin without pendingMargin as totalDepositedAmount
// since pendingMargin includes pending borrowingFee and fundingFee,
// it will be infinite loop dependency
uint256 totalDepositedAmount = uint256(_getVault().getSettledMargin(marketId, fundingConfig.basePool));
```

_*in the code, `totalDepositedAmount` is the variable that holds the `OracleMaker` margin._ 

However, the implementation is only getting `settledMargin` for calculating the `OracleMaker` margin, forgetting about the unrealized PnL. **The unrealized profits and losses are key for the margin calculation because they add or subtract the amount of profits/losses accumulated since the last position has been settled on the maker.** 

When we don't include the unrealized PnL on the `OracleMaker` margin, we're not including the profits or losses that the maker has incurred since the last position was settled on `OracleMaker`. If the price has changed since then, these extra profits/losses will not be accounted for on the funding rate, resulting in an inaccurate rate, potentially damaging the risk exposure on the `OracleMaker`. 

## Impact

When `OracleMaker` hasn't settled a position in a while and prices have changed, the unrealized PnL of `OracleMaker` won't be taken into account in the calculation of the funding fee. This will result in an inaccurate funding rate that will hurt the risk exposure of the `OracleMaker`. 

For example, when `OracleMaker` has some unrealized losses, those won't be accounted for in the margin when calculating the funding rate. Therefore, the final funding rate will be lower than it should be, charging a lower fee for positions that are opposite to the `OracleMaker`, not incentivizing enough to close them. The main purpose of the funding fee is to reduce the risk exposure on `OracleMaker` (not too long or too short), when the funding rate is not high enough that will hurt the risk exposure of `OracleMaker`, causing losses to the LPs. 

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L123

## Tool used

Manual Review

## Recommendation

To mitigate this issue is crucial to account for the unrealized PnL of `OracleMaker` in the calculation of the funding rate. Below is the pseudo-code of a possible implementation:

```diff
    function getCurrentFundingRate(uint256 marketId) public view returns (int256) {
        // ...

        // we can only use margin without pendingMargin as totalDepositedAmount
        // since pendingMargin includes pending borrowingFee and fundingFee,
        // it will be infinite loop dependency
        uint256 totalDepositedAmount = uint256(_getVault().getSettledMargin(marketId, fundingConfig.basePool));
+       int256 unrealizedPnL = _getVault().getUnrealizedPnl(marketId, fundingConfig.basePool, price);
+       totalDepositedAmount += unrealizedPnL;
        uint256 maxCapacity = FixedPointMathLib.divWad(
            totalDepositedAmount,
            uint256(OracleMaker(fundingConfig.basePool).minMarginRatio())
        );

        // ...
    }
```

The actual mitigation should include the correct type conversions, as well as the correct pyth price to get the unrealized PnL. 