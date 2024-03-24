Brilliant Aegean Dinosaur

medium

# In certain cases, users are unable to settle their orders with the PartialFill trade type.

## Summary
There are 2 `trade` types available:  `FoK` or `PartialFill`.
Users have the option to partially settle their `orders`.
However, in some cases, they can't settle their `orders.
## Vulnerability Detail
A user creates an `order` with the `PartialFill` `trade` type and a size of `S`.
Initially, he settles `50%` of this `order`.
At this point, the `totalFilledAmount` of this `order` has a value of `S/2`.
```solidity
function _fillTakerOrder(
    InternalContext memory context,
    SettleOrderParam memory settleOrderParams
) internal returns (InternalWithdrawMarginParam memory, uint256) {
    _getOrderGatewayV2Storage().totalFilledAmount[takerOrder.getKey()] += settleOrderParams.fillAmount;
}
```
After some time, the user attempts to settle the remaining `50%` of that `order`.
In the `_verifyOrder` function, we check whether there is enough `positionSize` available to settle this `order` if the `action` of the `order` is `ReduceOnly`.
However, the issue arises from not accounting for the previously settled amount.
```solidity
function _verifyOrder(IVault vault, Order memory order, uint256 fillAmount) internal view {
      uint256 totalFilledAmount = getOrderFilledAmount(order.owner, order.id);
      if (fillAmount > openAmount - totalFilledAmount) {
          revert LibError.ExceedOrderAmount(order.owner, order.id, totalFilledAmount);
      }
      if (order.action == ActionType.ReduceOnly) {
          int256 ownerPositionSize = vault.getPositionSize(order.marketId, order.owner);

          if (order.amount * ownerPositionSize > 0) {
              revert LibError.ReduceOnlySideMismatch(order.owner, order.id, order.amount, ownerPositionSize);
          }

          if (openAmount > ownerPositionSize.abs()) {  // @audit, here
              revert LibError.UnableToReduceOnly(order.owner, order.id, openAmount, ownerPositionSize.abs());  
          }
      }
}
```
In the initial settlement, half of the `order` was settled.
After the settlement, the `positionSize` decreases by `S/2` also and there are only `S/2` remaining in the `order`.
Therefore, we should compare `S/2` with the current `positionSize`.
However,  we compare `S` again without accounting for the previously settled amount.
As a result, the current `positionSize` might be lower than `S`, leading to the potential reversal of the settlement.
## Impact
This is a DoS and users can lose funds as gas fees.
## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L336
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L513-L515
## Tool used

Manual Review

## Recommendation
```solidity
function _verifyOrder(IVault vault, Order memory order, uint256 fillAmount) internal view {
      uint256 totalFilledAmount = getOrderFilledAmount(order.owner, order.id);
      if (fillAmount > openAmount - totalFilledAmount) {
          revert LibError.ExceedOrderAmount(order.owner, order.id, totalFilledAmount);
      }
      if (order.action == ActionType.ReduceOnly) {
          int256 ownerPositionSize = vault.getPositionSize(order.marketId, order.owner);

          if (order.amount * ownerPositionSize > 0) {
              revert LibError.ReduceOnlySideMismatch(order.owner, order.id, order.amount, ownerPositionSize);
          }

-          if (openAmount > ownerPositionSize.abs()) { 
+          if (openAmount - totalFilledAmount > ownerPositionSize.abs()) { 
              revert LibError.UnableToReduceOnly(order.owner, order.id, openAmount, ownerPositionSize.abs());  
          }
      }
}
```