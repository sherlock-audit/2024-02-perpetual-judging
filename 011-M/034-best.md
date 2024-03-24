Loud Steel Pelican

high

# `SpotHedgeBasedMaker` does not check `minMarginRatio` or any similar requirements on withdraw

## Summary

`SpotHedgeBasedMaker` allows LPs to provide liquidity to a maker which swaps certain amounts of assets to spot. 
This hedges risk in case asset price in or decreases. 

In case the maker has open positions, it is important to check margin requirements on withdrawal. 
This is important to prevent a bank run (many LPs withdrawing), as well as risk control, while there are still open positions.

The problem currently is that it does not check for `minMarginRatio` on withdraw, unlike `OracleMaker`. 
This will lead to many problems, most notable that LPs can withdraw too much, ultimately putting the Maker at a too risky position, leading to bad debt.

## Vulnerability Detail

Like in `OracleMaker`, `SpotHedgeBasedMaker` includes following funcion:

```solidity
function _checkMinMarginRatio(uint256 price) internal view {
    int256 marginRatio = _getVault().getMarginRatio(_getSpotHedgeBaseMakerStorage().marketId, address(this), price);
    int256 minMarginRatio_ = _getSpotHedgeBaseMakerStorage().minMarginRatio.toInt256();
    if (marginRatio < minMarginRatio_) revert LibError.MinMarginRatioExceeded(marginRatio, minMarginRatio_);
}
```

This is used to check that the maker positions are still healthy.

The [docs](https://perp.notion.site/SpotHedgeMaker-5b9d196723d4471c8818fe65ad7b9ce0#fc2c90b9ee0e44079fe3641b6128fdda) state that upon withdrawing, a revert should happen if `vaultMarginRatio < minMarginRatio` after withdrawal.

However, inside of `withdraw()` this check is never called, nor are any checks regarding available assets left in the pool relative to its open position. This means LPs can withdraw the majority of the pool, leaving only a small collateral portion, and leaving the Maker severely undercollateralized.

## Impact

The impact can be quite severe, in case LPs decided to withdraw but there are open positions, the user will occur losses when trying to close positions again.
In edge cases it will be impossible to close, because there are very few or no tokens left in the contract. 

In case the user uses another maker to close the position, the losses of the Spot Maker will be transferred to the LPs of the second maker.

Ultimately this will lead to the Maker taking a much riskier position than allowed due to collaterals being withdrawn more than allowed, which may even lead to protocol insolvency.

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L303-L357

## Tool used

Manual Review

## Recommendation

Note that unlike the Oracle Maker, the Spot Hedge Maker doesn't collateralize every deposits. Directly checking `minMarginRatio` may DoS withdraws, as withdraw does not change the collateral setting.

Some possible mitigations are:
- Checking `minMarginRatio` only if any quote tokens are withdrawn. 
- Ensure that the total value of Maker assets (not just the current margins!) is as least as much as the value of current opened positions (or their ratio is better than a certain admin-supplied threshold config).

We believe the second fix would better retain the hedging functionalities.