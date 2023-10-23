Ancient Daffodil Caribou

medium

# old borrowing key is used instead of `newBorrowingKey` when adding old loans to the newBorrowing in LiquidityBorrowingManager.takeOverDebt()
## Summary
when `_addKeysAndLoansInfo()` is called within LiquidityBorrowingManager.takeOverDebt(), old Borrowing Key is used and not `newBorrowingKey`  see [here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L440-L441)


## Vulnerability Detail
The old borrowing key credentials are deleted in  `_removeKeysAndClearStorage(oldBorrowing.borrower, borrowingKey, oldLoans);` see  [here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L429)

And a new borrowing key is created with the holdToken, saleToken, and the address of the user who wants to take over the borrowing in the `_initOrUpdateBorrowing()`. see [here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L430-L439)


now the old borrowing key whose credentials are already deleted is used to update the old loans in `_addKeysAndLoansInfo()` instead of the `newBorrowingKey` generated in  `_initOrUpdateBorrowing()` see [here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L440-L441)

## Impact
wrong borrowing Key is used (i.e the old borrowing key) when adding old loans to `newBorrowing` 

Therefore the wrong borrowing key (i.e the old borrowing key) will be added as borrowing key for tokenId of old Loans in `tokenIdToBorrowingKeys` in _addKeysAndLoansInfo()

(i.e when the bug of `update bool` being false, is corrected, devs should understand :))

## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L440-L441
## Tool used

Manual Review

## Recommendation
use newBorrowingKey when calling  `_addKeysAndLoansInfo()` instead of old borrowing key.