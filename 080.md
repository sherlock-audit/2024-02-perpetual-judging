Mythical Orchid Buffalo

medium

# Malicious orders will prevent the batch execution of limit orders

## Summary
The purpose of the `OrderGatewayV2` contract is to execute off-chain collected limit orders on-chain. In `OrderGatewayV2`, orders can only be settled by whitelisted matchers. Consequently, when executing orders, whitelisted matchers should perform batch executions rather than individually executing each order. If a revert occurs during the execution of orders, it would result in the failure of the batch execution and affect the remaining unexecuted orders.

## Vulnerability Detail
The purpose of `OrderGatewayV2.settleOrder()` is to execute limit orders in batch. Within the for loop, the protocol constructs taker order data and maker order data. 
```solidity
 for (uint256 i = 0; i < settleOrderParamsLength; i++) {
            SettleOrderParam memory settleOrderParam = settleOrderParams[i];
            uint256 marketId = settleOrderParam.signedOrder.order.marketId;
            context.imRatio = context.config.getInitialMarginRatio(marketId);
            context.oraclePrice = _getPrice(context.pythOracleAdapter, context.config, marketId);

            (InternalWithdrawMarginParam memory takerWithdrawMarginParam, uint256 takerRelayFee) = _fillTakerOrder(
                context,
                settleOrderParam
            );

            (
                bytes memory makerData,
                InternalWithdrawMarginParam memory makerWithdrawMarginParam,
                uint256 makerRelayFee
            ) = _fillMakerOrder(context, settleOrderParam);


```

It then calls `clearingHouse.openPositionFor()` to set the corresponding positions. 

```solidity
      _openPosition(
                InternalOpenPositionParams({
                    clearingHouse: clearingHouse,
                    settleOrderParam: settleOrderParam,
                    makerData: makerData,
                    takerRelayFee: takerRelayFee,
                    makerRelayFee: makerRelayFee
                })
            );

```

In the `_fillTakerOrder()` function, the protocol first requires users to deposit funds into the margin before proceeding with the execution. If the user's funds are insufficient at this point, the transaction will fail.
```solidity
  function _fillTakerOrder(
        InternalContext memory context,
        SettleOrderParam memory settleOrderParams
    ) internal returns (InternalWithdrawMarginParam memory, uint256) {
        Order memory takerOrder = verifyOrderSignature(settleOrderParams.signedOrder);

        _verifyOrder(context.vault, takerOrder, settleOrderParams.fillAmount);

        // deposit margin for taker
        _transferFundToMargin(context.vault, takerOrder);
```

The issue here is that if a malicious actor creates a limit order and immediately transfers the funds away, by the time the whitelisted matchers attempt to execute the order, a revert will occur due to insufficient funds. 
```solidity
   function _transferFundToMargin(IVault vault, Order memory order) internal {
        bytes32 orderKey = order.getKey();
        if (!_getOrderGatewayV2Storage().isMarginTransferred[orderKey] && order.marginXCD > 0) {
            vault.transferFundToMargin(order.marketId, order.owner, order.marginXCD);
            _getOrderGatewayV2Storage().isMarginTransferred[orderKey] = true;
        }
    }


```

As a result, not only the affected order but also other orders in the same batch cannot proceed with execution. Additionally, the protocol does not mark the executed orders, so problematic orders are likely to continue execution and negatively impact the execution of subsequent rounds of orders, leading to adverse effects on the protocol's order execution.

Apart from the failure caused by transferring the relayer fee, other parts of the protocol, such as `_settlePnl()`,`_chargeRelayFee()` and certain `_verifyOrder()` functions, can also result in transaction failures.
```solidity
    function _settlePnl(uint256 marketId, address trader, int256 realizedPnl) internal {
        PositionModelState storage state = _getPositionModelStorage().stateMap[marketId];
        Position storage position = state.positionMap[trader];

        int256 oldUnsettledPnl = position.unsettledPnl;
        int256 settledPnl = state.settlePnl(trader, realizedPnl);
        int256 newUnsettledPnl = position.unsettledPnl;

        _updateMargin(marketId, trader, settledPnl);

```
```solidity


```

## Impact
The settlement functionality of limit orders will be affected.



## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L161-L225

## Tool used

Manual Review

## Recommendation
The suggested fix is to use try-catch blocks.


