medium

# QVStrategy : Funds locked if no registered recipient or no recipientStatus >= reviewThreshold
## Summary
Both RFP and Merkle strategies have an in-build `withdraw()` function that allow poolManagers to withdraw funds in case of problem/no recipient registration/no recipient acceptation.
However QVStrategy doesn't have one , so in case of no registration or no acceptation of a recipient before `registrationEndTime` is passed  , funds deposited on the strategy contract will be lock.

## Vulnerability Detail
As explained above, both RFP and Merkle strategies have withdraw() function : 

- DonationVotingMerkleDistributionBaseStrategy.sol : 
```solidity
    function withdraw(uint256 _amount) external onlyPoolManager(msg.sender) {
        //E must be one month after allocationEndTime
        if (block.timestamp <= allocationEndTime + 30 days) {
            revert INVALID();
        }
        IAllo.Pool memory pool = allo.getPool(poolId);
        if (_amount > poolAmount) { 
            revert INVALID();
        }
        poolAmount -= _amount;
        // Transfer the tokens to the 'msg.sender' (pool manager calling function)
        _transferAmount(pool.token, msg.sender, _amount);
    }
```

- RFPSimpleStrategy.sol : 
```solidity
    function withdraw(uint256 _amount) external onlyPoolManager(msg.sender) onlyInactivePool {
        // Decrement the pool amount
        poolAmount -= _amount;
        // Transfer the amount to the pool manager
        _transferAmount(allo.getPool(poolId).token, msg.sender, _amount);
    }
```
But QV strategy doesn't implement an in-built withdraw function, so the only process to distribute funds locked on a QV Strategy would be to have a recipient reviewed by `poolManager` more than `reviewThreshold` time before `registrationEndTime` :
```solidity
//E Add 1 Review for recipient(s) application(s) and set status of the corresponding recipient struct ONLY if it passes the threshold
    function reviewRecipients(address[] calldata _recipientIds, Status[] calldata _recipientStatuses) external virtual
        onlyPoolManager(msg.sender)  //E if hasRole(pools[_poolId].managerRole, _address) || _isPoolAdmin(_poolId, _address);
        onlyActiveRegistration
    {
        .....
        for (uint256 i; i < recipientLength;) {
            ....
            //E mapping(address => mapping(Status => uint256)) public reviewsByStatus; //E recipientId -> status -> count
            reviewsByStatus[recipientId][recipientStatus]++;

            //E check if it passed threshold of reviewing
            if (reviewsByStatus[recipientId][recipientStatus] >= reviewThreshold) { //E @question what if reviewThreshold change ? PoolManager has to resend this tx
                Recipient storage recipient = recipients[recipientId];
                recipient.recipientStatus = recipientStatus;
                emit RecipientStatusUpdated(recipientId, recipientStatus, address(0));
            }
           ...
        }
    }
```
And then have a poolManager calling `_distribute()` function using Allo.sol (this function can be called anytime after `allocationEndTime`)

**However if no recipient passes `reviewThreshold` or no recipient at all register for this strategy, funds will be locked on the strategy contract.**
(We could think about modifying timestamps using `updatePoolTimestamps()` but this function reverts `if block.timestamp > _registrationStartTime`)
**So once the process of registration is launch on a QV Strategy there is no other possibilities to unlock the funds than having a recipient registering and accepted before end of `registrationEndTime`, if one of these conditions failed, funds will be locked.**

I validate this problem with sponsor who answered "Ah you're right ! We should have a withdraw function. Good catch on that"

## Impact

If between `registrationStartTime` and `registrationEndTime` there is no accepted registration , funds will be locked on this contract with no way to unlock them.

## Code Snippet

https://github.com/allo-protocol/allo-v2/blob/main/contracts/strategies/qv-base/QVBaseStrategy.sol#L369

## Tool used

Manual Review

## Recommendation

Add a `withdraw()` function for `poolManagers` on the QV Strategy in the same model as there is one in RFP Strategy


NB : it is also true if valid allocator forgets to vote as amount sent is calculated in `_getPayout()` function only if `totalRecipientVotes != 0` and even if this case would be strange it would necessite a `withdraw()` function