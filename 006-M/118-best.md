Amusing Cream Buffalo

medium

# SpotHedgeBaseMaker LPs will be able to extract value during a USDT/USDC de-peg

## Summary

The Perpetual protocol only supports USDT as collateral, but prices everything as though USDT were always equal to 1-for-1. Ongoing [small de-pegs](https://coinmarketcap.com/currencies/tether/) will only leave small amounts for arbitrage, because any arbitrage in size will skew the OracleMaker and UniswapV3 pools' prices to account for the opportunity. One case that cannot be handled is a large de-peg when the SpotHedgeBaseMaker has a lot of base tokens available.


## Vulnerability Detail

The protocol's contest [readme](https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/README.md?plain=1#L15) says that either USDT or USDC will be used as collateral, and the project [documentation](https://perp.notion.site/PythOraclePool-e99a88be051f4bc8be0b1310eb982cd4) says that it will use Pyth oracles for pricing base tokens. Pyth only has a single oracle for USDT and USDC [each](https://pyth.network/developers/price-feed-ids), which is their USD value. All other Pyth oracles are for the USD price, not the USDT or USDC price.

In the case of the SpotHedgeBaseMaker, when a user wishes to withdraw their base tokens, it calculates the total value of all of the LP shares, as the [value](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L734-L743) of the vault account plus the net base tokens contributed during all deposit()s/withdraw()s. The price that it uses to convert the value of the vault in the vault's collateral, into the value of the vault in the base token's units, is the &lt;base>/USD price, not a &lt;base>/USDT or &lt;base>/USDC price (because there isn't an oracle for that).


## Impact

If USDT is the collateral token and de-pegs by 30%, instead of valuing the account's collateral at 70% of what was deposited, it keeps it at 100%, letting each LP share withdraw more funds than it should be able to. Only base tokens can be withdrawn, so the first LPs to withdraw during the de-peg will get more than they should, leaving less for all others.

A similar issue exists with all markets whose SpotHedgeBaseMaker's LP deposit/withdraw tokens are wrapped/bridged tokens, vs the actual token itself (e.g. WETH vs ETH). The price the SpotHedgeBaseMaker uses is the price returned by the oracle for the market, which is the actual token's oracle, not the hedging token's oracle. If there is a de-peg there (e.g. because the wrapped token is considered by some jurisdictions as a 'security', whereas the underlying isn't), then the amount of tokens deposited will be over-valued, and at the end when the market is wound down, profits not covered by the deposited hedging tokens will be withdrawable as the collateral token, which will have been over-valued at the expense of the other LPs.


## Code Snippet

The function to calculate the vault value in units of the base token, takes in a price...:
```solidity
// File: src/maker/SpotHedgeBaseMaker.sol : SpotHedgeBaseMaker.withdraw()   #1

311            uint256 redeemedRatio = shares.divWad(totalSupply()); // in ratio decimals 18
...
320            uint256 price = _getPrice();
321 @>         uint256 vaultValueInBase = _getVaultValueInBaseSafe(vault, price);
322            uint256 withdrawnBaseAmount = vaultValueInBase.mulWad(redeemedRatio).formatDecimals(
323                INTERNAL_DECIMALS,
324                _getSpotHedgeBaseMakerStorage().baseTokenDecimals
325:           );
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L311-L325

...which refers to the market's oracle's price:
```solidity
// File: src/maker/SpotHedgeBaseMaker.sol : SpotHedgeBaseMaker.price   #2

769        function _getPrice() internal view returns (uint256) {
770            IAddressManager addressManager = getAddressManager();
771 @>         (uint256 price, ) = addressManager.getPythOracleAdapter().getPrice(
772                addressManager.getConfig().getPriceFeedId(_getSpotHedgeBaseMakerStorage().marketId)
773            );
774            return price;
775:       }
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L769-L775

which is the price returned from the [pyth](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/oracle/pythOracleAdapter/PythOracleAdapter.sol#L80-L83) contract (converted for decimals), which is a USD price.


## Tool used

Manual Review


## Recommendation

Use the &lt;quote-token>/USD oracle to convert the &lt;base-token>/USD price to a &lt;base-token>/&lt;quote-token> oracle
