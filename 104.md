Ancient Cream Finch

high

# Wrong Implementation of Borrowing Fee Mechanism causes lose of Funds for Whitelisted Makers

## Summary
According to the Perp V3 Documentation , Whitelisted Makers do not have to pay Borrowing Fee but instead they can receive borrowing fee based on its utilization ratio. In implementation , the borrowing fee is collected from all makers regardless of whether they are whitelisted or not . 
## Vulnerability Detail
Let's check the implementation of Borrowing Fee Mechanism for the flow : `OrderGatewayV2` -> `ClearingHouse` -> `Vault` . 
Here through `settleOrder` function of `OrderGatewayV2` , ClearingHouse's openPositionFor function is called where it indeed checks for whitelistedMakers as shown below: 
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L279C8-L279C84
```solidity
 bool hasMakerCallback = _isWhitelistedMaker(params.marketId, params.maker);
```
then in later part of the same function it calls `vault.settlePosition` as shown below:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L341C9-L349C15
```solidity
 vault.settlePosition(
            IVault.SettlePositionParams({
                marketId: params.marketId,
                taker: taker,
                maker: params.maker,
                takerPositionSize: result.base,
                takerOpenNotional: result.quote,
                reason: reason
            })
```
In `vault.settlePosition` we can see that borrowing is calculated and is subtracted from the margin balance of maker regardless of whether the Maker is a whitelisted maker or not as shown below : 
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/Vault.sol#L139C4-L176C10
```solidity
 function settlePosition(
        SettlePositionParams calldata params
    ) external override nonReentrant onlyClearingHouse marketExistsAndActive(params.marketId) {
        if (params.takerPositionSize == 0) revert LibError.ZeroAmount();

        // before hook
        int256 takerPendingMargin;
        int256 makerPendingMargin;
        IBorrowingFee borrowingFee = _getBorrowingFee();                                                                     <@audit
        bool hasBorrowingFee = address(borrowingFee) != address(0);
        if (hasBorrowingFee) {
            (int256 takerBorrowingFee, int256 makerBorrowingFee) = borrowingFee.beforeSettlePosition( 
                params.marketId,
                params.taker,
                params.maker,
                params.takerPositionSize,
                params.takerOpenNotional
            );
            takerPendingMargin -= takerBorrowingFee;
            makerPendingMargin -= makerBorrowingFee;                                                                             <@audit
        }
        IFundingFee fundingFee = _getFundingFee();
        if (address(fundingFee) != address(0)) {
            (int256 takerFundingFee, int256 makerFundingFee) = fundingFee.beforeSettlePosition(
                params.marketId,
                params.taker,
                params.maker
            );
            takerPendingMargin -= takerFundingFee;
            makerPendingMargin -= makerFundingFee;
        }
        // settle taker & maker's pending margin
        if (takerPendingMargin != 0) {
            _settlePnl(params.marketId, params.taker, takerPendingMargin);
        }
        if (makerPendingMargin != 0) {
            _settlePnl(params.marketId, params.maker, makerPendingMargin);                                 <@audit
        }
```
But here in the documentary of PerpV3 , under WhitelistedMaker section two points are clearly written : 
1) Can receive borrowing fee based on its utilization ratio.
2) Don’t need to pay borrowing fee.
https://perp.notion.site/Borrowing-Fee-Spec-v0-8-0-22ade259889b4c30b59762d582b4336d
## Impact
Loss of funds for Whitelisted Makers as Protocol fails to deliver the promises mentioned in the Documentation . 
## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L279C8-L279C84
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L341C9-L349C15
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/Vault.sol#L139C4-L176C10
## Tool used

Manual Review , Perp V3 Documentation given in Contest Page

## Recommendation
Remove the Borrowing Fee for Whitelisted Makers by adding a check inside `Vault.sol#settlePosition` .