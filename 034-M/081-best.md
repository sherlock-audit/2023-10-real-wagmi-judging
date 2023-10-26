Dandy Taupe Barracuda

high

# Borrowers that borrow from certain positions can get leverage without paying collateral due to inaccurate borrowingCollateral calculations
## Summary
During borrowing, a borrower has to pay some collateral in holdToken plus deposit for holding a position for 24 hours. However, if the uniswap position is above the current tick for token0 (or below it for token1) the borrower would pay 0 `borrowingCollateral`.
## Vulnerability Detail
The vulnerable line in question:
```solidity
uint256 borrowingCollateral = cache.borrowedAmount - cache.holdTokenBalance;
```
`cache.borrowedAmount` is the amount of token0 or token1 calculated in `_getSingleSideRoundUpBorrowedAmount()` function based on the provided liquidity and `zeroForSaleTokenFlag`. After that calculation, the liquidity is pulled from the uniswap position in call to `_decreasePostion()` and is sent to the `LiquidityBorrowingManager` contract. Then, the `borrow()` function continues to check how many borrowTokens and saleTokens were transferred from the position by checking contract's balances
```solidity
// Get the balance of the sale token and hold token in the pair
        (saleTokenBalance, cache.holdTokenBalance) = _getPairBalance(
            params.saleToken,
            params.holdToken
        );
```
And if `saleTokenBalance > 0`, the saleToken is swapped through the external swap function or through uniswap. The amount swapped is then added to `cache.holdTokenBalance`.

Now, suppose there are no saleTokens that were pulled from LP (meaning that the position is above the current tick if the borrowToken is token0). It means that `cache.holdTokenBalance == cache.borrowedAmount` and the `borrowingCollateral == 0`. Since it is the amount the borrower should pay if he decides to borrow, he can receive almost free leverage (he only needs to pay the liquidationBonus and the deposit for holding a position for 24 hours).
```solidity
        _pay(
            params.holdToken,
            msg.sender,
            VAULT_ADDRESS,
            borrowingCollateral + liquidationBonus + cache.dailyRateCollateral + feesDebt //@audit
        );
```
## Impact
Getting an almost collateral-free leverage is in contradiction with what expected by the team and breaks the whole point of leverage trading.

## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L492
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L848-L853
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L201-L209
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L116
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L349
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L869-L896
## Tool used

Manual Review

## Recommendation
If the `borrowingCollateral` turns out to be 0, ensure that some minimum amount of collateral is paid by the borrower. It can be a percent of the `borrowedAmount`.