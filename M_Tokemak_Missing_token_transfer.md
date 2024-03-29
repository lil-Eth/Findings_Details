lil.eth

medium

# Missing token transfer between LiquidationRow.sol and BaseAsyncSwapper.sol during liquidation of Vaults For Token
## Summary
`liquidateVaultsForToken()` function in LiquidationRow.sol calls the `swap()` function in BaseAsyncSwapper.sol for converting assets. However, After a discussion with sponsor we identified that it doesn't transfer the necessary tokens to IAsyncSwapper before initiating the swap. This will result in a failure for the swap operation due to an insufficient balance in IAsyncSwapper.

## Vulnerability Detail

1. Token Transfer Missing:
In `LiquidationRow.sol`, when the `_performLiquidation()` function calls the `swap()` function of `IAsyncSwapper`, the necessary tokens (`sellAmount`) are not transferred to IAsyncSwapper.
Code :
```solidity
uint256 amountReceived = IAsyncSwapper(asyncSwapper).swap(params);
```
2. Checking Balance in IAsyncSwapper:
In `BaseAsyncSwapper.sol`, the `swap()` function checks the balance of the token to be sold (using `sellToken.balanceOf(address(this))`). If the tokens are not previously transferred, this balance check will result in an insufficient balance, leading to the InsufficientBalance error.
Code :
```solidity
uint256 sellTokenBalance = sellToken.balanceOf(address(this));
if (sellTokenBalance < swapParams.sellAmount) {
    revert InsufficientBalance(sellTokenBalance, swapParams.sellAmount);
}
```

## Impact

- Asset Lock: If tokens are not sent to IAsyncSwapper, the swap operation will consistently fail due to an insufficient balance and as **there is no way to recover tokens from liquidationRow.sol , tokens will be locked.**
- Operational Disruption: Regular **operations of liquidating vaults will be halted** due to this oversight, affecting the functionality and efficiency of the entire platform.

## Code Snippet

https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/BaseAsyncSwapper.sol#L28

https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/LiquidationRow.sol#L251

## Tool used

Manual Review

## Recommendation

Either `sellToken.safeTransfer(sellAmount)` or approve asyncSwapper to `sellToken.safeTransferFrom(sellAmount)` before calling `IAsyncSwapper(asyncSwapper).swap(params)` 