Loud Steel Pelican

high

# OracleMaker's price with spread does not take into account the new position

## Summary

OracleMaker's `_getBasePriceWithSpread()` does not take into account the opening position's size, but only on the current position of the Maker.

This means there is no price impact protection against large trades (or any trades at all) for the Oracle Maker. Anyone can then bypass the spread by opening a reverse position before actually opening their intended position.

## Vulnerability Detail

To reduce risky positions, the Oracle Maker will quote a slightly worse price (for the trader) than the actual Oracle price for any positions that increases risk. This is also mentioned in the [audit docs](https://perp.notion.site/PythOraclePool-e99a88be051f4bc8be0b1310eb982cd4), Dynamic Premium section.

When a new position is requested, the Oracle Maker quotes a price that includes this spread:

```solidity
function fillOrder(
    bool isBaseToQuote,
    bool isExactInput,
    uint256 amount,
    bytes calldata
) external onlyClearingHouse returns (uint256, bytes memory) {
    uint256 basePrice = _getPrice();
    uint256 basePriceWithSpread = _getBasePriceWithSpread(basePrice, isBaseToQuote); // @audit here
```

There is no spread when the new position reduces risk (e.g. if the current Maker position is +3 ETH, and the new order implies -1 ETH, then the Maker will quote the oracle price directly).

However, `_getBasePriceWithSpread()` never uses the order's amount, therefore large positions will be quoted the same price as small positions, i.e. there is no price impact. This issue also exists if the new position passes the zero mark, where the Maker thinks it's de-risking, while in reality it's being subject to much more risk in the opposite direction.
- Suppose the Oracle Maker's current total position is 1 ETH long (+1 ETH)
- Someone opens a 100 ETH long position, the Oracle Maker thinks it's de-risking by being able to open a 100 ETH short, and quotes the oracle price.
- After the trade, the Maker is actually in a much riskier position of 99 ETH short (-99 ETH)

Anyone can then bypass the spread completely (or partially) by opening a position in the opposite direction before the intended direction:
- The Oracle Maker's current total position is 1 ETH long (+1 ETH).
- Alice wants to open a 10 ETH long (+10 ETH) position.

If Alice sends a 10 ETH long order now, she would have to accept the base price spread and get a slightly worse price than the Pyth oracle price. However Alice can bypass the spread by sending the following two orders in quick succession to the relayer:
- First order: 1 ETH short (-1 ETH)
  - The Maker thinks it's de-risking, so it quotes the oracle price directly.
  - After this order, the Maker has 0 ETH position.
- Second order: 11 ETH long (+11 ETH)
  - Since the Maker's position is zero, it quotes the oracle price.
  - Alice has opened a net total of +10 ETH as intended. However she is not subject to the price spread.
 
Alice was able to bypass the premium from the spread model, and force the Maker to quote exactly the Pyth oracle price. Note that Alice doesn't have to fully bypass the spread, she could have just opened a -0.5 ETH first and would still largely bypass the spread already. 

## Impact

- Dynamic Premiums from the price spread can be bypassed completely, traders can always be quoted the oracle price without spread (or with a heavy reduction of the spread).
- Maker is exposed to more risk than the intended design.

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L272

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L401-L409

## Tool used

Manual review

## Recommendation

`_getBasePriceWithSpread()` must take into account the average spread of the Maker's position before and after the trade, not just the position before the trade.
- See proof [here](https://sips.synthetix.io/sips/sip-279/#technical-specification). Note this formula still works when the new position movement crosses the zero mark, as the integral of a constant zero function is zero.