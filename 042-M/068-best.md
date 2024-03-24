Massive Blush Wasp

medium

# `MakerReporter` not accounting for unrealized PnL on whitelisted makers leads to inaccurate borrowing fee

## Summary

The margin of the whitelisted makers is key for calculating its utilization rate, which is key for calculating the borrowing rate. The borrowing fee is the fee that all traders will pay to whitelisted makers in exchange for providing liquidity to the market. However, the current calculations of the borrowing rate do not include the unrealized PnL on the maker's margin, leading to an inaccurate borrowing fee when makers haven't settled a position in a while. 

## Vulnerability Detail

The purpose of the borrowing fee is to make traders pay to whitelisted makers in exchange for them providing guaranteed liquidity in all conditions, without taking liquidity from others. The borrowing fee is based on the borrowing rate, which is defined as the following:

$$
\\
borrowingRate := maxBorrowingFeeRate \times utilizationRatio
\\
$$

The variable `maxBorrowingFeeRate` is a governance-defined value and `utilizationRatio` is defined as the weighted average of the utilization ratio of all whitelisted makers (`OracleMaker` and `SpotHedgeBaseMaker`). 

The utilization ratio of whitelisted makers is calculated via a special contract called `MakerReporter`, this is done to avoid manipulations by whitelisted makers. The function that returns the utilization ratio of a whitelisted maker is the following:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/makerReporter/MakerReporter.sol#L35-L63
```solidity
    function getUtilRatioFactor(uint256 marketId, address receiver) external view returns (uint256, uint256) {
        IVault vault = getAddressManager().getVault();
        int256 margin = vault.getMargin(marketId, receiver);
        int256 openNotional = vault.getOpenNotional(marketId, receiver);
        uint256 defaultUtilRatio;
        if (margin <= 0) {
            defaultUtilRatio = WAD;
        } else {
            // margin > 0, can type to uint directly
            uint256 positiveMargin = uint256(margin);
            defaultUtilRatio = FixedPointMathLib.min(WAD, openNotional.abs().divWad(positiveMargin));
        }

        uint256 openNotionalAbs = openNotional.abs();
        uint256 defaultUtilRatioFactor = defaultUtilRatio * openNotionalAbs;

        if (openNotional < 0) {
            return (0, defaultUtilRatioFactor);
        }
        return (defaultUtilRatioFactor, 0);
    }
```

This function calculates the utilization ratio of a whitelisted maker by applying the following formula:

$$
\\
utilizationRatio := \frac{abs(openNotional)}{margin}
\\
$$

However, the margin used to calculate the utilization ratio does not take into account the unrealized PnL. This will cause that when a whitelisted maker has some unrealized profits/losses, the resulting borrowing rate will be inaccurate, making traders pay too much or too little. 

## Impact

When `OracleMaker` or `SpotHedgeBaseMaker` (whitelisted makers) haven't settled a position in a while and the price has changed, the unrealized PnL won't be included in the calculation of the borrowing rate, leading to an inaccurate fee that traders will have to bear with. 

For example, when a whitelisted maker has unrealized profits, its margin will be bigger, which should drive its utilization rate lower, leading to a lower borrowing fee. However, because the borrowing rate is not taking unrealized PnL into account, the borrowing fees will continue to be higher than they should, leading to traders paying too much fees during that period. 

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/makerReporter/MakerReporter.sol#L37

## Tool used

Manual Review

## Recommendation

To mitigate this issue is recommended to include the unrealized PnL in the calculations for the borrowing rate. The following is the proposed fix for `MakerReporter` to mitigate the issue:

```diff
-   function getUtilRatioFactor(uint256 marketId, address receiver) external view returns (uint256, uint256) {
+   function getUtilRatioFactor(uint256 marketId, address receiver, uint256 price) external view returns (uint256, uint256) {
        IVault vault = getAddressManager().getVault();
-       int256 margin = vault.getMargin(marketId, receiver);
+       int256 margin = vault.getAccountValue(marketId, receiver, price);
        int256 openNotional = vault.getOpenNotional(marketId, receiver);
        uint256 defaultUtilRatio;
        // ...
    }
```