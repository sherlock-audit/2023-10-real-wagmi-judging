Proud Mocha Mustang

medium

# Borrowing functionality for specific hold token may be dossed
## Summary
The borrowing functionality for specific or all hold tokens in the smart contract may be disrupted

## Vulnerability Detail
In the following code snippet borrowingCollateral is calculated by subtracting cache.holdTokenBalance from cache.borrowedAmount
```solidity
uint256 borrowingCollateral = cache.borrowedAmount - cache.holdTokenBalance;
```
The problem arises from the _getBalance function, which retrieves the balance of the contract using balanceOf(address(this)). This can be exploited by a malicious user who sends tokens to the contract, causing the borrow function to revert due to underflow
```solidity
function _getBalance(address token) internal view returns (uint256 balance) {
        bytes memory callData = abi.encodeWithSelector(IERC20.balanceOf.selector, address(this));
        (bool success, bytes memory data) = token.staticcall(callData);
        require(success && data.length >= 32);
        balance = abi.decode(data, (uint256));
    }
```
You might argue that a user could borrow a larger amount of tokens than repay and obtain the attacker's tokens. However, the attacker can monitor the mempool, and if such a situation occurs, they can simply repay the loan they took earlier and retrieve their tokens. 

Additionally, some tokens have low liquidity on Uniswap. If an attacker sends a significant number of tokens, another user may not be able to borrow a sum high enough to exceed their holdTokenBalance.

## Impact
Borrowing functionality for one or all hold tokens may be unavailable.

## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L492
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L113-L118

## Tool used
Manual Review

## Recommendation
Use a more secure method to check the contract's token balance to prevent external manipulation.