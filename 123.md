Amusing Cream Buffalo

high

# Two Pyth prices can be used in the same transaction to attack the LP pools

## Summary

Pyth oracles use a pull model, where the consumer of the price needs to provide a signed price from an offline provider. There are no guarantees that the price at the current time is the freshest price, which means an attacker can enter an LP position at one base price, and exit in another, all in the same transaction.


## Vulnerability Detail

The OracleMaker and SpotHedgeBaseMaker both allow LPs to contribute funds in exchange for getting an LP position. Outside of the requirment that the current price is within the [maxAge](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/oracle/pythOracleAdapter/PythOracleAdapter.sol#L83), there are no other freshness checks. An attacker can create a contract which, given two signed base prices, calls [`updatePrice()`](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/oracle/pythOracleAdapter/PythOracleAdapter.sol#L51) and `deposit()`s at the lower price, then calls `updatePrice()` at the higher price, and calls `withdraw()` at the higher price, for a risk-free profit.

For both the OracleMaker and the SpotHedgeBaseMaker, there are no fees for doing `deposit()`/`withdraw()`, and a flash loan can be used to magnify the effect of any price difference between two oracle readings. While both makers support a having a whitelist for who is able to deposit/withdraw, the code [doesn't](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L156) require one, the whitelist isn't mentioned in the contest readme, and the comments in [both](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L270-L272) [makers](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L196-L198) anticipate having to deal with malicious LPs. It appears that the whitelist will be used to ensure [first-depositor inflation](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/README.md?plain=1#L65) [attacks](https://duckduckgo.com/?q=ethereum+%22inflation+attack%22&ia=web) are mitigated until the pools contain sufficient capital.

The fact that there is a `maxAge` available does not prevent the issue, because Pyth updates are multiple times a second, whereas a block can only have one timestamp.


## Impact

Value accrual that should have gone to the existing LPs is siphoned off by the attacker.


## Code Snippet

The number of shares given depends on whatever the most recently stored price is:
```solidity
// File: src/maker/OracleMaker.sol : OracleMaker.deposit()   #1

189 @>             uint256 price = _getPrice();
...    
201 @>             uint256 vaultValueXShareDecimals = _getVaultValueSafe(vault, price).formatDecimals(
202                    INTERNAL_DECIMALS,
203                    shareDecimals
204                );
205                uint256 amountXShareDecimals = amountXCD.formatDecimals(collateralToken.decimals(), shareDecimals);
206:@>             shares = (amountXShareDecimals * totalSupply()) / vaultValueXShareDecimals;
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L189-L206


The amount of collateral given back is based on whatever the most recently stored price is:
```solidity
// File: src/maker/OracleMaker.sol : OracleMaker.withdraw()   #2

234            uint256 redeemedRatio = shares.divWad(totalSupply());
...
241            uint256 price = _getPrice();
242 @>         uint256 vaultValue = _getVaultValueSafe(vault, price);
243            IERC20Metadata collateralToken = IERC20Metadata(_getAsset());
244 @>         uint256 withdrawnAmountXCD = vaultValue.mulWad(redeemedRatio).formatDecimals(
245                INTERNAL_DECIMALS,
246                collateralToken.decimals()
247:           );
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L234-L247


The SpotHedgeBaseMaker has the [same](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L274-L283) [issue](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L320-L325).


## Tool used

Manual Review


## Recommendation

Require that LP deposits and withdrawals be done by the trusted relayers
