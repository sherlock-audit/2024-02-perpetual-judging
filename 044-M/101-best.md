Fluffy Fuzzy Chicken

medium

# `settleOrder` doesn't calculate margin requirement correctly if `ActionType` is `reduceOnly`.

## Summary
`settleOrder` doesn't calculate margin requirement correctly if `ActionType` is `reduceOnly`. 
 
## Vulnerability Detail

For openPosition Action, checkMarginRequirement is executed at the end of execution after margin transfer and Pnl accumulation.
```solidity
function settleOrder(SettleOrderParam[] calldata settleOrderParams) external onlyRelayer(_sender()) nonReentrant {
        uint256 settleOrderParamsLength = settleOrderParams.length;

        __SNIP__

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

@>          _openPosition(
                InternalOpenPositionParams({
                    clearingHouse: clearingHouse,
                    settleOrderParam: settleOrderParam,
                    makerData: makerData,
                    takerRelayFee: takerRelayFee,
                    makerRelayFee: makerRelayFee
                })
            );

@>      // withdraw margin for taker reduce order
            if (takerWithdrawMarginParam.trader != address(0)) {
                _withdrawMargin(
                    context,
                    marketId,
                    takerWithdrawMarginParam.trader,
                    takerWithdrawMarginParam.requiredMarginRatio
                );
            }

            // withdraw margin for maker reduce order
            if (makerWithdrawMarginParam.trader != address(0)) {
                _withdrawMargin(
                    context,
                    marketId,
                    makerWithdrawMarginParam.trader,
                    makerWithdrawMarginParam.requiredMarginRatio
                );
            }
        }
    }
```
_openPosition will call ClearingHouse#_openPositionFor as you can see below.

```solidity
function _openPositionFor(address taker, OpenPositionParams memory params) internal returns (int256, int256) {
        uint256 marketId = params.marketId;
        uint256 price = _getPrice(marketId);

        IVault vault = _getVault();
        OpenPositionResult memory result = _openPosition(vault, taker, params, PositionChangedReason.Trade);

        _checkMarginRequirement(
            CheckMarginRequirementParams({
                vault: vault,
                marketId: marketId,
                trader: taker,
                price: price,
                isReducing: result.isTakerReducing
            })
        );
        _checkMarginRequirement(
            CheckMarginRequirementParams({
                vault: vault,
                marketId: marketId,
                trader: params.maker,
                price: price,
                isReducing: result.isMakerReducing
            })
        );

        return (result.base, result.quote);
    }
```
When order is filled by relayer, position will be opened and if ActionType is reduceOnly, margin according to increased amount than original margin ratio will be withdrawn.
```solidity
function _withdrawMargin(
        InternalContext memory context,
        uint256 marketId,
        address trader,
        uint256 requiredMarginRatio
    ) internal {
        uint256 withdrawableMargin = _getWithdrawableMargin(context, marketId, trader, requiredMarginRatio);
        if (withdrawableMargin > 0) {
            context.vault.transferMarginToFund(
                marketId,
                trader,
                withdrawableMargin.formatDecimals(INTERNAL_DECIMALS, context.collateralTokenDecimals)
            );
        }
    }
```
However, if ActionType is reduceOnly, checkMarginRequirement will not be executed at end of execution and after checkMarginRequirement, margin will be transferred to order owner, and addtional Pnl will be accured by withdrawal some margin.
```solidity
function _transferFundToMargin(
        uint256 marketId,
        address trader,
        uint256 amountXCD
    ) internal marketExistsAndActive(marketId) {
        if (amountXCD == 0) {
            revert LibError.ZeroAmount();
        }

        _updateFund(trader, -amountXCD.toInt256());

        // update accounting
        _formatAndUpdateMargin(marketId, trader, amountXCD.toInt256());

        // repay from margin right away if there's any unsettled loss before
        _settlePnl(marketId, trader, 0);
    }
```
Because check margin requirement is applied before transferring remaining margin, margin, and Pnl value changed after the check, so it affects final margin requirement.

## Impact
Margin Requirement will not be working correctly in case of reduceOnly ActionType.

## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L161-L225

## Tool used

Manual Review

## Recommendation
If ActionType is reduceOnly, checkMarginRequirement should be executed after withdrawal. 