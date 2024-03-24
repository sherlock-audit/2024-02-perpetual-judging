Melted Eggplant Bird

medium

# Nonces are not used in the signature checks

## Summary
Nonces are not used in the signature checks

## Vulnerability Detail
nonces can prevent an old value from being used when a new value exists. Without one, two transactions submitted in one order, can appear in a block in a different order
## Impact
a malicious miner can change the order of the transactions,

also the singed data doesn't include chainId which is also can cause reuse of the signed data in different forked chains 
## Code Snippet
```solidity
  function getOrderHash(Order memory order) public view returns (bytes32) {
        return _hashTypedDataV4(keccak256(abi.encode(ORDER_TYPEHASH, order)));
    }

```
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/orderGatewayV2/OrderGatewayV2.sol#L261
## Tool used

Manual Review

## Recommendation
- Include a nonce in what is signed and check it properly. 
- consider check chainId against chainId and recomputes the
DOMAIN_SEPARATOR if they are different.
