High Chartreuse Hedgehog

medium

# Price changes after borrowing or slippage during borrowing can cause non-emergency repay() to revert
## Summary
Price changes after borrowing or slippage during borrowing can cause non-emergency repay() to revert. Lenders are forced to use emergency liquidation; borrowers have to use a non-obvious workaround to fix the issue since they cannot do emergency repays, and they will have to pay more fees until they figure out how to get their collateral un-stuck.
## Vulnerability Detail
The Uniswap liquidity borrowed is stored in the `LoanInfo` struct to restore the loaner's liquidity when `repay()` is called to close/liquidate a position. When non-emergency `repay()` is called, an error will be thrown if the restored liquidity is less than the liquidity stored in the `LoanInfo`. See below:
```solidity
//BELOW CODE IS FROM THE _increaseLiquidity() FUNCTION, WHICH IS CALLED BY repay() DURING NON EMERGENCY CALLS
        // Check if the restored liquidity is less than the loan liquidity amount
        // If true, revert with InvalidRestoredLiquidity exception
        if (restoredLiquidity < loan.liquidity) {
            ...
            revert InvalidRestoredLiquidity(
```
This protects the loaner, but the problem is that the original liquidity amount is set by the borrower and is equal to the liquidity reduced from the loaner's position, which does not account for slippage or price changes.
```solidity
    /**
     * @dev Decreases the liquidity of a position by removing tokens. CALLED BY _extractLiquidity()
     * @param tokenId The ID of the position token.
     * @param liquidity The amount of liquidity to be removed. THIS VALUE IS SET BY THE BORROWER
     */
    function _decreaseLiquidity(uint256 tokenId, uint128 liquidity) private {
        // Call the decreaseLiquidity function of underlyingPositionManager contract
        // with DecreaseLiquidityParams struct as argument
        (uint256 amount0, uint256 amount1) = underlyingPositionManager.decreaseLiquidity(
            INonfungiblePositionManager.DecreaseLiquidityParams({
                tokenId: tokenId,
                liquidity: liquidity,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp
            })
        );
```
So if the liquidity restored during repayment is smaller due to price changes or borrow slippage, `repay()` will revert. Example:
1. Loaner's position is 10 `saleTokens` to 10 `holdTokens`. The price ratio of the tokens is 1:1.
2. Borrower calls `borrow()` to open a position, taking all the loaner's liquidity. Uniswap liquidity is calculated as $L=\sqrt{x*y}$, so the liquidity here is $\sqrt{100}$.
3. All the `saleTokens` are swapped to `holdTokens`, so the balance of the loan is now 20 `holdTokens`.
4. The price ratio of the pool changes over time to 1 `saleToken` to 2 `holdTokens`. $\sqrt{100}$ liquidity is now approx. equal to a LP position of 7 `saleTokens` and 14 `holdTokens`.
5. The borrower calls `repay()`, and 14 `holdTokens` are swapped for 7 `saleTokens` to prepare for restoring liquidity (liquidity should be provided at the current price/ratio). 6 `holdTokens` are left over.
6. 7 `saleTokens` and only 6 `holdTokens` are restaked, so the liquidity restored is far below $\sqrt{100}$. `InvalidRestoredLiquidity` error is thrown.
## Impact
The borrower's collateral balance is used in the restoring liquidity swap, so the borrower could call `increaseCollateralBalance()` to enable repay() to work. However, there is a check in this function that prevents anyone who's not the borrower from increasing the borrower's collateral balance: `(borrowing.borrowedAmount == 0 || borrowing.borrower != address(msg.sender)).revertError( ErrLib.ErrorCode.INVALID_BORROWING_KEY );`. 

So borrowers will have to call `increaseCollateralBalance()` (borrowers cannot do emergency liquidation) to get repay() to succeed. Loaners will be forced to use emergency liquidation for `repay()` to succeed, since they can't increase the borrower's collateral balance (emergency liquidation doesn't do swapping). The protocol's functionality is not working properly, and users will also lose gas from the reverted calls to `repay()`.

This is particularly an issue for borrowers, because the solution to this issue is not obvious, and there is no functionality in the protocol that makes clear how much the borrower needs to increase their collateral to successfully call `repay()`. Borrowers will have their collateral stuck in the contract until they figure out the workaround, and during that time their loan will be collecting extra fees, so they will have to pay extra fees.
## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L408-L425
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L344-L360
https://uniswapv3book.com/docs/introduction/uniswap-v3/#the-mathematics-of-uniswap-v3
https://docs.uniswap.org/contracts/v2/guides/smart-contract-integration/providing-liquidity
## Tool used
Manual Review
## Recommendation
Add a transferFrom into `repay()` so that borrowers don't need to separately call `increaseCollateralBalance()` for `repay()` to succeed. If their collateral funds are not sufficient to restore liquidity, throw an error notifying them that they need to approve the contract for X amount of `holdToken` and have that amount in their balance.