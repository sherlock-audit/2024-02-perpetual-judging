Ancient Cherry Mantaray

high

# maker may get the wrong  amount pendingFee  after  liquidation

## Summary
maker may get the wrong  amount pendingFee  after  liquidation

## Vulnerability Detail
For simplicity :

assume just one taker and one maker:

T1	
1.  current price 100
2. taker open long 50eth on maket
3. price goes down 82.5, taker is liquidatable(full liquidation)
4. after liquidation

- maker OpenNotional   5000000000000000000000    (**Here's the root cause of this issue**)
- maker PositionSize   -50000000000000000000
- taker OpenNotional  0   **(after full liquidation,taker's `OpenNotional` is 0)**
- taker PositionSize  0
- taker PendingFee 0
- maker PendingFee 0

5.skip 1 day, maker`s funding fee  incurred

- maker UnrealizedPnl  875000000000000000000
- maker PendingFee  -160916661693152975000   **（at this time，taker`s pendingFee=0,）**
- taker UnrealizedPnl   0
- taker PendingFee  0

POC：

**FundingFee.int.t.sol**

```solidity
   function test_acc() public {
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

        _mockPythPrice(825, -1);
        console2.log("taker PendingFee  before liqudate", fundingFee.getPendingFee(marketId, taker1));
        console2.log("maker PendingFee  before liqudate", fundingFee.getPendingFee(marketId, address(maker)));

        vm.prank(liquidator);
        //Liquidate taker Full
        clearingHouse.liquidate(
            IClearingHouse.LiquidatePositionParams({ marketId: marketId, trader: taker1, positionSize: 50 ether })
        );
        console2.log("-------Liquidate taker Full-------");
        console2.log("taker PendingFee  after liqudate", fundingFee.getPendingFee(marketId, taker1));
        console2.log("maker PendingFee after liqudate", fundingFee.getPendingFee(marketId, address(maker)));
        console2.log("--------------");
        console2.log("maker OpenNotional  after liqudate", vault.getOpenNotional(marketId, address(maker)));
        console2.log("maker PositionSize  after liqudate", vault.getPositionSize(marketId, address(maker)));
        console2.log("--------------");
        console2.log("taker OpenNotional  after liqudate", vault.getOpenNotional(marketId, taker1));
        console2.log("taker PositionSize after liqudate", vault.getPositionSize(marketId, taker1));
        console2.log("--------------");
      
        console2.log("------------after liqudate ,skip 1 days--------------");
        skip(1);
        console2.log("taker UnrealizedPnl after liqudate ", vault.getUnrealizedPnl(marketId, taker1, 82.5 ether));
        console2.log(
            "maker UnrealizedPnl after liqudate",
            vault.getUnrealizedPnl(marketId, address(maker), 82.5 ether)
        );
        console2.log("taker PendingFee  after liqudate 1 days", fundingFee.getPendingFee(marketId, taker1));
        console2.log("maker PendingFee after liqudate 1 days", fundingFee.getPendingFee(marketId, address(maker)));
}

Logs:
  taker PendingFee  before liqudate 0
  maker PendingFee  before liqudate 0
  -------Liquidate taker Full-------
  taker PendingFee  after liqudate 0
  maker PendingFee after liqudate 0
  --------------
  maker OpenNotional  after liqudate 5000000000000000000000
  maker PositionSize  after liqudate -50000000000000000000
  --------------
  taker OpenNotional  after liqudate 0
  taker PositionSize after liqudate 0
  --------------
  ------------after liqudate ,skip 1 days--------------
  taker UnrealizedPnl after liqudate  0
  maker UnrealizedPnl after liqudate 875000000000000000000
  taker PendingFee  after liqudate 1 days 0
  maker PendingFee after liqudate 1 days -160916661693152975000

```

Code Snippet:
The root cause is that the `maker` in` SettlePositionParams` is the `liquidator`.

https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L187

```solidity
vault.settlePosition(
            IVault.SettlePositionParams({
                marketId: params.marketId,
                taker: params.trader,
>>           maker: liquidator, 
                takerPositionSize: result.liquidatedPositionSizeDelta,
                takerOpenNotional: result.liquidatedPositionNotionalDelta,
                reason: PositionChangedReason.Liquidate
            })
        );
```

so in fact here is not update maket's `OpenNotional` and `PositionSize`

https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/vault/Vault.sol#L179
```solidity
   _addPosition(
            // Note: for maker, the reason for PositionChanged is always `Trade` because he is the counter-party
            AddPositionParams({
                marketId: params.marketId,
                trader: params.maker,
                maker: params.maker,
                positionSizeDelta: -params.takerPositionSize,
                openNotionalDelta: -params.takerOpenNotional,
                reason: PositionChangedReason.Trade
            })
        );
        _addPosition(
            AddPositionParams({
                marketId: params.marketId,
                trader: params.taker,
                maker: params.maker,
                positionSizeDelta: params.takerPositionSize,
                openNotionalDelta: params.takerOpenNotional,
                reason: params.reason
            })
        );
```

https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/fundingFee/FundingFee.sol#L164
Then `openNotional` used to calculate `Funding Fee`.
```solidity
function _settleFundingFee(uint256 marketId, address trader) internal returns (int256) {
        int256 fundingGrowthLongIndex = _getFundingFeeStorage().fundingGrowthLongIndexMap[marketId];
>>   int256 openNotional = _getVault().getOpenNotional(marketId, trader);
```

## Impact


1.If maker is payer will suffer losses（because of pendingFee）
POC:

Two Takers With Two Makers:

maker2 is payer

**FundingFee.int.t.sol**

```solidity
function test_poc() public {
        // taker1 short 10 eth on maker1
        vm.prank(taker1);
        clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(maker),
                isBaseToQuote: true,
                isExactInput: true,
                amount: 10 ether,
                oppositeAmountBound: 1000 ether,
                deadline: block.timestamp,
                makerData: ""
            })
        );

        // taker2 long 5 eth on maker2
        vm.prank(taker2);
        clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(maker2),
                isBaseToQuote: false,
                isExactInput: false,
                amount: 50 ether,
                oppositeAmountBound: 5000 ether,
                deadline: block.timestamp,
                makerData: ""
            })
        );

        skip(5);
        _mockPythPrice(80, 0);

        vm.prank(liquidator);
        //Liquidate taker2 Full
        clearingHouse.liquidate(
            IClearingHouse.LiquidatePositionParams({ marketId: marketId, trader: taker2, positionSize: 50 ether })
        );
       //99, 291029340535175000
        console2.log("maker2 PendingFee after liqudate ", fundingFee.getPendingFee(marketId, address(maker2)));
        console2.log("maker2 OpenNotional  after liqudate", vault.getOpenNotional(marketId, address(maker2)));
        console2.log("maker2 PositionSize  after liqudate", vault.getPositionSize(marketId, address(maker2)));

        skip(2);
       //139, 007441076749245000
        console2.log("maker2 PendingFee after liqudate 2 days", fundingFee.getPendingFee(marketId, address(maker2)));
    }


Logs:
  maker2 PendingFee after liqudate  99 291029340535175000
  maker2 OpenNotional  after liqudate 5000000000000000000000
  maker2 PositionSize  after liqudate -50000000000000000000
  maker2 PendingFee after liqudate 2 days 139 007441076749245000
```
 

2.result in an increase of system bad debt.
https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/vault/PositionModelUpgradeable.sol#L86



## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/fundingFee/FundingFee.sol#L164
## Tool used

Manual Review

## Recommendation
update maket's `OpenNotional`  and `PositionSize`  after  liquidation
