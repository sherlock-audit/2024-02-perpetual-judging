Loud Steel Pelican

medium

# OracleMaker LPs are unnecessarily forced-exposed to risk when CircuitBreaker's rate limit is close to triggering

## Summary

When Circuit Breaker's rate limit is close to being triggered, Oracle Maker LPs may be unable to withdraw due to triggering the rate limit. The problem is that they are forced to keep such funds as margin, while it's possible to support a soft withdraw and allow.

## Vulnerability Detail

In the Oracle Maker, LPs are able to deposit collateral tokens. These tokens are then immediately used as margin against the market that it is providing liquidity for. When liquidity is withdrawn, the funds are taken out of margin and transferred to the LP.

Withdrawing margin is a two-step process, where the margin is first transferred to fund, then the fund can be withdrawn to the user. The second step where the funds are withdrawn to the user, which may be prevented by the Circuit Breaker's rate limit.

However, in OracleMaker's withdrawal, these two steps are bundled, reverting the withdrawal when any of the two step fails. If the Circuit Breaker rate is close to triggering, then LPs are force-exposed to risk by not being able to withdraw margin to funds.

Consider the following scenario:
- Let the withdrawal rate limit be 30%.
- Current withdrawal rate has been 29%.
- OracleMaker LPs are still counted as margin, which still exposes them to risk, while it's possible to do a soft withdraw by transferring the margin to funds first, without actually withdrawing the assets.

## Impact

OracleMaker LPs are unnecessarily forced-exposed to risk by being forced to keep their funds as margin

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L250-L251

## Tool used

Manual review

## Recommendation

Support a Oracle Maker LP soft-withdraw by allow calling `transferMarginToFund()` only, then actually claiming the funds at a later time.
