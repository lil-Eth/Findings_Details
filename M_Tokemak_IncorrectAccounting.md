lil.eth

medium

# Incorrect Accounting of newRewards in the queueNewRewards() function of AbstractRewarder Contract
## Summary
In rewarders/AbstractRewarder.sol#queueNewRewards() function, `newRewards` is incorrectly accounted. This leads to incorrect calculation of `currentRate`  and an incorrect transfer of tokens when `safeTransferFrom` is invoked, as the contract attempts to fetch `newRewards + queuedRewards`, while `queuedRewards` has already been fetched earlier and each time this function is called it tries to fetch all `newRewards + queuedRewards` which will end in a failed transaction.

## Vulnerability Detail
The `queueNewRewards()` function is called from a liquidation process in `LiquidationRow.sol#liquidateVaultsForToken()` and it is supposed to queue a specified amount of new rewards for distribution to stakers . The function takes `newRewards` as a parameter, which represents the new rewards to be queued.

```solidity
function queueNewRewards(uint256 newRewards) external onlyWhitelisted {
    uint256 startingQueuedRewards = queuedRewards;
    uint256 startingNewRewards = newRewards;
    newRewards += startingQueuedRewards;
    ......
    IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards);
}
```
The issue arises because `newRewards` is incremented by `startingQueuedRewards`, which represents the rewards that are currently queued. This increment makes the `newRewards` variable larger than it should be.

Later in the code, safeTransferFrom is invoked to fetch newRewards into the contract.
```solidity
IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards); //E from LiquidationRow.sol
```
Since `newRewards` was incorrectly incremented, the contract attempts to fetch `newRewards + queuedRewards`, which is more than the actual amount of new rewards and more than what is approved in LiquidationRow.sol when queueing rewards : 
```solidity
function _performLiquidation(
        uint256 gasBefore, //E Amount of gas when the liquidliquidateVaultsForToken function was called
        address fromToken, //E token that needs to be liquidated
        address asyncSwapper, //E address of the async swapper contract
        IDestinationVault[] memory vaultsToLiquidate, //E list of vaults that need to be liquidated
        SwapParams memory params, //E Parameters for the async swap
        uint256 totalBalanceToLiquidate, //E total balance that needs to be liquidated
        uint256[] memory vaultsBalances //E balances of the vaults
    ) private {
        ... 
        for (uint256 i = 0; i < length; ++i) {
            ...
            uint256 amount = amountReceived * vaultsBalances[i] / totalBalanceToLiquidate; 
            if (amount == 0) {continue }
            //E approve main rewarder to pull the newRewards tokens
            LibAdapter._approve(IERC20(params.buyTokenAddress), address(mainRewarder), amount);
            //E Rewards are added to a reward queue, which is then distributed to stakers based on their staked balances
            mainRewarder.queueNewRewards(amount);
        ...
    }
```

All of this will lead to an exception if the sender (LiquidationRow.sol) does not have enough balance or does not have enough approval to cover the incorrectly inflated newRewards.


## Impact
Too much rewards fetch from LiquidationRow leading to a DOS for all reward process if it does not have enough balance  or approval to cover the error

## Code Snippet
https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/AbstractRewarder.sol#L235

## Tool used

Manual Review

## Recommendation
To resolve this issue, the `newRewards` variable should not be incremented by `queuedRewards`. Instead, `newRewards` should be handled separately, and `queuedRewards` should be managed in a way that ensures it is not fetched again.

```solidity
function queueNewRewards(uint256 _newRewards) external onlyWhitelisted {
    uint256 startingQueuedRewards = queuedRewards;
    uint256 startingNewRewards = _newRewards;
    queuedRewards += _newRewards;
    ......
    IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), _newRewards);
}
```

OR it should be done as it is said in the description of the function and First transfer new rewards from the caller
```solidity
    /**
     * @notice Queues the specified amount of new rewards for distribution to stakers.
     * @param newRewards The amount of new rewards.
     * @dev First, the function transfers the new rewards from the caller to this contract, @audit false
        // on LiquidationRow       
             //E approve main rewarder to pull the tokens
            LibAdapter._approve(IERC20(params.buyTokenAddress), address(mainRewarder), amount);
            //E Rewards are added to a reward queue, which is then distributed to stakers based on their staked balances
            mainRewarder.queueNewRewards(amount);
     
     *      ensuring that the deposited amount matches the declared rewards.
     *      Then, irrespective of whether we're near the start or the end of a reward period, if the accrued rewards
     *      are too large relative to the new rewards (i.e., queuedRatio is greater than newRewardRatio), the new
     *      rewards will be added to the queue rather than being immediately distributed.
     */
function queueNewRewards(uint256 newRewards) external onlyWhitelisted {
     IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards);
     ....
}
```