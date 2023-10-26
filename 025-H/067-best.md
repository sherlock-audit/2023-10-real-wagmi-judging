Dandy Taupe Barracuda

high

# Collateral sent during borrowing is lost due to not accounting for borrowingCollateral in borrowing.DailyRateCollateralBalance
## Summary
During borrowing, a borrower has to pay some collateral in holdToken plus deposit for holding a position for 24 hours, which is a product of the `borrowedAmount` and the `currentDailyRate`. The latter is added to the collateral balance of the borrower, while the former is not. This makes the collateral stuck in the Vault.
## Vulnerability Detail
In the [`borrow()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L488-L503) function only the `dailyRateCollateral` is added to the `borrowing.DailyRateCollateralBalance`, while both `dailyRateCollateral` and `borrowingCollateral` are transferred to the system.

`borrowing.DailyRateCollateralBalance` is the collateral balance before fees accrual and it is used to determine if a loan is liquidatable or not by calculating the actual collateral balance of the loan (that is, collateral balance minus the fees accrued).
```solidity
            (collateralBalance, currentFees) = _calculateCollateralBalance(
                borrowing.borrowedAmount,
                borrowing.accLoanRatePerSeconds,
                borrowing.dailyRateCollateralBalance,
                accLoanRatePerSeconds
            );
```
The only way of retrieving collateral from the Vault is calling the `repay()` function which can be called is case of liquidation or closing a position by the borrower.

[In case of liquidation](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L573-L578) (i.e., when `borrowing.DailyRateCollateralBalance` minus the fees < 0), the `borrowing.DailyRateCollateralBalance` is added to borrowing.feesOwed after which tokens are transferred from the Vault and liquidity is being restored in uniswap position [in the call](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L650-L660) to `_restoreLiquidity()`, [during which](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L306-L315) fees owed to a position owner is also transferred from the Vault.

[In case of closing position](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L565-L572) by the borrower the same happens, only his `borrowing.DailyRateCollateralBalance` is applied not to borrowing.feesOwed, but to the liquidationBonus which he receives at the end of the call.

There is [a 3rd case of emergency closure](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L620-L624) by the uniswap position owner but it is the same in a sense that no more tokens are drawn from the Vault that are borrowed and owed.

This means that the `borrowingCollateral` which is transferred upon borrowing is lost in the system.
## Impact
Loss of funds.
## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L488-L490
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L492-L503
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L552-L557
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/DailyRateAndCollateral.sol#L103-L118
## Tool used

Manual Review

## Recommendation
Add borrowing collateral to the collateral balance
```solidity
borrowing.dailyRateCollateralBalance +=
            (cache.dailyRateCollateral *
            Constants.COLLATERAL_BALANCE_PRECISION) + (borrowingCollateral * Constants.COLLATERAL_BALANCE_PRECISION);
```