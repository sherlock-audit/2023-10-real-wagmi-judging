Rough Pearl Wombat

medium

# `computePoolAddress()` will not work on ZkSync Era
## Summary

When using the wagmi protocol, multiple swap can happen when borrowing or repaying a position. When the swap uses Uniswap v3 it checks that the callback is a pool by computing the address but the computation won't match on ZkSync Era.

## Vulnerability Detail

When borrowing or repaying a position a user can either use a custom router that was approved by the wagmi team to make the swaps required or can use Uniswap v3 as a fallback.

When using the Uniswap v3 as a fallback the [`_v3SwapExactInput()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L204) internal function is being called. This function uses [`computePoolAddress()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L271) to find the pool address to use. [`computePoolAddress()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L271) is also used during the [`uniswapV3SwapCallback()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L242) to make sure the `msg.sender` is a valid pool.

On ZkSync Era the `create2` addresses are not computed the same way see [here](https://era.zksync.io/docs/reference/architecture/differences-with-ethereum.html#address-derivation).

This will result in the swaps on Uniswapv3 to revert. If a user was able to open a position using a custom router but the custom router is removed later on by the team or if the liquidity was one sided so no swap happened. The borrower and liquidators could find themself not able to close the positions until a new router is whitelisted.

The borrower could be forced to pay collateral for a longer time as he won't be able to close his position.

## Impact

Medium. Unlikely to happen but would result in short-term DOS and more fees paid by the borrower.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L146
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L204
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L271

## Tool used

Manual Review

## Recommendation

Consider calling the Uniswap factory getter `getPool()` to get the address of the pool.