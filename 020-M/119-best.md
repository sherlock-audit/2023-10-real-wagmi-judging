Rough Pearl Wombat

medium

# Wrong `accLoanRatePerSeconds` in `repay()` can lead to underflow
## Summary

When a Lender call the [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) function of the LiquidityBorrowingManager contract to do an emergency liquidity restoration using `isEmergency = true`, the `borrowingStorage.accLoanRatePerSeconds` is updated if the borrowing position hasn't been fully closed.

The computation is made in sort that the missing collateral can be computed again later so we don't loose the missing fees in case someone take over or the borrower decide to reimburse the fees.

But this computation is wrong and can lead to underflow.

## Vulnerability Detail

Because the [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) function resets the `dailyRateCollateralBalance` to 0 when the lender call didn't fully close the position. We want to be able  to compute the missing collateral again.

To do so we substract the percentage of collateral not paid to the `accLoanRatePerSeconds` so on the next call we will be adding extra second of fees that will allow the contract to compute the missing collateral.

The problem lies in the fact that we compute a percentage using the borrowed amount left instead of the initial borrow amount causing the percentage to be higher. In practice this do allows the contract to recompute the missing collateral.

But in the case of the missing `collateralBalance` or `removedAmt` being very high (ex: multiple days not paid or the loan removed was most of the position's liquidity) we might end up with a percentage higher than the `accLoanRatePerSeconds` which will cause an underflow.

In case of an underflow the call will revert and the lender will not be able to get his tokens back.

Consider this POC that can be copied and pasted in the test files (replace all tests and just keep the setup & NFT creation):

```js
it("Updated accRate is incorrect", async () => {
        const amountWBTC = ethers.utils.parseUnits("0.05", 8); //token0
        let deadline = (await time.latest()) + 60;
        const minLeverageDesired = 50;
        const maxCollateralWBTC = amountWBTC.div(minLeverageDesired);

        const loans = [
            {
                liquidity: nftpos[3].liquidity,
                tokenId: nftpos[3].tokenId,
            },
            {
                liquidity: nftpos[5].liquidity,
                tokenId: nftpos[5].tokenId,
            },
        ];

        const swapParams: ApproveSwapAndPay.SwapParamsStruct = {
            swapTarget: constants.AddressZero,
            swapAmountInDataIndex: 0,
            maxGasForCall: 0,
            swapData: swapData,
        };

        const borrowParams = {
            internalSwapPoolfee: 500,
            saleToken: WETH_ADDRESS,
            holdToken: WBTC_ADDRESS,
            minHoldTokenOut: amountWBTC,
            maxCollateral: maxCollateralWBTC,
            externalSwap: swapParams,
            loans: loans,
        };

        //borrow tokens
        await borrowingManager.connect(bob).borrow(borrowParams, deadline);

        await time.increase(3600 * 72); //72h so 2 days of missing collateral
        deadline = (await time.latest()) + 60;

        const borrowingKey = await borrowingManager.userBorrowingKeys(bob.address, 0);

        let repayParams = {
            isEmergency: true,
            internalSwapPoolfee: 0,
            externalSwap: swapParams,
            borrowingKey: borrowingKey,
            swapSlippageBP1000: 0,
        };

        const oldBorrowingInfo = await borrowingManager.borrowingsInfo(borrowingKey);
        const dailyRateCollateral = await borrowingManager.checkDailyRateCollateral(borrowingKey);

        //Alice emergency repay but it reverts with 2 days of collateral missing
        await expect(borrowingManager.connect(alice).repay(repayParams, deadline)).to.be.revertedWithPanic();
    });
```

## Impact

Medium. Lender might not be able to use `isEmergency` on `repay()` and will have to do a normal liquidation if he want his liquidity back.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L613

## Tool used

Manual Review

## Recommendation

Consider that when a lender do an emergency liquidity restoration they give up on their collateral missing and so use the initial amount in the computation instead of borrowed amount left.

```solidity
borrowingStorage.accLoanRatePerSeconds =
                    holdTokenRateInfo.accLoanRatePerSeconds -
                    FullMath.mulDiv(
                        uint256(-collateralBalance),
                        Constants.BP,
                        borrowing.borrowedAmount + removedAmt //old amount
                    );
```
