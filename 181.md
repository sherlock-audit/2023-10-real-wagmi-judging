Rough Pearl Wombat

medium

# `MINIMUM_AMOUNT` will result in higher rate for tokens with low decimals
## Summary

The WAGMI contract has a `MINIMUM_AMOUNT` constant that is used to define minimum on certain amounts in different functions.

It is currently `1000` which will result in higher value than expected for tokens will low decimals like GUSD (2 decimals).

## Vulnerability Detail

The [`borrow()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L465) function will charge a minimum of `dailyRateCollateral` of `MINIMUM_AMOUNT`. This means that if we were to send the `MINIMUM_BORROWED_AMOUNT` which is `100000`.

We would result in 1000 / 100000 * 100 = 1.
1% of the borrowed amount would be charged as collateral, in the case of GUSD which has 2 decimals it would mean that a loan of 1000 GUSD would make us pay 1% even if the real rate is smaller.

The [`getLiquidationBonus()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L683C6-L683C6) function will also charge us a minimum of `MINIMUM_AMOUNT` even if real rate is smaller.

And if we were to close our position using [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) we wouldn't get back the collateral balance if the fees charged is less than `MINIMUM_AMOUNT`.

## Impact

Medium. When using protocol with low decimals tokens like GUSD, unexpected fees and losses can arise.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L465
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L683C6-L683C6
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532

## Tool used

Manual Review

## Recommendation

Consider computing `MINIMUM_AMOUNT` and `MINIMUM_BORROWED_AMOUNT` with the token's decimals.
