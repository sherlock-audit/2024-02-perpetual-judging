Suave Honeysuckle Wren

medium

# Liquidations of unhealthy positions may fail when the oracle is facing issues

## Summary

If the Pyth oracle is facing problems, liquidations can't be executed in the protocol, due to a revert that is happening when trying to fetch the price from the Pyth oracle via `PythOracleAdapter.getPrice()`.

## Vulnerability Detail

1. A liquidator calls `ClearingHouse.liquidate()` to liquidate a trader with an unhealthy position.

1. `_getPrice()` is being called subsequently (line 175 ClearingHouse.sol), which then calls `PythOracleAdapter.getPrice()` (line 532 ClearingHouse.sol)

Inside the function `PythOracleAdapter.getPrice()` a revert may be triggered if the Pyth oracle returns an error (line 87 PythOracleAdapter.sol).

This may cause liquidations to fail due to this revert.

## Impact

If liquidations may not be executed due to this issue, unhealthy positions with insufficient collateral may expose the protocol to risking a loss.

The risk depends on the volatility of the unhealthy position's market. In a volatile market, not liquidating an unhealthy position in time may result in a loss for the protocol.

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L175

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L530-L534

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/oracle/pythOracleAdapter/PythOracleAdapter.sol#L80-L87

## Tool used

Manual Review

## Recommendation

Consider adjusting `PythOracleAdapter.getPrice()` to not revert during liquidations. Instead returning the latest price that was fetched before.

```solidity
// PythOracleAdapter.sol
+80    function getPrice(bytes32 priceFeedId, bool isLiquidation) public view returns (uint256, uint256) {
...
86        } catch (bytes memory reason) {
+87           if (isLiquidation && latestPrice != 0) { return latestPrice; } // @audit: return stored value
+88           revert LibError.OracleDataRequired(priceFeedId, reason);
+89       }
```