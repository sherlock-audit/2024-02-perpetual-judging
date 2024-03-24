Bouncy Ebony Shark

high

# Loss of gas yield for protocol across all contracts deployed on Blast

## Summary
Based on the protocol provided context, perpetual is intended to be deployed on Optimism/Blast.

Blast has the unique implementation of directing sequencer fees to contracts that uses fees. Default gas mode for all contracts is VOID. This means that none of the accumulated sequencer gas yield will be shared with these contracts regardless of the amount of usage. In order for contracts to partake in this revenue sharing model, they will need to interact with Blast precompile at `0x4300000000000000000000000000000000000002` to change their gas mode.  

## Vulnerability Detail
Because perpetual protocol does not have any way to set its gas mode, it is unable to accrue any yield generated from users interacting with their contracts.

## Impact
Massive loss on gas yield across all contracts deployed on Blast.

## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/vault/Vault.sol#L27

## Tool used

Manual Review

## Recommendation
Add Blast wrappers to the current contracts so that gas mode can be set and gas yields can be claimed.

