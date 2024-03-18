Massive Blush Wasp

high

# Underwater positions won't be liquidated when the PnL pool is empty

## Summary

When a position has created bad debt and must be liquidated and the PnL pool is empty, liquidators won't be incentivized to liquidate that position because they won't be able to withdraw the profits of the liquidation in the same transaction.

## Vulnerability Detail

When a position has to be liquidated, anyone can call the function `ClearingHouse.liquidate` to liquidate it and earn a fee that will be settled from the PnL pool. This fee will be transferred from the margin of the liquidated account to the liquidator's margin. 

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L193
```solidity
vault.transferMargin(params.marketId, params.trader, liquidator, result.feeToLiquidator);
```

In cases where the trader doesn't have enough margin to pay the liquidator fee, it will create bad debt on the protocol. When a liquidation generates bad debt and the PnL pool is empty, the liquidator won't receive the fee but it will be added as an unsettled margin on his account. 

The function `vault.transferMargin` will call `_settlePnl`:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/Vault.sol#L214-L222
```solidity
_settlePnl(marketId, from, -(amount.toInt256()));
_settlePnl(marketId, to, amount.toInt256());
```

And the function `_settlePnl` will call [`settlePnl`](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/LibPositionModel.sol#L19-L67), that will try to transfer the margin between the liquidated trader and the liquidator, using the PnL pool in between. In cases where the trader doesn't have enough margin and the PnL pool is empty, the liquidator won't have the margin increased, but that value will be stored in `unsettledMargin`.

When a liquidator has the fee added to `unsettledMargin` instead of margin, it won't be able to withdraw that fee because it will have to wait for the PnL pool to increase. The issue is that the liquidator doesn't know if the PnL pool is going to be replenished again, therefore it won't risk liquidating an underwater position in this scenario. 

Given that the liquidators are always bots that seek only profits, a liquidator won't be willing to spend funds to liquidate an account if there are not going to be instant profits on the same transaction.

## Impact

Positions that are underwater when the PnL pool is empty won't be liquidated because liquidators won't receive the fee in the same transaction. This will cause the position isn't liquidated when it should be and it may create more bad debt for the protocol if the market continues to move against that position. 

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L193-L193

## Tool used

Manual Review

## Recommendation

To mitigate this issue, is recommended to create a special mechanism that will always give the fee to liquidators, even if the PnL pool is empty at that moment. This mitigation is crucial because if underwater positions are not liquidated on time, is possible that they will create even more bad debt, which will hurt the protocol's health. 