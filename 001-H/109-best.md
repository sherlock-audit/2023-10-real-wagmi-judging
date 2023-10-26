Big Charcoal Cod

high

# No slippage protection during repayment due to dynamic slippage params and easily influenced `slot0()`
## Summary
The repayment function lacks slippage protection. It relies on slot0() to calculate sqrtLimitPrice, which in turn determines amounts for restoring liquidation. The dynamic calculation of slippage parameters based on these values leaves the function without adequate slippage protection, potentially reducing profit for the repayer.

## Vulnerability Detail
The absence of slippage protection can be attributed to two key reasons. Firstly, the `sqrtPrice` is derived from `slot0()`, **which can be easily manipulated:**
```Solidity
     function _getCurrentSqrtPriceX96(
        bool zeroForA,
        address tokenA,
        address tokenB,
        uint24 fee
    ) private view returns (uint160 sqrtPriceX96) {
        if (!zeroForA) {
            (tokenA, tokenB) = (tokenB, tokenA);
        }
        address poolAddress = computePoolAddress(tokenA, tokenB, fee);
        (sqrtPriceX96, , , , , , ) = IUniswapV3Pool(poolAddress).slot0(); //@audit-issue can be easily manipulated
    }
```
The calculated `sqrtPriceX96` is used to determine the amounts for restoring liquidation and the number of holdTokens to be swapped for saleTokens:
```Solidity
(uint256 holdTokenAmountIn, uint256 amount0, uint256 amount1) = _getHoldTokenAmountIn(
                params.zeroForSaleToken,
                cache.tickLower,
                cache.tickUpper,
                cache.sqrtPriceX96,
                loan.liquidity,
                cache.holdTokenDebt
            );
``` 
After that, the number of `SaleTokemAmountOut` is gained based on the sqrtPrice via QuoterV2.

Then, the slippage params are calculated 
`amountOutMinimum: (saleTokenAmountOut * params.slippageBP1000) /
                                Constants.BPS
                        })`
However, the `saleTokenAmountOut` is a dynamic number calculated on the current state of the blockchain, based on the calculations mentioned above. This will lead to the situation that the swap will always satisfy the `amountOutMinimum`.

As a result, if the repayment of the user is sandwiched (frontrunned), the profit of the repayer is decreased till the repayment satisfies the restored liquidity.

### Proof of concept
A Proof of Concept (PoC) demonstrates the issue with comments. Although the swap does not significantly impact a strongly founded pool, it does result in a loss of a few dollars for the repayer.

```Javascript
       let amountWBTC = ethers.utils.parseUnits("0.05", 8); //token0
        const deadline = (await time.latest()) + 60;
        const minLeverageDesired = 50;
        const maxCollateralWBTC = amountWBTC.div(minLeverageDesired);

        const loans = [
            {
                liquidity: nftpos[3].liquidity,
                tokenId: nftpos[3].tokenId,
            },
        ];

        const swapParams: ApproveSwapAndPay.SwapParamsStruct = {
            swapTarget: constants.AddressZero,
            swapAmountInDataIndex: 0,
            maxGasForCall: 0,
            swapData: swapData,
        };

let  params = {
            internalSwapPoolfee: 500,
            saleToken: WETH_ADDRESS,
            holdToken: WBTC_ADDRESS,
            minHoldTokenOut: amountWBTC,
            maxCollateral: maxCollateralWBTC,
            externalSwap: swapParams,
            loans: loans,
        };

await borrowingManager.connect(bob).borrow(params, deadline);

const borrowingKey = await borrowingManager.userBorrowingKeys(bob.address, 0);
        const swapParamsRep: ApproveSwapAndPay.SwapParamsStruct = {
            swapTarget: constants.AddressZero,
            swapAmountInDataIndex: 0,
            maxGasForCall: 0,
            swapData: swapData,
        };

       
        amountWBTC = ethers.utils.parseUnits("0.06", 8); //token0

let swapping: ISwapRouter.ExactInputSingleParamsStruct = {
            tokenIn: WBTC_ADDRESS,
            tokenOut: WETH_ADDRESS,
            fee: 500,
            recipient: alice.address,
            deadline: deadline,
            amountIn: ethers.utils.parseUnits("100", 8),
            amountOutMinimum: 0,
            sqrtPriceLimitX96: 0
        };
        await router.connect(alice).exactInputSingle(swapping);
        console.log("Swap success");

 let paramsRep: LiquidityBorrowingManager.RepayParamsStruct = {
            isEmergency: false,
            internalSwapPoolfee: 500,
            externalSwap: swapParamsRep,
            borrowingKey: borrowingKey,
            swapSlippageBP1000: 990, //<=slippage simulated
        };
 await borrowingManager.connect(bob).repay(paramsRep, deadline);
        // Without swap
// Balance of hold token after repay:  BigNumber { value: "993951415" }
// Balance of sale token after repay:  BigNumber { value: "99005137946252426108" }
// When swap
// Balance of hold token after repay:  BigNumber { value: "993951415" }
// Balance of sale token after repay:  BigNumber { value: "99000233164653177505" }
```

The following table shows difference of recieved sale token:
| Swap before repay transaction | Token |  Balance of user after Repay |
|---------------------------|---------------------|----------------------|
| No                        | WETH  | 99005137946252426108 |
| Yes                       | WETH  |99000233164653177505 |

The difference in the profit after repayment is 4904781599248603 weis, which is at the current market price of around 8 USD. The profit loss will depend on the liquidity in the pool, which depends on the type of pool and related tokens.

## Impact
The absence of slippage protection results in potential profit loss for the repayer.

## Code Snippet
[Slot0 is used here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L341)
[Dynamic slippage params are created here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L265) - saleTokenAmount is dynamic variable calculated on the state of blockchain.

## Tool used

Manual Review

## Recommendation
To address this issue, avoid relying on slot0 and instead utilize Uniswap TWAP. Additionally, consider manually setting values for amountOutMin for swaps based on data acquired before repayment.
