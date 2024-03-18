Macho Vinyl Shark

medium

# SpotHedgeBaseMaker is Vulnerable to Price Manipulation Attacks Because Insufficient Slippage Protection

## Summary
SpotHedgeBaseMaker lacks protection against slippage, exposing makers to potential fund losses.
## Vulnerability Detail
When SpotHedgeBaseMaker is the maker for an openPosition call, it initiates a swap of the same amount via UniSwap to hedge its position. However, during this swap, slippage parameters --specifically "amountOutMinimum" if IsExactInput is 0, and "amountInMaximum" if IsExactOutput equals the balance(address(this))-- are not enforced:
```solidity
            if (isExactInput) {
                uint256 baseTokenRequired = _formatPerpToSpotBaseDecimals(amount);
                quoteTokenAcquired = _uniswapV3ExactInput(
                    UniswapV3ExactInputParams({
                        tokenIn: address(_getSpotHedgeBaseMakerStorage().baseToken),
                        tokenOut: address(_getSpotHedgeBaseMakerStorage().quoteToken),
                        path: path,
                        recipient: maker,
                        amountIn: baseTokenRequired,
                        amountOutMinimum: 0
                    })
                );
                oppositeAmount = _formatSpotToPerpQuoteDecimals(quoteTokenAcquired);
                // Currently we don't utilize fillOrderCallback for B2Q swaps,
                // but we still populate the arguments anyways.
                fillOrderCallbackData.amountXSpotDecimals = baseTokenRequired;
                fillOrderCallbackData.oppositeAmountXSpotDecimals = quoteTokenAcquired;
            } else {
                quoteTokenAcquired = _formatPerpToSpotQuoteDecimals(amount);
                uint256 oppositeAmountXSpotDecimals = _uniswapV3ExactOutput(
                    UniswapV3ExactOutputParams({
                        tokenIn: address(_getSpotHedgeBaseMakerStorage().baseToken),
                        tokenOut: address(_getSpotHedgeBaseMakerStorage().quoteToken),
                        path: path,
                        recipient: maker,
                        amountOut: quoteTokenAcquired,
                        amountInMaximum: _getSpotHedgeBaseMakerStorage().baseToken.balanceOf(maker)
                    })
```

 This means that slippage is not considered during the swap process. The sole slippage check occurs after the swap, through checkPriceBand() in ClearingHouse, once HedgeMaker returns the exact amounts from the UniSwap swap. However, this priceBand check does not fully safeguard against slippage, as it only verifies that the amount falls within a range of ±10% of the PythOracle price. Consequently, it remains possible to exploit HedgeMaker through price manipulation, resulting in potential gains of up to 10%.
## Impact
The vulnerability allows for value extraction from HedgeMaker, consequently affecting its liquidity providers. This results in gains for traders but leads to fund losses for liquidity providers. Additionally, accounting issues may arise within HedgeMaker due to the drop in its marginRatio following these swaps.
## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol/#L398-L425
## Tool used

Manual Review

## Recommendation
Reducing the priceBand won't resolve this issue and could potentially create additional problems, such as reverting benign transactions. Since pythPriceOracle is already utilized in ClearingHouse's openPosition() call (even when the maker is HedgeMaker which uses UniSwap), extending its use during the swap call and constraining the output from the swap (slippage) accordingly can effectively address the problem.