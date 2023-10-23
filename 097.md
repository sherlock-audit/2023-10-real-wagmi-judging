Rough Pearl Wombat

medium

# No check on liquidation and daily rates update while borrowing
## Summary

When a user calls the [`borrow()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L465) function of the LiquidityBorrowingManager contract. He expect that the `currentDailyRate` and `liquidationBonusForToken` are the same as what was displayed by the UI.

But given asynchronous nature of blockchains, an admin could update these values for the tokens the borrower is about to borrow and result in more token charged than expected.

## Vulnerability Detail

During the [`borrow()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L465) function, the token to be sent by the borrower are computed using multiple state variables that define the rates and default amounts for the tokens to borrow.

These variables can be modified by the `owner/dailyRateOperator`. While the function check the `deadline` and `maxCollateral` given by the borrower. It doesn't check that the `currentDailyRate` nor `liquidationBonusForToken` changes.

If these values are changed while the transaction is being confirmed, it could result in more tokens charged to the user than he expected.

## Impact

Medium. If some state parameters are changed while a borrower's transaction is being confirmed it could result in more token charged and a different daily fee than expected.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L465
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L211
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/OwnerSettings.sol#L65

## Tool used

Manual Review

## Recommendation

Consider adding `maxDailyRate` and `maxLiquidationBonus` parameters and check them.
