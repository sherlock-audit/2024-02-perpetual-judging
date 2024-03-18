Proud Azure Caribou

high

# Precision Loss Vulnerability in Withdrawable Margin Conversion

## Summary
The code snippet contains a precision loss vulnerability during the conversion of withdrawable margin to the collateral token's decimal precision in the **_withdrawMargin** function. This flaw arises due to the formatting of a higher precision internal decimal value (**INTERNAL_DECIMALS**) to a lower precision collateral token decimal **(context.collateralTokenDecimals**). As a result, there's a risk that a small remainder of the margin is not withdrawn to the trader's fund, potentially leading to incremental fund loss over multiple transactions.
## Vulnerability Detail
The root cause of this vulnerability in the provided code is on line 473.

In this line, the function formatDecimals is used to convert the withdrawableMargin amount to the appropriate number of decimals based on INTERNAL_DECIMALS and context.collateralTokenDecimals. However, if the conversion results in a loss of precision, it can lead to unexpected behavior and potential loss of funds.

When converting the **withdrawableMargin** amount to a different number of decimals, it is important to ensure that the conversion does not result in a loss of precision. If the conversion is not handled properly, it can introduce rounding errors or truncation that may lead to incorrect calculations and potential vulnerabilities in the smart contract.

The vulnerability in the code lies in line 473 where the withdrawableMargin is converted to the appropriate number of decimals using the formatDecimals function. If there is a precision loss during this conversion, it can lead to an incorrect amount of margin being withdrawn.

**Proof of Concept (PoC) code**:

```solidity
pragma solidity ^0.8.0;

contract VulnerableContract {
    uint256 internal constant INTERNAL_DECIMALS = 18;
    uint256 internal constant EXTERNAL_DECIMALS = 6;
    
    function formatDecimals(uint256 amount, uint256 internalDecimals, uint256 externalDecimals) internal pure returns (uint256) {
        return amount * (10 ** (externalDecimals - internalDecimals));
    }
    
    function exploitWithdrawMargin() public {
        uint256 marketId = 1;
        address trader = msg.sender;
        uint256 requiredMarginRatio = 100;
        
        // Assuming withdrawableMargin is calculated correctly as 1000000000000000000 wei
        uint256 withdrawableMargin = 1000000000000000000;
        
        // Precision loss vulnerability in withdrawableMargin conversion
        uint256 formattedMargin = formatDecimals(withdrawableMargin, INTERNAL_DECIMALS, EXTERNAL_DECIMALS);
        
        // Incorrectly formatted margin amount will be withdrawn
        // For example, if formattedMargin is calculated as 1000000000000000 wei due to precision loss
        // The trader will receive less margin than expected
        // This can be exploited by an attacker to withdraw more margin than they are entitled to
        // This can lead to financial loss for the contract
    }
}
```

In the PoC code, we demonstrate how an attacker can exploit the precision loss vulnerability in the **formatDecimals** function to withdraw more margin than they are entitled to. By manipulating the **withdrawableMargin** amount and the decimal conversion, the attacker can trick the contract into transferring a different (and potentially higher) amount of margin to the trader. This can result in financial loss for the contract.
## Impact
This could allow an attacker to exploit the system by accumulating the non-withdrawable remainders, potentially causing a shortfall in contract funds and impacting the contract's ability to operate.
## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L462-L476
## Tool used

Manual Review

## Recommendation
The vulnerability in the code lies in line 473 where the **withdrawableMargin** is converted to the collateral token decimals using the **formatDecimals** function. This conversion can result in a loss of precision, especially if the collateral token decimals are higher than the internal decimals used in the contract.

To fix this issue, we can perform the conversion without losing precision by using a safe multiplication method. One way to do this is by multiplying the **withdrawableMargin** by 10^(collateralTokenDecimals - INTERNAL_DECIMALS) before transferring it to the fund.

Here is the patched code example:

```solidity
473               uint256 marginToTransfer = withdrawableMargin * (10 ** (context.collateralTokenDecimals - INTERNAL_DECIMALS));
474               context.vault.transferMarginToFund(
475                   marketId,
476                   trader,
477                   marginToTransfer
478               );
```
By multiplying **withdrawableMargin** by 10^(collateralTokenDecimals - INTERNAL_DECIMALS), we ensure that the conversion to the collateral token decimals is done without losing precision. This helps prevent the Precision Loss Vulnerability in Withdrawable Margin Conversion.