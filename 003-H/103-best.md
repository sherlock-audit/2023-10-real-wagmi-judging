Rough Pearl Wombat

high

# `repay()` is prone to sandwich attacks
## Summary

The [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) function of the LiquidityBorrowingManager contract is used to close a borrowing position.

When called by the borrower itself or by a liquidator bot, the amount of token received by the caller can be less than it should if the transaction is sandwiched. Greatly reducing potential profits for a borrower or liquidator.

## Vulnerability Detail

When a position is at profit, the lender can call [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) to close it. If the collateral if negative a liquidator can also call this function to reimburse collateral and profit from the `liquidationBonus` as well as the position's profits.

To close the position and realize the profits, if a the position is not out of range, a swap is made. Some `holdToken` are swapped for `saleToken` to be able to add the liquidity back to the position in the function [`_restoreLiquidity()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L223).

The current issue is that the minimum amount out of the swap is determined onchain which makes sandwich attacks possible as an MEV bot could bundle our transaction with 2 swaps one before and one after to affect the amount we receive on our swap effectively sandwiching out transaction.

The bot would make profits from our unrealized profits leading to the borrower or the liquidator to make less or even 0 profits.

## Impact

High. If the `repay()` transaction is sandwiched the profits will be greatly reduced.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L223

## Tool used

Manual Review

## Recommendation

Consider adding an array of `minOut` to the parameters that will be used for each loan swap, reducing the gas cost as we won't need to get a quote anymore as well as protect the users from sandwich attacks.
