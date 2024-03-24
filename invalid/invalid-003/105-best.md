Early Lipstick Cobra

medium

# setAuthorization functions should not be hard coded

## Summary

`setAuthorization` functions in ClearingHouse contracts should not be hard coded

## Vulnerability Detail

The `ClearingHouse` contract inherits and override the `AuthorizationUpgradeable`, but uses hard coding to allow users to authorize only ordergateway & ordergatewayv2. 
The code snippet is as follows:
```solidity
/// @dev in the beginning it only open for orderGateway & orderGatewayV2 to be authorized
/// @inheritdoc AuthorizationUpgradeable
function setAuthorization(address authorized, bool isAuthorized_) public override {
    IAddressManager addressManager = getAddressManager();
    if (
        isAuthorized_ &&
        authorized != address(addressManager.getOrderGateway()) &&
        authorized != address(addressManager.getOrderGatewayV2())
    ) {
        revert LibError.NotWhitelistedAuthorization();
    }
    super.setAuthorization(authorized, isAuthorized_);
}
```

## Impact

1. Due to the hard coding of `setAuthorization` function, the only way to enable users to authorize accounts other than ordergateway & ordergatewayv2 is to upgrade the `ClearingHouse` contract, which we believe is too expensive.

2. In addition, ordergateway & ordergatewayv2 contracts do not have the function of maker, and any taker trade with them in `ClearingHouse` will fail, because ordergateway & ordergatewayv2 contracts cannot hold any margin in vault contracts.

## Code Snippet

ClearingHouse.sol:
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L227-L237

## Tool used

Manual Review
Foundry Vscode

## Recommendation

1. Add a switching variable to the `ClearingHouse` contract, such as `isOpenForAll` to control whether users are allowed to authorize more accounts in the future.
2. Or add a dynamic white list to allow users to authorize accounts in the white list



