Ancient Malachite Jay

medium

# Pragma isn't compatible with Arbitrum and other rollups that don't support Push0
## Summary

`pragma` has been set to `0.8.21`. The problem with this is that Arbitrum and other rollups are [NOT compatible](https://developer.arbitrum.io/solidity-support) with `0.8.20` and newer since it doesn't support Push0. Contracts compiled with this version will result in a nonfunctional or potentially damaged version that won't behave as expected.

## Vulnerability Detail

See summary

## Impact

Damaged or nonfunctional contracts when deployed on Arbitrum and other rollups

## Code Snippet

[LiquidityBorrowingManager.sol#L2](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L2)

## Tool used

Manual Review

## Recommendation

Use a lower `pragma` such as `0.8.19`