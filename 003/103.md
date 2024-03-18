Bumpy Eggplant Tadpole

high

# OracleMaker._getBasePriceWithSpread returns unfavorable price for itself.

## Summary
`OracleMaker._getBasePriceWithSpread` returns unfavorable price for itself. 

## Vulnerability Detail

### `_getPositionRate`
`OracleMaker._getPositionRate` returns a positive value when the maker is in long position and a negative value when the maker is in short position.
It is because the return value has an opposite sign against `openNotional`.
```solidity
    int256 uncappedPositionRate = (-openNotional * 1 ether) / maxPositionNotional;
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/d39d7be735a3df01fb958542f9473cc5b80af956/perp-contract-v3/src/maker/OracleMaker.sol#L476

### `reservationPrice`

`reservationPrice` has 2 cases.
It leads to the following results according to the code.
```solidity
        int256 spreadRatio = (_getOracleMakerStorage().maxSpreadRatio.toInt256() * positionRate) / 1 ether;
        uint256 reservationPrice = (basePrice * (1 ether - spreadRatio).toUint256()) / 1 ether;
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/d39d7be735a3df01fb958542f9473cc5b80af956/perp-contract-v3/src/maker/OracleMaker.sol#L401-L404

#### Case1. maker's current position size > 0
1. positionRate > 0
2. spreadRatio > 0
3. reservationPrice < basePrice 

#### Case2. maker's current position size < 0
1. positionRate < 0
2. spreadRatio < 0
3. reservationPrice > basePrice

### `_getBasePriceWithSpread`
`OracleMaker._getBasePriceWithSpread` has four cases.

#### Case1. trader short & maker long 
- If maker's current position size > 0, then reservationPrice < basePrice.
  - A maker wants not to increase its long position, so it normally buys at reservationPrice which is cheaper than basePrice.
- If maker's current position size < 0, then reservationPrice > basePrice.
  - A maker wants to increase its long position, so it gives a favor to buy at reservationPrice which is higher than basePrice.

#### Case2. trader long & maker short
- If maker's current position size > 0, then reservationPrice < basePrice.
  - A maker wants to decrease its long position, so it gives a favor to sell at reservationPrice, which is lower than basePrice.
- If maker's current position size < 0, then reservationPrice > basePrice.
  - A maker wants not to increase its short position, so it normally sells at reservationPrice which is higher than basePrice.

Therefore, it is always favorable for the maker to trade at `reservationPrice` in four cases.

## POC
Add this test in [`OracleMakerFillOrderSpec` after setup function](https://github.com/sherlock-audit/2024-02-perpetual/blob/d39d7be735a3df01fb958542f9473cc5b80af956/perp-contract-v3/test/oracleMaker/FillOrder.spec.t.sol#L32) 
and run `forge test --mt testFuzz_fillOrder_B2Q_exactInput_Normal_om_POC`. I explain the test case in the comment.

```solidity
    function testFuzz_fillOrder_B2Q_exactInput_Normal_om_POC() public {
        // @audit Set max spread ratio to 0.1%
        maker.setMaxSpreadRatio(0.1e18);

        uint256 baseAmount = 100;

        _mockAccValue(100 ether);
        // @audit Suppose the maker already has a net short position. Set position rate = -1 ether / 100 ether = -1%.
        _mockOpenNotional(1 ether);
        vm.prank(mockClearingHouse);
        // @audit trader's short order
        (uint256 oppositeAmount, bytes memory callbackData) = maker.fillOrder(
            true, // isBaseToQuote
            true, // isExactInput
            baseAmount,
            ""
        );

        // @audit oppositeAmount should be reservationPrice, not basePrice.
        // This is the original expectedOppositeAmount in testFuzz_fillOrder_B2Q_exactInput_Normal_om.
        // uint256 expectedOppositeAmount = (baseAmount * price) / 1 ether;
        uint256 basePriceAmount = (baseAmount * price) / 1e18; // 123.45e18

        // @audit The calculation is as below.
        // maxSpreadRatio = 1e17 // 0.1%
        // int256 spreadRatio = maxSpreadRatio * positionRate / 1e18 = 1e17 * -1e16 / 1e18 = -1e15;
        // uint256 expectedOppositeAmount = 123.45e18 * (1e18 + 1e15) / 1e18 = 123.57345e18
        uint256 expectedSpreadRatio = 1e15;
        uint256 expectedOppositeAmount = basePriceAmount * (1 ether - 1e15) / 1e18;

        // @audit It should suggest a higher price to buy base token from the trader because this trade decreases the current short position.
        // basePrice * amount = 123.45e18
        // reservationPrice * amount = 123.57345e18
        // The original code fails here because it selects `FixedPointMathLib.min(basePrice, reservationPrice)` and basePrice is lower.
        assertEqDecimal(oppositeAmount, expectedOppositeAmount, 18, "fillOrder oppositeAmount mismatch");
    }

    function _mockOpenNotional(int256 openNotional) private {
        vm.mockCall(
            mockVault,
            abi.encodeWithSelector(IMarginProfile.getOpenNotional.selector, marketId, address(maker)),
            abi.encode(openNotional)
        );
    }
```

## Impact
The target position size of `OracleMaker` is 0, but the current implementation of `OracleMaker._getBasePriceWithSpread` returns a price to increase its position size.
reference: https://perp.notion.site/PythOraclePool-e99a88be051f4bc8be0b1310eb982cd4#ce111965016b46f28e25dc61d2c7c805

## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/d39d7be735a3df01fb958542f9473cc5b80af956/perp-contract-v3/src/maker/OracleMaker.sol#L401-L409

## Tool used

Manual Review

## Recommendation

Update as below.

```diff
-   return
-       isBaseToQuote
-           ? FixedPointMathLib.min(basePrice, reservationPrice)
-           : FixedPointMathLib.max(basePrice, reservationPrice);
+   return reservationPrice;
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/d39d7be735a3df01fb958542f9473cc5b80af956/perp-contract-v3/src/maker/OracleMaker.sol#L405-L408