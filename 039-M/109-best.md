Rare Cobalt Beaver

medium

# The Order Key Does Not Include Complete Order Data

## Summary
The Order Key Does Not Include Complete Order Data

## Vulnerability Detail
In `OrderGatewayV2`, the calculation of the order key only includes the trader address and the order ID. This leads to the possibility of multiple distinct orders having the same order key.

## Impact
The contract uses the order key to record the quantity of an order that has already been filled, which may result in a reduction of the amount that other order can clear, or even render them unable to clear. 
The contract also uses the order key to determine whether an order has been cancelled, leading to a risk of orders being mistakenly identified as cancelled. 
At the same time, the same order key can only carry out the function of `transferFundToMargin` once. This could allow subsequent orders with the same order key to be properly executed, but they would be unable to correctly increase the margin, posing risk to the positions.

## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/orderGatewayV2/LibOrder.sol#L11-L17

https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L251-L259

https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L312-L318

## Tool used

Manual Review

## Recommendation

It is recommended to use the complete order content to generate the order key.