Rough Pearl Wombat

medium

# No deadline and slippage check on `takeOverDebt()` can lead to unexpected results
## Summary

The function [`takeOverDebt()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L395) in the LiquidityBorrowingManager contract doesn't have any deadline check like [`borrow`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L465) or [`repay`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532).

Additionally it also doesn't have any "slippage" check that would make sure the position hasn't changed between the moment a user calls [`takeOverDebt()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L395) and the transaction is confirmed.

## Vulnerability Detail

Blockchains are asynchronous by nature, when sending a transaction, the contract targeted can see its state changed affecting the result of our transaction.

When a user wants to call [`takeOverDebt()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L395) he expects that the debt he is gonna take over will be the same (or very close) as when he signed the transaction.

By not providing any deadline check, if the transaction is not confirmed for a long time then the user might end up with a position that is not as interesting as it was.

Take this example:

- A position on Ethereum is in debt of collateral but the borrowed tokens are at profit and the user think the token's price is going to keep increasing, it makes sense to take over.
- The user makes a transaction to take over.
- The gas price rise and the transaction takes longer than he thoughts to be confirmed.
- Eventually the transaction is confirmed but the position is not in profit anymore because price changed during that time.
- User paid the debt of the position for nothing as he won't be making profits.

Additionally when the collateral of a position is negative, lenders can call [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) as part of the "emergency liquidity restoration mode" which will reduce the size of the position. If this happens while another user is taking over the debt then he might end up with a position that is not as interesting as he thoughts.

Take this second example:

- A position on Ethereum with 2 loans is in debt of collateral but the borrowed tokens are at profit and the user think the token's price is going to keep increasing, it makes sense to take over.
- The user makes a transaction to take over.
- While the user's transaction is in the MEMPOOL a lender call ['repay()'](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) and get back his tokens.
- The user's transaction is confirmed and he take over the position but it only has 1 loan now as one of the 2 loans was sent back to the lender. the position might not be at profit anymore or less than it was supposed to be.
- User paid the debt of the position for nothing as he won't be making profits.

## Impact

Medium. User might pay collateral debt of a position for nothing.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L395

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L581

## Tool used

Manual Review

## Recommendation

Consider adding the modifier `checkDeadline()` as well as a parameter `minBorrowedAmount` and compare it to the current `borrowedAmount` to make sure no lender repaid their position during the take over.