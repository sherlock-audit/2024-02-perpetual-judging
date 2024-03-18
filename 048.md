Delightful Frost Flamingo

medium

# Unsafe typecast can cause incorrect computation for the funding fee

## Summary

The `getCurrentFundingRate` function makes an unsafe type cast of the settled margin of the base pool contract.

## Vulnerability Detail

The variable `totalDepositedAmount` is computed via calling `getSettledMargin` which is just the `margin + unsettledPnL` from the base pool account. This variable can be negative if the base pool account has incurred losses such that `unsettledPnL > margin`. The issue is that this variable is typecasted to uint256 without validating if it is negative or not.

## Impact

In the case of a black swan event where `totalDepositedAmount` is negative then the typecast will cause `maxCapacity` to be computed to an incorrect value which in turns make the computation of the `fundingRate` incorrect.

## Code Snippet

Code snippet of the unsafe type cast:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L123

## Tool used

Manual Review

## Recommendation

Use `toUint256()` instead of a raw typecast.
