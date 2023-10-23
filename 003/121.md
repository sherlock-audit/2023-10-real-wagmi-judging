Big Eggshell Mink

high

# Borrowers are overcharged fees because both `borrowing.dailyRateCollateralBalance` is decremented and `borrowing.feesOwed` is incremented
## Summary

Currently, we update the fees that a user owes in `_initOrUpdateBorrowing`. Later on, the `dailyRateCollateralBalance` is transferred to the `LiquidityBorrowingManager`, fees are transferred to the creditor from the `LiquidityBorrowingManager`, and then the remaining tokens are transferred to the borrower. However, because we both decrement `borrowing.dailyRateCollateralBalance` and increment `borrowing.feesOwed`, the borrower ends up being double charged for the fees. 
 
## Vulnerability Detail

Let's say that a user has called borrow once, and then calls borrow again in `LiquidityBorrowingManager`. Then,  `_initOrUpdateBorrowing` in `LiquidityBorrowingManager` is called. Let's say the position isn't underwater, so we eventually reach:

```solidity
borrowing.dailyRateCollateralBalance -= currentFees;
```

However, we also call `borrowing.feesOwed += currentFees;`. 

Then, when we go to repay, we have the following code snippet:

```solidity
(collateralBalance, currentFees) = _calculateCollateralBalance(
                borrowing.borrowedAmount,
                borrowing.accLoanRatePerSeconds,
                borrowing.dailyRateCollateralBalance,
                accLoanRatePerSeconds
            );

            (msg.sender != borrowing.borrower && collateralBalance >= 0).revertError(
                ErrLib.ErrorCode.INVALID_CALLER
            );

            // Calculate liquidation bonus and adjust fees owed

            if (
                collateralBalance > 0 &&
                (currentFees + borrowing.feesOwed) / Constants.COLLATERAL_BALANCE_PRECISION >
                Constants.MINIMUM_AMOUNT
            ) {
                liquidationBonus +=
                    uint256(collateralBalance) /
                    Constants.COLLATERAL_BALANCE_PRECISION;
            } else {
                currentFees = borrowing.dailyRateCollateralBalance;
            }
```

and:

```solidity
            Vault(VAULT_ADDRESS).transferToken(
                borrowing.holdToken,
                address(this),
                borrowing.borrowedAmount + liquidationBonus
            );
```

The TLDR here is that some adjusted version (for fees) of the `borrowing.dailyRateCollateralBalance` is added to `liquidationBonus`, and then `borrowing.borrowedAmount + liquidationBonus` is transferred to the Vault. Later, in LiquidityManager.sol, we have:

```solidity
            uint256 liquidityOwnerReward = FullMath.mulDiv(
                params.totalfeesOwed,
                cache.holdTokenDebt,
                params.totalBorrowedAmount
            ) / Constants.COLLATERAL_BALANCE_PRECISION;

            Vault(VAULT_ADDRESS).transferToken(cache.holdToken, creditor, liquidityOwnerReward);
```

So, an already lower amount of token (lower because `currentFees` was subtracted from `borrowing.dailyRateCollateralBalance` as we saw above) is being transferred to the vault, but then `currentFees` is transferred out again to the creditor. The borrower is therefore being charged twice for the fee.  

## Impact

Borrower is charged twice for the fees

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L942-L947

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L548-L636

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L315

## Tool used

Manual Review

## Recommendation
I would recommend you just keep track of the total amount of `holdToken` that's been transferred in to date by the borrower, and then send that amount back to the `Vault` when `repay` is called. The fees can then be charged on this amount and other processing can also be done on this amount. 