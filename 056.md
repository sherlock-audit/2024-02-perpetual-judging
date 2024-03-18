Refined Maroon Tarantula

medium

# Protocol is incompatible with USDT

## Summary
USDT will revert if the current allowance is greater than 0 and an non-zero approval is made. There are multiple instances throughout the contracts where this causes issues. In some places this can create scenarios where it becomes impossible to liquidate and/or borrow it.
## Vulnerability Detail
Check summary 
## Impact
Following functions will not be functional for USDT 

[Maker-OracleMaker::Deposit](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L217)
[Maker-SpotHedgeBaseMaker::Deposit & Uniswap methods](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L570-L615)

## Code Snippet
[Maker-OracleMaker::Deposit](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/OracleMaker.sol#L217)
[Maker-SpotHedgeBaseMaker::Deposit & Uniswap methods](https://github.com/sherlock-audit/2024-02-perpetual/blob/main/perp-contract-v3/src/maker/SpotHedgeBaseMaker.sol#L570-L615)

## Tool used

Manual Review

## Recommendation
Add approve 0 call before every approval like following 

```solidity

IERC_Interface(tokenAddress).approve(targetAddress,0);

```