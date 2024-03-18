Amusing Cream Buffalo

medium

# Attackers can sandwich their own trades up to the price bands

## Summary

Malicious users can sandwich their own SpotHedgeBaseMaker trades up to the price band cap


## Vulnerability Detail

The SpotHedgeBaseMaker allows one to settle trades against a UniswapV3 pool. The assumption is that the pool prices tokens properly, and any imbalance in the pool is reflected in the price paid/amount received by the trader interacting with the SpotHedgeBaseMaker. This assumption is incorrect, because an attacker can sandwich their own trade, taking value out of the Perpetual system.

The attacker would get a large flash loan, imbalance the pool, use the ClearingHouse to settle and opening trade, then re-balance the pool, all in the same transaction. 


## Impact

For example, assume the actual exchange rate is $4,000/1WEth, and the attacker is able to skew it such that the exchange rate temporarily becomes $1/1WEth. The attacker opening a short of 1Eth means that the SpotHedgeBaseMaker ends up going long 1Eth, and hedges that long by swapping 1WEth for $1. The attacker ends up using ~1 in margin to open the short. After the attacker unwinds the skew, they've gained ~1WEth (~$4k) from the rebalance, and they can abandon the perp account worth -$4k.

In reality, the attacker won't be able to skew the exchange rate by quite that much, because there's a price band [check](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L331) at the end of the trade, ensuring that the price gotten on the trade is roughly equivalent to the oracle's price. The test [comments](https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/test/orderGatewayV2/OrderGatewayV2.settleOrder.int.t.sol#L1271) indicate that the bands are anticipated to be +/- 10%. If the price bands are set to zero (the swap price must be the oracle price), then the SpotHedgeBaseMaker won't be usable at all, since uniswap charges a fee for every trade. If the bands are widened to be just wide enough to accommodate the fee, then other parts of the system, such as the OracleMaker won't work properly (see other submitted issue). Therefore, either _some_ value will be extractable, or parts of the protocol will be broken.

Because of these restrictions/limitations I've submitted this as a Medium.


## Code Snippet

The SpotHedgeBaseMaker doesn't require that the [sender](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L281) be the [Gateway/GatewayV2](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L231-L232), so anyone can execute it directly from the ClearingHouse:
```solidity
// File: src/maker/SpotHedgeBaseMaker.sol : SpotHedgeBaseMaker.isValidSender()   #1

514        function isValidSender(address) external pure override returns (bool) {
515            return true;
516:       }
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L504-L526


## Tool used

Manual Review


## Recommendation

Require that SpotHedgeBaseMaker be executed by a relayer