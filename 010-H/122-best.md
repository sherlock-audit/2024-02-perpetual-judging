Generous Quartz Barbel

high

# Incorrect slippage check for taker in `ClearingHouse`

## Summary
Slippage checks in `ClearingHouse._openPosition` are incorrect. 

## Vulnerability Detail
`_checkExactInputSlippage` and `_checkExactOutputSlippage` implements slippage checks based on `isExactInput`. But it should be implemented based on taker position side which is determined by `isBaseToQuote`

The functions `_checkExactInputSlippage` and `_checkExactOutputSlippage` are currently working got `OrderGatewayV2` because `isBaseToQuote` and `isExactInput` being same in `OpenPositionForParams`.

The functions fails if users implement new routers to interact with clearingHouse by sending for Ex: `isBaseToQuote=false` `isExactInput=true` in `OpenPositionForParams`

Take this scenario

Lets suppose a trade is happening between userA(taker) and userB(maker). Taker open 10 long for ETH and Maker opens 10 short for ETH

filledAmount = 10

taker.price for 1 ETH = 12 USDC
maker.price for 1 ETH = 14 USDC

oppositeAmountBound =  taker.price * filledAmount
oppositeAmount = maker.price * fillAmount

oppositeAmountBound = 12 * 10 = 120
oppositeAmount = 14 * 10 = 140

It should revert is oppositeAmount is less than oppositeAmountBound because taker opening the long will get loss if maker gives a large price

But as the oppositeAmount is grater in this case, it doesn't revert thus bound check fails

## Impact

Incorrect slippage check leads to loss of user funds

## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L309

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L320

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L406

## Tool used

Manual Review

## Recommendation

Check slippage based on both `isBaseToQuote` and `isExactInput`
