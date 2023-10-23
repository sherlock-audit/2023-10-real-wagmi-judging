Festive Daffodil Grasshopper

medium

# Under the emergency liquidation state, malicious borrowers can retain the liquidationBonus
## Summary

Under the emergency liquidation state, if the current lenders liquidate the loan completely, who can receive a liquidationBonus.
However, the borrower can frontrun to borrow an additional loan again, causing the current lender to be unable to obtain the bonus. The borrower can completely designate a loan to his other account to retain the bonus.

## Vulnerability Detail

```solidity
            // If loansInfoLength is 0, remove the borrowing key from storage and get the liquidation bonus
            if (completeRepayment) {
                LoanInfo[] memory empty;
                _removeKeysAndClearStorage(borrowing.borrower, params.borrowingKey, empty);
                feesAmt += liquidationBonus;
            } else {
                BorrowingInfo storage borrowingStorage = borrowingsInfo[params.borrowingKey];
                borrowingStorage.dailyRateCollateralBalance = 0;
                borrowingStorage.feesOwed = borrowing.feesOwed;
                borrowingStorage.borrowedAmount = borrowing.borrowedAmount;
                // Calculate the updated accLoanRatePerSeconds
                borrowingStorage.accLoanRatePerSeconds =
                    holdTokenRateInfo.accLoanRatePerSeconds -
                    FullMath.mulDiv(
                        uint256(-collateralBalance),
                        Constants.BP,
                        borrowing.borrowedAmount // new amount
                    );
            }
```

It can be clearly seen from the code that the liquidationBonus will be issued only when completeRepayment is completed, which means that only the last lenderer can get the bonus.
The borrower can completely designate his own account as the lender to retain the bonus.

## Impact

The lender's rewards may be retained maliciously, reducing user incentives.

## Code Snippet

- https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L602C37-L605

## Tool used

Manual Review

## Recommendation

The liquidationBonus should be distributed to the lender in proportion
