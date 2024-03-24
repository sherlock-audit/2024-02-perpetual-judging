Generous Smoke Donkey

high

# Loss of funds for trader because whitelisted maker can't be liquidated

## Summary

In the current implementation, a whitelisted maker cannot be liquidated, thus it can accumulate losses even after the available margin is exhausted. This leads to losses for the traders because they won't be able to close their profitable positions due to a revert on _checkMarginRequirement.

## Vulnerability Detail

Scenario:

1) A trader opens a long position against a whitelisted maker.
2) After some time, the price increases significantly. At this point, there are more long positions open than short positions, so this maker incurs large losses. The margin is not sufficient to cover them. However, the maker cannot be liquidated and continues to accumulate losses.
3) The trader decides to close their position and withdraw their profit. However, this cannot happen because the _closePositionFor function calls _checkMarginRequirement for the maker. vault.getFreeCollateralForTrade is < 0, leading to a revert. The trader cannot close their position. The only solution is for LPs to deposit additional collateral to cover the losses, but it doesn't make sense to deposit funds to cover losses.
4) The price decreases, and the trader loses their profit. Due to fees or a sudden price drop, the trader may also lose part of the margin.

In the described circumstances, the trader takes the risk by opening a position, but there is no way to close it and withdraw the profit. Thus, instead of gaining from the winning position, they incur losses.

<details>
<summary>POC</summary>

```solidity

    function testNotEnoughBalancePnlPool() public 
    {
        _deposit(marketId, taker1, 10000e6);

        vm.prank(taker1);
        clearingHouse.openPosition(
            IClearingHouse.OpenPositionParams({
                marketId: marketId,
                maker: address(maker),
                isBaseToQuote: false,
                isExactInput: false,
                amount: 1000 ether,
                oppositeAmountBound: 100000 ether,
                deadline: block.timestamp,
                makerData: ""
            })
        );

        console.log("Pnl pool balance: %d", vault.getPnlPoolBalance(marketId));

        maker.setBaseToQuotePrice(111e18);
        maker2.setBaseToQuotePrice(111e18);
        _mockPythPrice(111, 0);

        console.log("getFreeCollateralForTrade:");
        console.logInt(vault.getFreeCollateralForTrade(marketId, address(maker), 111e18, MarginRequirementType.MAINTENANCE));

        vm.prank(taker1);
        //this will revert with NotEnoughFreeCollateral error
        clearingHouse.closePosition(
            IClearingHouse.ClosePositionParams({
                marketId: marketId,
                maker: address(maker),
                oppositeAmountBound: 1000 ether,
                deadline: block.timestamp,
                makerData: ""
            })
        );
    }
```

</details>

## Impact

Loss of funds for the trader + broken core functionality because of inability to close a position.

## Code Snippet

https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L267-L356

https://github.com/sherlock-audit/2024-02-perpetual/blob/02f17e70a23da5d71364268ccf7ed9ee7cedf428/perp-contract-v3/src/clearingHouse/ClearingHouse.sol#L488-L524

## Tool used

Manual Review

## Recommendation

The best solution is to implement liquidation for whitelisted maker. If not possible, you can mitigate the issue with larger margin requirement for whitelisted makers.
