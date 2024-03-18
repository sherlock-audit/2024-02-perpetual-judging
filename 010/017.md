Future Brunette Pig

high

# Anyone can claim Locked tokens due to lack of access control

## Summary
The `claimLockedFunds` function of the [`CircuitBreaker.sol`](https://github.com/sherlock-audit/2024-02-perpetual/blob/d1f68ffb481384ed604608fe25899635445d5311/perp-contract-v3/src/circuitBreaker/CircuitBreaker.sol#L141C14-L141C30) lacks access control

## Vulnerability Detail
The `claimLockedFunds` function allow users to claim their locked funds in case of an emergency locking of the funds due to an attack or anything that triggered the threshold set, and user funds will be locked until after an admin unlock these funds.
What can go wrong here? well, the function as we can observe lacks access control :
```solidity
function claimLockedFunds(address _asset, address _recipient) external onlyOperational {
        if (lockedFunds[_recipient][_asset] == 0) revert LibError.NoLockedFunds();
        if (isRateLimited) revert LibError.RateLimited();

        uint256 amount = lockedFunds[_recipient][_asset];
        lockedFunds[_recipient][_asset] = 0;

        emit LockedFundsClaimed(_asset, _recipient);

        _safeTransferIncludingNative(_asset, _recipient, amount);
    }
```
This will allow anyone to claim locked funds that does not belong to them.

**Attack scenario**
1. Alice tokens got locked during an attack or anything that trigger the lock
2. Malicious Bob immediately called the `claimLockedFunds` function with alice's address as the recipient and drained the funds
3. Now alice called the same function , the call will revert due to the check here:
 ```solidity
 if (lockedFunds[_recipient][_asset] == 0) revert LibError.NoLockedFunds();
```



## Impact
This will allow draining of the entire locked funds if the attacker claimed before the original owners and by just knowing their addresses

## Code Snippet
https://github.com/sherlock-audit/2024-02-perpetual/blob/d1f68ffb481384ed604608fe25899635445d5311/perp-contract-v3/src/circuitBreaker/CircuitBreaker.sol#L141-L151

## Tool used

Manual Review

## Recommendation
Add `msg.sender` instead of the recipient as the parameter 
