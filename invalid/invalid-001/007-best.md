Proud Azure Caribou

high

# Potential Undercollateralization Due to Incorrect Share Calculation with Fee-on-Transfer Tokens

## Summary
The deposit function in the provided smart contract is susceptible to a protocol insolvency vulnerability due to potential mismanagement of fee-on-transfer tokens. The function calculates the number of shares to mint based on the expected transfer amount of **amountXCD** rather than the actual amount received by the vault after the transfer is completed. If the **collateralToken** used is a fee-on-transfer token, the fees deducted during the transfer would result in a lower amount being received than the **amountXCD** specified.
## Vulnerability Detail
The root cause of the vulnerability "Potential Undercollateralization Due to Incorrect Share Calculation with Fee-on-Transfer Tokens" in the provided code is that the calculation of shares in the deposit function does not take into account the possibility of fee-on-transfer tokens.

In line 206, the calculation of shares is done based on the transferred amount of the collateral token, without considering any potential fees that might be deducted during the transfer. This can lead to a discrepancy between the actual amount of collateral received and the calculated shares minted, resulting in potential undercollateralization of the system.

An attacker can exploit this vulnerability by using a fee-on-transfer token for collateral. When the attacker deposits the token, the transfer function will deduct a fee from the deposited amount. However, the shares calculation does not consider this fee, leading to a discrepancy between the actual amount of collateral received by the contract and the amount used to calculate the shares.

To exploit this vulnerability, an attacker can create a fee-on-transfer token where a percentage of the transferred amount is burned or sent to another address. The attacker can then deposit this token into the contract, causing the shares calculation to be based on the full amount before the fee deduction. This results in the attacker receiving more shares than they should based on the actual collateral deposited.

**Proof of Concept (PoC) Code:**

1. Create a fee-on-transfer token contract with a transfer function that deducts a fee:

```solidity
contract FeeOnTransferToken {
    function transfer(address to, uint256 amount) external {
        // Deduct a fee from the transferred amount
        uint256 fee = amount * 5 / 100; // 5% fee
        uint256 amountAfterFee = amount - fee;
        
        // Transfer the reduced amount
        _transfer(msg.sender, to, amountAfterFee);
        
        // Burn the fee or send it to another address
        _burn(msg.sender, fee);
    }
}
```
2. Deploy the **FeeOnTransferToken** contract and obtain the token address.
3. Call the deposit function in the vulnerable contract with the **FeeOnTransferToken** address as the collateral token. Deposit a certain amount of tokens.
4. Verify that the shares received are calculated based on the full amount before the fee deduction, leading to undercollateralization.

By following these steps, the attacker can exploit the vulnerability in the smart contract due to incorrect share calculation with fee-on-transfer tokens.

## Impact
This discrepancy leads to the minting of an incorrect number of shares, which, over time, could cause the vault to become undercollateralized. If not enough assets are present to back the minted shares, this could lead to the vault's insolvency and its inability to fulfill the redemption of shares at their expected value, posing a high risk to the protocol's integrity and user funds.

## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L175-L226
## Tool used

Manual Review

## Recommendation
To fix this issue, the calculation of shares should be adjusted to include the fee deducted from the transferred amount. One way to do this is to calculate the actual amount transferred after the fee deduction and use that amount in the share calculation.

Here is an example of a patch code to fix the vulnerability:

```solidity
206           uint256 fee = calculateFee(amountXCD); // Calculate the fee deducted from the transferred amount
207           uint256 transferredAmountAfterFee = amountXCD - fee; // Calculate the actual amount transferred after the fee deduction
208           uint256 vaultValueXShareDecimals = _getVaultValueSafe(vault, price).formatDecimals(
209               INTERNAL_DECIMALS,
210               shareDecimals
211           );
212           uint256 amountXShareDecimals = transferredAmountAfterFee.formatDecimals(collateralToken.decimals(), shareDecimals);
213           shares = (amountXShareDecimals * totalSupply()) / vaultValueXShareDecimals;
```


In this patch, we calculate the fee deducted from the transferred amount and then adjust the share calculation based on the actual amount transferred after the fee deduction. This ensures that the share calculation takes into account the fee-on-transfer tokens and prevents potential undercollateralization due to incorrect share calculation.