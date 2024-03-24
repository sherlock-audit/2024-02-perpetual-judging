Amusing Cream Buffalo

high

# Signature validation callbacks can be used to make margin withdrawal calculations invalid

## Summary

The Perpetual protocol uses [ERC-6492](https://eips.ethereum.org/EIPS/eip-6492) signatures for order validation, and this signature scheme allows the user to execute arbitrary code, including updates to the Pyth price oracle.


## Vulnerability Detail

The code that calculates the ReduceOnly margin ratio, is done using a price that was fetched prior to the order validation callbacks being triggered, which means the prices and therefore margin calculations done previously will be invalid if either the maker or the taker update the Pyth oracle with a price that is different from the one used by the gateway. With crypto's fast-paced markets, price changes can be very large.

The actual settling of the order by the ClearingHouse verifies that the current margin requirements are [fulfilled](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L360-L384) based on the current Pyth price, but the code in the OrderGatewayV2 that tries to maintain the same margin ratio before and after the order, does not re-check the ratio with the new price.


## Impact

A user on the border of liquidation, who properly reduces their position to within the limits, could end up being liquidated in the next block, because too much margin was withdrawn. Alternatively, a user that knows they'll be liquidated soon, can withdraw more margin than they should be able to, leading to higher risk of bad debt for the exchange.


## Code Snippet

The price that the Relayer set is fetched and saved to `context.oraclePrice`, and then used in [`_fillTakerOrder()`](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L343)/[`_fillMakerOrder()`](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L378) to calculate the `requiredMarginRatio`, but by the time `_openPosition()` is called, the [price](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L360) it uses to fill the order may be [different](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L275):
```solidity
// File: src/orderGatewayV2/OrderGatewayV2.sol : OrderGatewayV2.settleOrder()   #1

182 @>             context.oraclePrice = _getPrice(context.pythOracleAdapter, context.config, marketId);
183    
184                (InternalWithdrawMarginParam memory takerWithdrawMarginParam, uint256 takerRelayFee) = _fillTakerOrder(
185 @>                 context,
186                    settleOrderParam
...
191                    InternalWithdrawMarginParam memory makerWithdrawMarginParam,
192                    uint256 makerRelayFee
193 @>             ) = _fillMakerOrder(context, settleOrderParam);
194    
195                _openPosition(
...
203                );
204    
205                // withdraw margin for taker reduce order
206                if (takerWithdrawMarginParam.trader != address(0)) {
207                    _withdrawMargin(
208                        context,
209                        marketId,
210                        takerWithdrawMarginParam.trader,
211:@>                     takerWithdrawMarginParam.requiredMarginRatio
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L191-L211


## Tool used

Manual Review


## Recommendation

Pass through the stored price through every call that requires it, rather than having each contract fetch it from Pyth. It can be a Relayer-specific code path
