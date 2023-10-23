Big Eggshell Mink

high

# Borrower collateral that they are owed can get stuck in Vault and not sent back to them after calling `repay`
## Summary

There's a case where a borrower calls `borrow`, perhaps does a bunch of intermediate actions like calling `increaseCollateralBalance`, and then calls `repay` a short while later (so fees haven't had a time to increase), but the collateral they are owed is stuck in the `Vault` instead of being sent back to them after they repay. 

## Vulnerability Detail

First, let's say that a borrower called `borrow` in `LiquidityBorrowingManager`. Then, they call increase `increaseCollateralBalance` with a large collateral amount. A short time later, they decide they want to repay so they call `repay`. 

In `repay`, we have the following code:

```solidity
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

Notice that if we have `collateralBalance > 0` BUT `!((currentFees + borrowing.feesOwed) / Constants.COLLATERAL_BALANCE_PRECISION >
                Constants.MINIMUM_AMOUNT)` (i.e. the first part of the if condition is fine but the second is not. It makes sense the second part is not fine because the borrower is repaying not long after they borrowed, so fees haven't had a long time to accumulate), then we will still go to `currentFees = borrowing.dailyRateCollateralBalance;` but we will not do:

```solidity
                liquidationBonus +=
                    uint256(collateralBalance) /
                    Constants.COLLATERAL_BALANCE_PRECISION;
```

However, later on in the code, we have:

```solidity
            Vault(VAULT_ADDRESS).transferToken(
                borrowing.holdToken,
                address(this),
                borrowing.borrowedAmount + liquidationBonus
            );
``` 

So, the borrower's collateral will actually not even be sent back to the LiquidityBorrowingManager from the Vault (since we never incremented `liquidationBonus`). We later do:

```solidity
            _pay(borrowing.holdToken, address(this), msg.sender, holdTokenBalance);
            _pay(borrowing.saleToken, address(this), msg.sender, saleTokenBalance);
```

So clearly the user will not receive their collateral back. 

## Impact
User's collateral will be stuck in Vault when it should be sent back to them. This could be a large amount of funds if for example `increaseCollateralBalance` is called first. 

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L565-L575

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L632-L670

## Tool used

Manual Review

## Recommendation

You should separate:

```solidity
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

Into two separate if statements. One should check if `collateralBalance > 0`, and if so, increment liquidationBonus. The other should check `(currentFees + borrowing.feesOwed) / Constants.COLLATERAL_BALANCE_PRECISION >
                Constants.MINIMUM_AMOUNT` and if not, set `currentFees = borrowing.dailyRateCollateralBalance;`. 