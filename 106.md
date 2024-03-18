Loud Steel Pelican

medium

# Bad debt liquidation leaves liquidated user with negative margin, which can cause bank run and loss of funds for the last users to withdraw

## Summary

In case there is bad debt liquidations, the liquidated trader will end up with negative margin. 
Currently there is no reason to repay this negative margin, which will lead to trader abandoning this account.
In this case the negative margin amount is taken from other users, but is not socialized.
In case of large liquidations this will lead to bank run, blocking the withdraw of the last user.

## Vulnerability Detail

Following unit test gives a good example of how bad debt liquidations work: 
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/test/clearingHouse/Liquidate.int.t.sol#L626-L632

In the end it can be seen that the margin of the account is negativ, this also makes sense because the trader has to pay liquidation fee, which is higher then the available margin. 

Now if every position is closed, this negativ margin is not considered, which will force the last user to withdraw pay with his funds.

This can be seen by taking a look at the calculation of withdraw amount in FundModel, when withdrawing from vault.

```solidity
if (fundDeltaXCD >= 0) {
            $.fundMap[trader] += fundDeltaAbsXCD;
        } else {
            // when fundDelta is negative (withdraw), check if trader has enough fund
            if ($.fundMap[trader] < fundDeltaAbsXCD) {
                revert LibError.InsufficientFund(trader, $.fundMap[trader], fundDeltaXCD);
            }

            $.fundMap[trader] -= fundDeltaAbsXCD;
        }

```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/FundModelUpgradeable.sol#L44-L54

Since all users know about this feature, after bad debt they will race to be the first to withdraw, triggering a bank run.

## Impact

After ANY bad debt, the protocol collateral for all non-negative users will be higher than protocol funds available, 
which can cause a bank run and a loss of funds for the users who are the last to withdraw.

Even if someone covers the shortfall for the user with negative margin, this doesn't guarantee absence of bank run:

If the shortfall is not covered quickly for any reason, the other users can notice disparency between collateral and funds in the protocol and start to withdraw
It is possible that bad debt is so high that any entity ("insurance fund") just won't have enough funds to cover it.

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/FundModelUpgradeable.sol#L44-L54
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/test/clearingHouse/Liquidate.int.t.sol#L626-L632

## Tool used

Manual Review

## Recommendation

Implementation can be non trivial, in general there should not exist an account with negative margin. 