Festive Daffodil Grasshopper

medium

# _restoreLiquidity may often DOS due to swap slippage
## Summary

_restoreLiquidity will swap holdToken for saleToken, and then add the corresponding LP for repayment.
The total amount of holdToken is `borrowing.borrowedAmount + liquidationBonus`, but only the `borrowedAmount - holdTokenAmount` of each loan is used during the swap, that is, the total swap amount of holdToken is `borrowedAmount`, and liquidationBonus does not participate in the swap, which includes `real liquidationBonus + extra collateralBalance`.
Due to the slippage problem of swap, the amount of saleTokens may be lower than the amount required by LP, which will DOS the liquidation process.

## Vulnerability Detail

```solidity
    function _getHoldTokenAmountIn(
        bool zeroForSaleToken,
        int24 tickLower,
        int24 tickUpper,
        uint160 sqrtPriceX96,
        uint128 liquidity,
        uint256 holdTokenDebt
    ) private pure returns (uint256 holdTokenAmountIn, uint256 amount0, uint256 amount1) {
        // Call getAmountsForLiquidity function from LiquidityAmounts library
        // to get the amounts of token0 and token1 for a given liquidity position
        (amount0, amount1) = LiquidityAmounts.getAmountsForLiquidity(
            sqrtPriceX96,
            TickMath.getSqrtRatioAtTick(tickLower),
            TickMath.getSqrtRatioAtTick(tickUpper),
            liquidity
        );
        // Calculate the holdTokenAmountIn based on the zeroForSaleToken flag
        if (zeroForSaleToken) {
            // If zeroForSaleToken is true, check if amount0 is zero
            // If true, holdTokenAmountIn will be zero. Otherwise, it will be holdTokenDebt - amount1
            holdTokenAmountIn = amount0 == 0 ? 0 : holdTokenDebt - amount1;
        } else {
            // If zeroForSaleToken is false, check if amount1 is zero
            // If true, holdTokenAmountIn will be zero. Otherwise, it will be holdTokenDebt - amount0
            holdTokenAmountIn = amount1 == 0 ? 0 : holdTokenDebt - amount0;
        }
    }
```

holdTokenDebt and borrowAmount are equal, _getHoldTokenAmountIn will calculate the remaining holdTokenAmount for swap, regardless of liquidationBonus.
If the swap result cannot meet the LP quantity due to slippage and market environment, repay / liquidation will fail.

## Impact

_restoreLiquidity may often DOS due to swap slippage and market environment.

## Code Snippet

- https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L458-L466

## Tool used

Manual Review

## Recommendation

Due to the additional funds of liquidationBonus, holdToken is sufficient, should use exactOutputSingle to ensure the amount of saleToken, instead of using exactInputSingle and limited holdToken for swap
