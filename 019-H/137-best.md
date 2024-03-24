Generous Quartz Barbel

high

# Incorrect premium calculation in OracleMaker

## Summary
 Position rate should not be influenced by free collateral of the OracleMaker

## Vulnerability Detail
Position rate determines the exposure of OracleMaker. Position rate is used to determine the amount of premium based on the current oracleMaker position. 

For ex: If maker has 100 ETH long position and if a user wants to open a short then the price which he has to open will be less than actual market price. If user takes the long, then its in favour of oracleMaker, so no premium will be imposed 

Here is how the position rate is being calculated

```solidity
    function _getPositionRate(uint256 price) internal view returns (int256) {
        IVault vault = _getVault();
        uint256 marketId_ = _getOracleMakerStorage().marketId;
        // accountValue = margin + unrealisedPnl
        int256 accountValue = vault.getAccountValue(marketId_, address(this), price);
        int256 unrealizedPnl = vault.getUnrealizedPnl(marketId_, address(this), price);
        int256 unsettledMargin = accountValue - unrealizedPnl;
        int256 collateralForOpen = FixedPointMathLib.min(unsettledMargin, accountValue);
        // TODO: use positionMarginRequirement
        //int256 collateralForOpen = positionMarginRequirement + freeCollateralForOpen;
        if (collateralForOpen <= 0) {
            revert LibError.NegativeOrZeroMargin();
        }

        int256 maxPositionNotional = (collateralForOpen * 1 ether) / _getOracleMakerStorage().minMarginRatio.toInt256();

        // if maker has long position, positionRate > 0
        // if maker has short position, positionRate < 0
        int256 openNotional = vault.getOpenNotional(marketId_, address(this));
        int256 uncappedPositionRate = (-openNotional * 1 ether) / maxPositionNotional;

        // util ratio: 0 ~ 1
        // position rate: -1 ~ 1
        return
            uncappedPositionRate > 0
                ? FixedPointMathLib.min(uncappedPositionRate, 1 ether)
                : FixedPointMathLib.max(uncappedPositionRate, -1 ether);
    }

```
The problem here is the `uncappedPositionRate` also depends on `maxPositionNotional`. `maxPositionNotional` is the maximum openNotional possible for current `openCollateral` of oracleMaker with given `minMarginRatio`.

`int256 uncappedPositionRate = (-openNotional * 1 ether) / maxPositionNotional;`

So in this case if the `freeCollateral` is more in the oracle maker than the premium becomes proportionally low which is not correct.

Consider this scenario
 Currently oracleMaker has 100 ETH short position. If a user wants to take a 10 ETH long position there must premium against user. So the reservation price at which user opens will be higher than market price. 
 
Now the user deposits large amount of collateral as LP. As the. freeCollateral increases, for sufficient collateral  given the premium free decreases proportionally. After the user opens the position with less premium, he just withdraws the collateral back. Thus bypassing the premium enforced by oracle maker

This is possible by considering free collateral while calculating the position rate. Position rate should solely depend on openNotional and not on free collateral in any way

## Impact
Incorrect handling of risk exposure 

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L411

## Tool used

Manual Review

## Recommendation

Modify the position rate calculation by excluding `maxPositionNotional`
