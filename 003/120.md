Big Eggshell Mink

medium

# `feesDebt` is charged twice rather than once
## Summary

Users who incur a `feesDebt` are charged twice for this `feesDebt`

## Vulnerability Detail

Let's say that a user calls borrow (`LiquidityBorrowingManager.sol`) once and then holds their position for a while, and then calls borrow again with the same `sellToken` and `holdToken`. Then, `_initOrUpdateBorrowing` will be called. Let's say the users position is now underwater. To be more specific, in the code lines:

```solidity
(int256 collateralBalance, uint256 currentFees) = _calculateCollateralBalance(
                borrowing.borrowedAmount,
                borrowing.accLoanRatePerSeconds,
                borrowing.dailyRateCollateralBalance,
                accLoanRatePerSeconds
            );
```
We have `collateralBalance` negative.

Then the following lines of code are executed:
```solidity
            if (collateralBalance < 0) {
                feesDebt = uint256(-collateralBalance) / Constants.COLLATERAL_BALANCE_PRECISION + 1;
                borrowing.dailyRateCollateralBalance = 0;
            }
```

And we also increment the fees owed `borrowing.feesOwed += currentFees;`. However, because the user has to pay the fees debt:

```solidity
        _pay(
            params.holdToken,
            msg.sender,
            VAULT_ADDRESS,
            borrowingCollateral + liquidationBonus + cache.dailyRateCollateral + feesDebt
        );
```

But it is also added to `borrowing.feesOwed` (since `feesDebt` is a part of `currentFees`), the borrower is actually being double charged, since when the user repays the loan they have to pay `borrowing.feesOwed` to the creditor (this is in `LiquidityManager.sol`):

```solidity
            uint256 liquidityOwnerReward = FullMath.mulDiv(
                params.totalfeesOwed,
                cache.holdTokenDebt,
                params.totalBorrowedAmount
            ) / Constants.COLLATERAL_BALANCE_PRECISION;

            Vault(VAULT_ADDRESS).transferToken(cache.holdToken, creditor, liquidityOwnerReward);
```

## Impact

Borrower is overcharged `feesDebt` fees, which could be large in the event that `feesDebt` is large, which is if the position goes very underwater due to the daily collateral interest rate. 

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L938-L947

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L498-L503

## Tool used

Manual Review

## Recommendation
One idea is to keep track of the total feesDebt that the user has paid with that `borrowKey` and transfer that back to the Vault upon `repay` before giving fees to the creditor. 