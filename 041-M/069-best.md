Massive Blush Wasp

medium

# The position rate on `OracleMaker` is not including unrealized profits, which will penalize extra hard traders that may increase long/short exposure

## Summary

The `OracleMaker` has a pricing system to penalize traders that increase the maker's position by offering a slightly worse price than the market price. This system is designed to discourage traders that aim to increase the long/short exposure on the maker. The mechanism, however, is flawed because it doesn't account for unrealized profits on the maker's margin, penalizing extra hard traders that increase the maker's position. 

## Vulnerability Detail

When a trader wants to settle an order with `OracleMaker`, the function `OracleMaker.fillOrder` will be called. This function will calculate the price with spread, which will be the price that the trader will have its order filled with. Here is the function:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L265-L310

```solidity
    function fillOrder(
        bool isBaseToQuote,
        bool isExactInput,
        uint256 amount,
        bytes calldata
    ) external onlyClearingHouse returns (uint256, bytes memory) {
        uint256 basePrice = _getPrice();
>>      uint256 basePriceWithSpread = _getBasePriceWithSpread(basePrice, isBaseToQuote);

        // ...
    }
```

The function `_getBasePriceWithSpread` will calculate the price with the spread that will be applied to the order settlement. To calculate the spread, it will first calculate the position rate, which is defined by the following formulas:

$$
\begin{aligned}
\\
\text{maxPositionNotional} &:= \frac{\min(\text{unsettledMargin}, \text{accountValue})}{\text{minMarginRatio}} 
\\\\
\text{uncappedPositionRate} &:= -\frac{\text{openNotional}}{\text{maxPositionNotional}}
\\\\
\text{positionRate} &:= 
\begin{cases} 
\min(\text{uncappedPositionRate}, 1) & \text{if } \text{uncappedPositionRate} > 0 \\
\max(\text{uncappedPositionRate}, -1) & \text{otherwise}
\end{cases}
\\\\
\end{aligned}
$$

These formulas are applied to the function `OracleMaker._getPositionRate`:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L411-L437

```solidity
    function _getPositionRate(uint256 price) internal view returns (int256) {
        IVault vault = _getVault();
        uint256 marketId_ = _getOracleMakerStorage().marketId;
        int256 accountValue = vault.getAccountValue(marketId_, address(this), price);
        int256 unrealizedPnl = vault.getUnrealizedPnl(marketId_, address(this), price);
        int256 unsettledMargin = accountValue - unrealizedPnl;
        int256 collateralForOpen = FixedPointMathLib.min(unsettledMargin, accountValue);
        // TODO: use positionMarginRequirement
        //int256 collateralForOpen = positionMarginRequirement + freeCollateralForOpen;
        if (collateralForOpen <= 0) {
            revert LibError.NegativeOrZeroMargin();
        }

        int256 maxPositionNotional = (collateralForOpen * 1 ether) / _getOracleMakerStorage().minMarginRatio.toInt256();

        // if maker has long position, positionRate > 0
        // if maker has short position, positionRate < 0
        int256 openNotional = vault.getOpenNotional(marketId_, address(this));
        int256 uncappedPositionRate = (-openNotional * 1 ether) / maxPositionNotional;

        // util ratio: 0 ~ 1
        // position rate: -1 ~ 1
        return
            uncappedPositionRate > 0
                ? FixedPointMathLib.min(uncappedPositionRate, 1 ether)
                : FixedPointMathLib.max(uncappedPositionRate, -1 ether);
    }
```

The issue here is the numerator of $maxPositionNotional$. By setting the numerator to $\min(unsettledMargin, accountValue)$ instead of just $accountValue$, we're not accounting for the unrealized profits of `OracleMaker`. 

In a scenario where `OracleMaker` has some amount of unrealized profits, $maxPositionNotional$ will be a lower value than it should be, and that will lead to $uncappedPositionRate$ being a higher value, resulting in a higher position rate. When the position rate is higher because it doesn't account for unrealized profits, traders that may increase the position size of `OracleMaker` will be offered a price way worse than the market price, leading to losses for those opened positions. 

## Impact

When `OracleMaker` has some amount of unrealized profits on its margin due to price changes, those profits won't be accounted for in the price spread. This will cause the price spread to be bigger than it should be, offering a worse price to traders that are increasing the maker's position. 

When the price spread becomes too big due to these unrealized profits, these traders will be penalized extra hard for trading. This will result in positions being created with a price too distant from the market price, causing losses to those traders when they close their positions. 

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L417

## Tool used

Manual Review

## Recommendation

To mitigate this issue is recommended to include the unrealized profits of the maker in the calculations for the position rate. That will fix the current issue and will offer traders a fair price for opening positions, a price not too distant from the current market price:

```diff

    function _getPositionRate(uint256 price) internal view returns (int256) {
        IVault vault = _getVault();
        uint256 marketId_ = _getOracleMakerStorage().marketId;
        int256 accountValue = vault.getAccountValue(marketId_, address(this), price);
        int256 unrealizedPnl = vault.getUnrealizedPnl(marketId_, address(this), price);
        int256 unsettledMargin = accountValue - unrealizedPnl;
-       int256 collateralForOpen = FixedPointMathLib.min(unsettledMargin, accountValue);
+       int256 collateralForOpen = accountValue;
        // TODO: use positionMarginRequirement
        //int256 collateralForOpen = positionMarginRequirement + freeCollateralForOpen;
        if (collateralForOpen <= 0) {
            revert LibError.NegativeOrZeroMargin();
        }

        // ...
    }
```
