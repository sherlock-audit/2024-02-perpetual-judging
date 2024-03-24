Macho Vinyl Shark

high

# Arbitrary Price Selection Enables Fund Theft Which Brokes The Vault Accounting

## Summary
By selecting an arbitrary price in an order involving non-whitelisted makers, it becomes possible to gain free funds, ultimately compromising the integrity of vault accounting.
## Vulnerability Detail
Takers are able to open positions with makers that are not whitelisted, allowing these makers to provide arbitrary prices for these positions. Consequently, takers and makers can agree upon a price that does not reflect the true value of an asset, thereby exploiting both other users of the protocol and the protocol itself. The vulnerability lies in the _openPosition() function, as illustrated by the following code snippet:
```solidity
            // Note that params.maker here could be arbitrary account, so we do allow arbitrary account
            // to set arbitrary price when fillOrder.
            // However, we also check margin requirements with oracle price for both taker and maker after openPosition
            // FIXME: this is not enough, see OrderGatewayV2SettleOrderIntTest.test_SettleOrder_AtExtremePrice()
            _checkIsSenderAuthorizedBy(params.maker);
            try this.decodeMakerOrder(params.makerData) returns (MakerOrder memory makerOrder) { 
                oppositeAmount = makerOrder.amount; // @audit - check explanation above
            } catch (bytes memory) {
                revert LibError.InvalidMakerData();
```
In the provided test case within the code comments, makers and takers agree upon a price of 1 wei for one unit of ETH. Consequently, both parties acquire ETH at the value of 1 wei, with one party taking a short position and the other a long position. Subsequently, they sell their positions, resulting in significant profits, as they essentially obtained tokens from the protocol for free and could exchange them with users.
```solidity
    // FIXME: We shouldn't allow this to happen
    // probably add price band in OrderGatewayV2 or ClearingHouse,
    // only allow order.price to be oracle price +- 10% when settling orders
    // See https://app.asana.com/0/1202133750666965/1206662770651731/f
    function test_SettleOrder_AtExtremePrice() public {
```
It appears that developers attempted to address this issue by introducing priceBand() as mentioned in test comments; however, this does not entirely mitigate the problem. 
```solidity
        _checkPriceBand(params.marketId, result.quote.abs().divWad(result.base.abs()));
```
The allowed price difference via priceBand is ±10%, leaving the vulnerability open to exploitation by anyone. Consequently, traders can purchase tokens with a 10% discount and then exchange them with other makers at their true value, thereby profiting from the disparity
## Impact
Users can effortlessly exploit this loophole to extract value from the protocol, potentially resulting in catastrophic consequences. With continued exploitation, the integrity of the vault's accounting could be completely compromised.
## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol/#L267
## Tool used

Manual Review

## Recommendation
To mitigate this vulnerability, it is imperative to prevent makers from selecting arbitrary prices for positions. Instead, always rely on oracle prices, even when two non-whitelisted parties are initiating positions against each other. This measure is essential as it not only impacts the involved parties but also influences the accuracy of protocol accounting. It is important to mention that further restricting priceBand is not a sufficient solution to this problem since it can create further problems such as reverting benign calls.