Ancient Cherry Mantaray

high

# When the market is suspended(non Malicious),the system will accumulate bad debts

## Summary
When the market is suspended(non Malicious),the system will accumulate bad debts

## Vulnerability Detail
For simplicity we assume there are  one taker and one maker.


****A.** Under normal circumstances:**

1.current price 100

2.taker open long 50eth on maket

3.price goes down 80, taker is liquidatable

4.result in  system bad debt. 125，000000000000000000


-----------------------------------------------------------------------------

**B. Market suspended for one day result in accumulation of bad debt**

1. current price 100

2.taker open long 50eth on maket

3. price goes down 80, taker is liquidatable

4 .Market suspended

5.skip 1 day,resume market

6.Liquidation taker 

7.result in  system bad debts 285,916661693152975000  =(160,916661693152975000（ pendingfee） +125，000000000000000000(old bad debt) )

----------------------------------------------------------------------------------------------------------------------------
The root cause is  that the` fundingfee` is still being calculated during the market suspension period.

Relevant  code snippet：

First, positions cannot be settled,open or close when the market is suspended(only marketExistsAndActive)：

https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/vault/Vault.sol#L141C56-L141C77
```solidity
 function settlePosition(
        SettlePositionParams calldata params
    ) external override nonReentrant onlyClearingHouse marketExistsAndActive(params.marketId) {
```

https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/fundingFee/FundingFee.sol#L73
```solidity
  function beforeSettlePosition(
        uint256 marketId,
        address taker,
        address maker
    ) external onlyVault returns (int256, int256) {
        _updateFundingGrowthIndex(marketId);
        int256 takerFundingFee = _settleFundingFee(marketId, taker);
        int256 makerFundingFee = _settleFundingFee(marketId, maker);
        return (takerFundingFee, makerFundingFee);
    }
```

Secondly ,`FundingGrowthIndex` is still being calculated during the market pause

```solidity
    function _updateFundingGrowthIndex(uint256 marketId) internal {
        int256 fundingRate = getCurrentFundingRate(marketId);
        // index increase -> receive funding
        // index reduce   -> pay funding
        _getFundingFeeStorage().fundingGrowthLongIndexMap[marketId] +=
            fundingRate *
 >>  here         int256(block.timestamp - _getFundingFeeStorage().lastUpdatedTimestampMap[marketId]);
        _getFundingFeeStorage().lastUpdatedTimestampMap[marketId] = block.timestamp;
    }

```


POC:
**FundingFee.int.t.sol**
```solidity
    function test_poc() public {
        vm.prank(taker1);
        clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(maker),
                isBaseToQuote: false,
                isExactInput: false,
                amount: 50 ether,
                oppositeAmountBound: 5000 ether,
                deadline: block.timestamp,
                makerData: ""
            })
        );
        //Funding Rate -32183332338630595
	console2.log(fundingFee.getCurrentFundingRate(marketId));
        _mockPythPrice(80, 0);
		
	/*************************************/
	//assume market suspended  here
        systemStatus.suspendMarket(marketId);
        assertTrue(systemStatus.marketSuspendedMap(marketId));
        skip(1);
        systemStatus.resumeMarket(marketId);
	/************************************/
        vm.prank(liquidator);
        clearingHouse.liquidate(
            IClearingHouse.LiquidatePositionParams({ marketId: marketId, trader: taker1, positionSize: 50 ether })
        );
		//bad debt 285,916661693152975000
        console2.log("baddebt", vault.getBadDebt(marketId));
         //maker PendingFee   -160916661693152975000
        console2.log("maker PendingFee", fundingFee.getPendingFee(marketId, address(maker)));
    }

```

## Impact
1.Lead to more bad debt in the market

2. Additional losses to the users due to pendingFee incurred during the market  suspended 

## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/fundingFee/FundingFee.sol#L151C1-L159C6

## Tool used

Manual Review

## Recommendation
`Fundingfee ` should be suspended during the market suspension period.
