Suave Honeysuckle Wren

high

# Potential revert due to an underflow in `LibLugiaMath.applyDelta()` may cause several issues

## Summary

An arithmetic underflow that causes a revert may happen inside the function `LibLugiaMath.applyDelta()` on line 8 in LugiaMath.sol. This function is vital for different key functionalities in the protocol like liquidations, withdrawals, opening a position etc.


## Vulnerability Detail

`LibLugiaMath.applyDelta()` is invoked from `LibPositionModel.updateBadDebt()` (line 93 LibPositionModel.sol).

**Example:**

Let's assume the following values when `LibPositionModel.updateBadDebt()` is being called:

* `self.badDebt` = 10 (`PositionModelState.badDebt`)
* `oldUnsettledPnl` = -40
    * Note: from `position.unsettledPnl` on line 81 in PositionModelUpgradeable.sol
* `newUnsettledPnl` = -20
    * Note: from line 83 in PositionModelUpgradeable.sol


Now `LibPositionModel.updateBadDebt()` is calculating:

1. Line 80: `delta = oldUnsettledPnl - newUnsettledPnl;` --> `delta = -40 --20 = -20`

1. Line 93: `self.badDebt = LibLugiaMath.applyDelta(10, -20);`

1. Then inside `LibLugiaMath.applyDelta()` on line 8 an arithmetic underflow happens which reverts:
* `return a - uint256(-delta);` --> `return 10 - uint256(--20)` --> `return 10 - 20` --> `underflow revert`


**Traces:**

This should help and illustrate how the function `LibLugiaMath.applyDelta()` may be invoked. Further traces are described in the "Impact" section.

* `ClearingHouse.liquidate()`
    * `Vault.transferMargin()`
        * `PositionModelUpgradeable._settlePnl()`
            * `PositionModelUpgradeable._updateBadDebt()`
                * `LibPositionModel.updateBadDebt()`
                    * `LibLugiaMath.applyDelta()`


* `ClearingHouse.liquidate()`
    * `Vault.settlePosition()`
        * `PositionModelUpgradeable._settlePnl()`
            * `PositionModelUpgradeable._updateBadDebt()`
                * `LibPositionModel.updateBadDebt()`
                    * `LibLugiaMath.applyDelta()`

Note: The traces above are almost identical with the difference that `PositionModelUpgradeable._settlePnl()` is being invoked both from `Vault.settlePosition()` (line 183 ClearingHouse.sol) and from `Vault.transferMargin()` (line 193 ClearingHouse.sol).

## Impact

Due to this issue some transactions may revert:

* liquidate transactions may revert.
* withdraw/fillorder/openPosition transactions may revert:
    * `OracleMaker.withdraw()` subsequently calls `vault.transferMarginToFund()` -> `vault._transferMarginToFund()` -> `PositionModelUpgradeable._settlePnl()` -> `PositionModelUpgradeable._updateBadDebt()` -> `LibPositionModel.updateBadDebt()` -> `LibLugiaMath.applyDelta()`
    * `SpotHedgeBaseMaker._withdraw()` subsequently calls `vault.transferMarginToFund()` which then follows the same trace as shown above in `OracleMaker.withdraw()`.
        * Note: `SpotHedgeBaseMaker._withdraw()` is called from `SpotHedgeBaseMaker.withdraw()` and from `SpotHedgeBaseMaker._fillOrderCallback()`.
    * `ClearingHouse.openPosition()` -> `ClearingHouse._openPositionFor()` -> `ClearingHouse._openPosition()` -> `Vault.settlePosition()` -> `PositionModelUpgradeable._settlePnl()` which then follows the same trace as shown above in `OracleMaker.withdraw()`.
* Fee calculations `BorrowingFee.beforeSettlePosition()` may revert:
    * `BorrowingFee.beforeSettlePosition()` -> `BorrowingFee._settleBorrowingFee()` -> `BorrowingFeeModel._settleTrader_()` -> `LibBorrowingFee.addPayerOpenNotional()` -> `LibLugiaMath.applyDelta()`
        * Note that `LibBorrowingFee.addReceiverOpenNotional()` -> `LibLugiaMath.applyDelta()` may be invoked instead of `LibBorrowingFee.addPayerOpenNotional()`.
    

As a consequence multiple different types of transactions may revert. In the case of liquidations reverting, this issue may render the protocol exposed to potential losses due to the existence of unhealthy positions.

Furthermore a malicious actor may exploit this issue by intentionally forging a position that can't be liquidated.

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/common/LugiaMath.sol#L5-L11

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/LibPositionModel.sol#L93

## Tool used

Manual Review

## Recommendation

Consider mitigating the potential underflow issue inside `LibLugiaMath.applyDelta()`:

```solidity
// LugiaMath.sol
7        if (delta < 0) {
+8           if (delta.abs() > a) {
+9               // @audit: mitigation
+10          }
11           return a - uint256(-delta);
```