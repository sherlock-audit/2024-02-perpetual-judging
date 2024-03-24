Massive Blush Wasp

medium

# The cooldown period on `CircuitBreaker` will never be triggered because transactions will revert

## Summary

When a hack occurs and a big amount of tokens start flowing out of the protocol, the Circuit Breaker should stop the withdrawals and start a cooldown period that allows the protocol team to address the issue. However, because of the current use of the Circuit Breaker, when the rate limit is triggered, the cooldown period will not start and withdrawals will start reverting until the next withdrawal period (i.e., 4 hours) starts, making the cooldown period useless.

## Vulnerability Detail

The main objective of the Circuit Breaker is to halt the outflow of funds when the rate limit is triggered. After the rate is reached, there's a cooldown period (i.e., 3 days) that allows the protocol team to address the issue, pausing the protocol in case there's been a hack. 

The issue is that in case there's a hack, the cooldown period will never be triggered because when the rate limit is triggered, transactions will revert instead of starting the cooldown period. Therefore, when the next withdrawal period starts, withdrawals will stop reverting and more funds will flow out of the protocol until the rate limit is triggered again. 

In the protocol documentation, the following is stated:

> 2. If within 4 hours, total withdrawal amount ≥ 50% of 1m, it triggers rate limit
> 3. In 3 days, no one can withdraw. this is the period we can investigate. After 3 days, anyone can call `overrideExpiredRateLimit()` to override/reset the rate limit, so users can claim their funds by calling `claimLockedFunds()`.

However, the current use of the Circuit Breaker will cause the cooldown period of 3 days to never start, allowing more funds to be stolen from the protocol in case of a hack. 

When a user withdraws from the Vault, a parameter is passed to indicate if the Circuit Breaker should revert the transaction when the rate limit is hit:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/Vault.sol#L110
```solidity
    // revert if hit withdrawal rate limit
    // we don't use the lockedFund-related features in Circuit Breaker for now
    _transferCollateralOut(trader, amountXCD, true);
```

In case of a hack, if the rate limit is hit, the Circuit Breaker won't halt the withdrawals during the whole cooldown period but it will only stop withdrawals during the rest of the current withdrawal period. 

## Impact

When a hack happens and the Circuit Breaker should stop withdrawals during a cooldown period of 3 days, it will only revert withdrawals during the rest of that period (usually 4 hours), allowing more funds to be stolen when the current period ends. 

## PoC

The following test can be pasted in `CircuitBreaker.spec.t.sol` and be run with the following command: `forge test --match-test test_hack_no_cooldown_period`.

```solidity
function test_hack_no_cooldown_period() public {
    vm.startPrank(trader);

    // We simulate current liquidity is 1000 tokens
    vault.deposit(trader, 1000e6);
    skip(4 hours);

    // We simulate a hack that withdraws 300 tokens
    vault.withdraw(300e6);

    // Now, all next withdrawals will be reverted because rate limit is hit (70%)
    vm.expectRevert(abi.encodeWithSelector(LibError.RateLimited.selector));
    vault.withdraw(1e6);

    // Cooldown period won't be activated so after 4 hours, funds will start flowing out again
    skip(4 hours);
    vault.withdraw(200e6);

    vm.stopPrank();
}
```

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/Vault.sol#L110

## Tool used

Manual Review

## Recommendation

To mitigate this issue is recommended to do either one of these two actions:

1. Use the lock functionality of Circuit Breaker
2. Enclose the call to Circuit Breaker on a `try/catch` statement so that if the rate limit is hit and withdrawals start reverting, pause the entire protocol to give the protocol team time to deal with the issue. 