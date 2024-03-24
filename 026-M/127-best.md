Amusing Cream Buffalo

high

# Whale LPs can make the admin's risk control parameters ineffective

## Summary

Unlike other exchanges where the funding fee is used to incentivize the perpetual price to match the spot price, the Perpetual exchange always offers the spot price, and uses the funding fee to balance the much smaller OracleMaker's position exposure. Because the OracleMaker's position is much smaller than the exchange as a whole's position balance, it's much easier to manipulate than a normal exchange, and therefore will require more frequent changes to funding-related parameters controlled by the admin. The funding fee rate depends not only on admin-controlled parameters, but with the amount of LP funds available, which is under the control of the LPs.


## Vulnerability Detail

As the setup for the attack, a well-funded attacker (e.g. a competing exchange) can use sock puppets to add LP liquidity to the OracleMaker. When the funding rate needs to be modified (e.g. because the attacker decides to spend funds to drastically skew the OracleMaker's position balance), after the admin makes the change, the attacker can magnify the effect of the parameter by withdrawing the LP liquidity, or they can mute it by adding a lot of liquidity. Because the circuitbreaker nets in-flows against out-flows, the attacker can prevent hitting the limit by depositing margin to an arbitrary position, prior to calling `withdraw()` on the OracleMaker.


## Impact

Traders expecting to be paid a funding fee for taking their positions, can suddenly find themselves having to _pay_, at the maximum rate. Customers will not use an exchange where the funding fee suddenly flips, and can't be reliably hedged/arbitraged against another exchange's funding fee.


## Code Snippet

The factor and exponent set by the admin are modulated by the capacity, which is based on the maximum leverage available, given the total amount deposited by the LPs:
```solidity
// File: src/fundingFee/FundingFee.sol : FundingFee.getCurrentFundingRate()   #1

123            uint256 totalDepositedAmount = uint256(_getVault().getSettledMargin(marketId, fundingConfig.basePool));
124            uint256 maxCapacity = FixedPointMathLib.divWad(
125 @>             totalDepositedAmount,
126                uint256(OracleMaker(fundingConfig.basePool).minMarginRatio())
127            );
128    
129            // maxCapacity = basePool.totalDepositedAmount / basePool.minMarginRatio
130            // imbalanceRatio = basePool.openNotional^fundingExponentFactor / maxCapacity
131            // fundingRate = fundingFactor * imbalanceRatio
132            // funding = trader.openNotional * fundingRate * deltaTimeInSeconds
133            uint256 fundingRateAbs = FixedPointMathLib.fullMulDiv(
134 @>             fundingConfig.fundingFactor,
135                FixedPointMathLib
136 @>                 .powWad(openNotionalAbs.toInt256(), fundingConfig.fundingExponentFactor.toInt256())
137                    .toUint256(),
138 @>             maxCapacity
139:           );
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L123-L139


## Tool used

Manual Review


## Recommendation

Rate-limit or time-lock changes to the OracleMaker's LP levels
