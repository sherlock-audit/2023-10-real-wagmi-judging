Restless Ocean Chipmunk

medium

# Platform fees is counted twice when takingOverDebt
## Summary

Platform fees is counted from the old borrow when a user takes over the debt. The fees is counted again when the user initializes a new borrow.

## Vulnerability Detail

When calling `takeOverDebt()`, `_pickUpPlatformFees()` is called and appended to the platformsFeesInfo mapping. Then, when `_initOrUpdateBorrowing()` is called in `takeOverDebt()`, `_pickUpPlatformFees()` is called again. The user that takes over the debt has to pay for the platform fees twice, and the original borrower does not have to pay for any platform fees. 

```solidity
            currentFees = _pickUpPlatformFees(oldBorrowing.holdToken, currentFees);
            oldBorrowing.feesOwed += currentFees;
```

```solidity
            // Pick up platform fees from the hold token's current fees
            currentFees = _pickUpPlatformFees(holdToken, currentFees);
            // Increment the fees owed in the borrowing position
            borrowing.feesOwed += currentFees;
```

## Impact

The person that takes over the borrow has to pay the platform fees twice.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L944-L947

## Tool used

Manual Review

## Recommendation

`_pickUpPlatformFees()` do not need to be called in `takeOverDebt()`, since `_initOrUpdateBorrowing()` already calls the fees.