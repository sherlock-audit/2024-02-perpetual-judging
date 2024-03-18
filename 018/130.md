Macho Vinyl Shark

medium

# Unfair LP Fund Losses in SpotHedgeBaseMaker Due to CircuitBreaker Rate-Limiting

## Summary
SpotHedgeBaseMaker LPs Face Unfair Fund Losses Due to CircuitBreaker Rate-Limiting and Vault.withdraw() Usage in Long Positions that opened using HedgeMaker as a maker.
## Vulnerability Detail
CircuitBreaker can prevent from withdrawing for some amount of time if there has been massive withdraws from protocol in the last withdrawalPeriod. To make it easier to understand let's use perpetual documentation regarding CircuitBreaker:
>  1. For instance, assume:
    a. we have total 1m deposited usdc
    b. we set `constructor(3 days, 4 hours, 1 hour)` 
    c. we set `registerAsset(usdc, 5000, 0e6)`
 > 2. If within 4 hours, total withdrawal amount ≥ 50% of 1m, it triggers rate limit
  >3. In 3 days, no one can withdraw. this is the period we can investigate. After 3 days, anyone can call `overrideExpiredRateLimit()` to override/reset the rate limit, so users can claim their funds by calling `claimLockedFunds()`
 >   4. If we found it’s an expliot during `_rateLimitCooldownPeriod`, admin can call `markAsNotOperational()` to prevent any future deposit / withdraw / claimLockedFund

The problem is during this period, SpotHedgeBaseMaker became vulnerable because it tries to use withdraw() functionality from vault if position opened against HedgeMaker is long. Hence HedgeMaker can only be used against short positions in this period where it is not possible to withdraw from vault. Malicious traders can open large amounts of short positions against HedgeMaker in this period. Also it is not a necessity for traders to act maliciously, it might not be a malicious action, just a profitable action as I will explain in the next scenario:

In the case of market crash, both will be more likely since base token's value will be decreasing massively hence it will be profitable to open short positions against HedgeMaker, and also during market crashes it is more likely for CircuitBreaker to rateLimit the system because users will try to leave the market more than likely. 

In the end of this period, HedgeMaker's position in vault will be in lost and even can be below liquidation threshold. Since it is not possible to liquidate HedgeMaker currently, it will continue to lose value.

It is also important to mention that in order for a liquidity provider to withdraw from HedgeMaker, they should first open a position that withdraws funds from vault (hence decrease it's short exposure), hence this also won't be possible in this period.

In extreme cases it is even possible that HedgeMaker accountValue in vault can end up with nearly 0 hence all LP's can lose all funds they have deposited.

## Impact
Fund loss for liquidity providers in SpotHedgeBaseMaker with amount of loss increasing as we go from normal to edge cases.
## Code Snippet
[SpotHedgeBaseMaker.sol/withdraw()](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol/#L303)
[SpotHedgeBaseMaker.sol/fillOrder()](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol/#L365)
[SpotHedgeBaseMaker.sol/_fillOrderCallback](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol/#L617)
[Vault.sol/__transferCollateralOut](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/Vault.sol/#L343)
[CircuitBreaker.sol/_onTokenOutflow](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/circuitBreaker/CircuitBreaker.sol/#L273)
## Tool used

Manual Review

## Recommendation
Reconsider the withdraw interactions between SpotHedgeBaseMaker and the vault. In a system where only withdrawals can be prevented (due to the usage of CircuitBreaker), while deposits and position openings can continue, requiring a withdrawal in a position opening function is not advisable and can lead to exploits, as described.