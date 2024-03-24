Amusing Cream Buffalo

high

# Borrow fees can be arbitrarily increased without the maker providing any value

## Summary

The SpotHedgeBaseMaker LPs can maximize their LP returns by closing their trades against other whitelisted makers


## Vulnerability Detail

The [whitelisted makers](https://perp.notion.site/Borrowing-Fee-Spec-v0-8-0-22ade259889b4c30b59762d582b4336d), which the SpotHedgeBaseMaker and the OracleMaker are, `[c]an receive borrowing fee based on [their] utilization ratio` and `[d]on’t need to pay borrowing fee` themselves. The borrowing fee is meant to be compensation for providing liquidity to the market, but makers like the SpotHedgeBaseMaker, which are able to hedge their position risk, can arbitrarily increase their utilization ratio by opening positions against the OracleMaker, and immediately closing them against the SpotHedgeBaseMaker, maximizing their fees without having to provide liquidity over time.

An attacker can choose a specific market direction, then monitor the utilization of the OracleMaker. Any time the OracleMaker's utilization is flat, the attacker would open a position in the chosen market direction against the OracleMaker (to minimize the dynamic premium), then immediately close the position by offsetting it with a taker order against the SpotHedgeBaseMaker. The only risk the attacker has to take is holding the position for the approximately ~2 second optimism block time, until they're able to offset the position using the ClearingHouse to interact directly with the SpotHedgeBaseMaker.

## Impact

Value extraction in the form of excessive fees, at the expense of traders on the other side of the chosen position direction.


## Code Snippet

Utilization does not take into account whether the taker is reducing their position, only that the maker is increasing theirs:
```solidity
// File: src/borrowingFee/LibBorrowingFee.sol : LibBorrowingFee.updateReceiverUtilRatio()   #1

40            /// spec: global_ratio = sum(local_ratio * local_open_notional) / total_receiver_open_notional
41            /// define factor = local_ratio * local_open_notional; global_ratio = sum(factor) / total_receiver_open_notional
42            /// we only know 1 local diff at a time, thus splitting factor to known_factor and other_factors
43            /// a. old_global_ratio = (old_factor + sum(other_factors)) / old_total_open_notional
44            /// b. new_global_ratio = (new_factor + sum(other_factors)) / new_total_open_notional
45            /// every numbers are known except new_global_ratio. sum(other_factors) remains the same between old and new
46            /// expansion formula a: sum(other_factors) = old_global_ratio * old_total_open_notional - old_factor
47            /// replace sum(other_factors) in formula b:
48            /// new_global_ratio = (new_factor + old_global_ratio * old_total_open_notional - old_factor) / new_total_open_notional
49            uint256 oldUtilRatioFactor = self.utilRatioFactorMap[receiver];
50 @>         uint256 newTotalReceiverOpenNotional = self.totalReceiverOpenNotional;
51            uint256 oldUtilRatio = self.utilRatio;
52            uint256 newUtilRatio = 0;
53            if (newTotalReceiverOpenNotional > 0) {
54                // round up the result to prevent from subtraction underflow in next calculation
55 @>             newUtilRatio = FixedPointMathLib.divUp(
56                    oldUtilRatio * self.lastTotalReceiverOpenNotional + newUtilRatioFactor - oldUtilRatioFactor,
57                    newTotalReceiverOpenNotional
58                );
59:           }   
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/borrowingFee/LibBorrowingFee.sol#L40-L59

and [whitelisted makers](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/borrowingFee/BorrowingFee.sol#L299-L303) never have to pay any fee:
```solidity
// File: src/borrowingFee/BorrowingFee.sol : BorrowingFee.getPendingFee()   #2

165        function getPendingFee(uint256 marketId, address trader) external view override returns (int256) {
166 @>         if (_isReceiver(marketId, trader)) {
167                return _getPendingReceiverFee(marketId, trader);
168            }
169            return _getPendingPayerFee(marketId, trader);
170:       }
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/borrowingFee/BorrowingFee.sol#L165-L170


## Tool used

Manual Review


## Recommendation

There is no incentive to reduce utilization, and I don't see a good solution that doesn't involve the makers having to actively re-balance their positions, e.g. force makers to also have to pay the fee, and only pay the fee to the side that has the largest net non-maker open opposite position

