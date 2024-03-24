Great Taupe Gibbon

medium

# Lack of Freshness Check in getPrice Function

## Summary
The `getPrice` function in the `PythOracleAdapter` contract lacks a freshness check, potentially leading to the usage of stale prices. This vulnerability could result in inaccurate or outdated data being utilized by smart contracts relying on price feeds.


## Vulnerability Detail
The `getPrice` function retrieves the price for a given price feed ID without verifying its freshness. Without a freshness check, there's a risk of utilizing stale prices, which may not accurately reflect the current market conditions. This could lead to incorrect decisions being made by smart contracts relying on these prices, potentially resulting in financial losses or incorrect system behavior.
```solidity
function getPrice(bytes32 priceFeedId) public view returns (uint256, uint256) {
    // Get the current block timestamp
    uint256 currentTimestamp = block.timestamp;

    // Check if the price feed exists
    if (!priceFeedExists(priceFeedId)) revert LibError.IllegalPriceFeed(priceFeedId);

    // Try to get the price from Pyth, ensuring it is not older than the maximum allowed age
    try _pyth.getPriceNoOlderThan(priceFeedId, _maxPriceAge) returns (PythStructs.Price memory price) {
        // Ensure the price is fresh
        require(price.publishTime + _maxPriceAge >= currentTimestamp, "Price is stale");

        // Assumes base price is against quote
        return (_convertToUint256(price), price.publishTime);
    } catch (bytes memory reason) {
        revert LibError.OracleDataRequired(priceFeedId, reason);
    }
}
```

## Impact
The lack of a freshness check in the `getPrice` function can lead to smart contracts using outdated or stale prices, potentially resulting in incorrect decisions, financial losses, or system malfunctions.
## Code Snippet
[#L80-L89](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/oracle/pythOracleAdapter/PythOracleAdapter.sol#L80-L89)
## Tool used

Manual Review

## Recommendation
Implement a freshness check in the getPrice function to ensure that only recent prices are utilized.