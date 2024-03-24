Blunt Porcelain Pig

medium

# pythOracleAdapter excess funds can be drained by any user

## Summary
The ```pythOracleAdapter#updatePrice``` function can be repeatedly called to update miscellaneous oracles in order to drain the contract of all its native tokens.
## Vulnerability Detail
The ```pythOracleAdapter``` contract is used to interact with the pyth oracle network to retrieve and update prices. The ```depositOracleFee``` function is used to deposit native token used by the ```updatePrice``` function to cover the pyth fees required in order to update the price feed.
```solidity
    function depositOracleFee() external payable {
        emit OracleFeeDeposited(_msgSender(), msg.value);
    }
...
    function updatePrice(bytes32 priceFeedId, bytes memory signedData) external {
        if (!priceFeedExists(priceFeedId)) revert LibError.IllegalPriceFeed(priceFeedId);

        bytes[] memory pythUpdateData = new bytes[](1);
        pythUpdateData[0] = signedData;

        // Get fee amount to pay to Pyth
        uint256 fee = _pyth.getUpdateFee(pythUpdateData);
        uint256 balance = address(this).balance;
        if (balance < fee) revert LibError.OracleFeeRequired(fee);

@>     _pyth.updatePriceFeeds{ value: fee }(pythUpdateData);
   }
```
Since this function is permissionless and allows any priceFeedId to be passed, a malicious user can take advantage of this by calling the function to update irrelevant price feeds. Since the fee is taken from the contracts balance, this will eventually drain it of all funds.
## Impact
The impact is two-fold:
1. The funds locked inside the ```PythOracleAdapter``` contract can be griefed and lost by any user and spent updating irrelevant oracles
2. The whole protocol could face frequent DoS.
Since pyth oracles are pull-based instead of push-based, they rely on users to update them before usage. The ```getPrice``` function does the following:
```solidity
    function getPrice(bytes32 priceFeedId) public view returns (uint256, uint256) {
@>      try _pyth.getPriceNoOlderThan(priceFeedId, _maxPriceAge) returns (PythStructs.Price memory price) {
            return (_convertToUint256(price), price.publishTime);
        } catch (bytes memory reason) {
            revert LibError.OracleDataRequired(priceFeedId, reason);
        }
    }
```

The call to ```getPriceNoOlderThan``` will revert if the price feed hasn't been updated in ```_maxPriceAge``` seconds. This value has a default of 60 seconds and the function will revert if the last time the feed was updated exceeds this. Since the on-chain price is infrequently updated, its likely that a call to ```updatePrice``` will be needed for all of the main protocol functionality such as adding liquidity, removing liquidity, creating orders, and liquidating. If ```updatePrice``` fails due to lack of pyth funds then ```getPrice``` will revert and the entire protocol will face downtime and possibly gain bad debt due to lack of liquidations.
## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/oracle/pythOracleAdapter/PythOracleAdapter.sol#L51
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/oracle/pythOracleAdapter/PythOracleAdapter.sol#L63
## Tool used

Manual Review

## Recommendation
Make the ```updatePrice``` function payable and force the ```_updatePriceFeeds{value: fee}``` call to use msg.value instead of the contracts funds.