Dandy Taupe Barracuda

high

# Incorrect calculations of borrowingCollateral leads to DoS for positions in the current tick range due to underflow
## Summary
The `borrowingCollateral` is the amount of collateral a borrower needs to pay for his leverage. It should be calculated as the difference of holdTokenBalance (the amount borrowed + holdTokens received after saleTokens are swapped) and the amount borrowed and checked against user-specified maxCollateral amount which is the maximum the borrower wishes to pay. However, in the current implementation the `borrowingCollateral` calculation is most likely to underflow.
## Vulnerability Detail
This calculation is most likely to underflow
```solidity
uint256 borrowingCollateral = cache.borrowedAmount - cache.holdTokenBalance;
```
The `cache.borrowedAmount` is the calculated amount of holdTokens based on the liquidity of a position. `cache.holdTokenBalance` is the balance of holdTokens queried after liquidity extraction and tokens transferred to the `LiquidityBorrowingManager`. If any amounts of the saleToken are transferred as well, these are swapped to holdTokens and added to `cache.holdTokenBalance`. 

So in case when liquidity of a position is in the current tick range, both tokens would be transferred to the contract and saleToken would be swapped for holdToken and then added to `cache.holdTokenBalance`. This would make `cache.holdTokenBalance > cache.borrowedAmount` since `cache.holdTokenBalance == cache.borrowedAmount + amount of sale token swapped` and would make the tx revert due to underflow.
## Impact
Many positions would be unavailable to borrowers. For non-volatile positions like that which provide liquidity to stablecoin pools the DoS could last for very long period. For volatile positions that provide liquidity in a wide range this could also be for more than 1 year.

## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L492-L503
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L470
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L848-L896
## Tool used

Manual Review

## Recommendation
The borrowedAmount should be subtracted from holdTokenBalance
```solidity
uint256 borrowingCollateral = cache.holdTokenBalance - cache.borrowedAmount;
```