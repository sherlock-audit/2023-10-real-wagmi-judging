Rough Pearl Wombat

medium

# Wrong check in `repay()` makes borrower loose its `dailyCollateral` if closing position quickly after opening it.
## Summary

In the [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) function of the LiquidityBorrowingManager contract, there is a check that verifies that the borrower's `dailyCollateralBalance` left after fees are applied is above 0 and that we paid more than the minimum amount of fees.

If so then it adds the `dailyCollateralBalance` to the tokens that we will receive when closing our position.

But in the case of the fees paid not being above the minimum it will not add it to the tokens we should receive and will count it as fees owned to the lenders. Making the borrower paying more fees than he should for no apparent reason as the amount will be way more than the minimum.

## Vulnerability Detail

In the [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) there is a check [line 567](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L567) that makes sure the borrower paid more than `MINIMUM_AMOUNT` but this check shouldn't be the condition to add `dailyCollateralBalance` to `liquidationBonus`.

If a borrower borrows and then tries to repay his position before the minimum amount of fees is met, he will loose the whole `dailyCollateralBalance` that will be considered as fees although it could be way more than `MINIMUM_AMOUNT`.

Take this example:

- User borrows 100 usdc (hold token) to short USDC-ETH LP on Polygon network.
- After 5 minutes he decides that afterall he doesn't want to short this position anymore and so call `repay()`.

The USDC token decimals is 6 so 100 usdc -> 100 * 1e6 <=> 100,000,000.
The `MINIMUM_AMOUNT` is currently set to 1000 in the [Constant library](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/libraries/Constants.sol).
Let's say the current USDC rate is set to `DEFAULT_DAILY_RATE` which is 10 (0.1%).

This means that the current daily fees is 0.1 USDC/day.
With decimals and in second that's 100,000 / (3600 * 24) ~= 1.157 wei per second.
So if only 5min went by that means the borrower's debt is 1.157 * (5 * 60) = 347.2.

- When repaying the check on fee to pay being more than `MINIMUM_AMOUNT` return false and so `dailyCollateral` is kept.
- User looses 0.1 usdc instead of 347 wei of USDC or even `MINIMUM_AMOUNT` wei of USDC.

This situation can be way worse and apply to any token and amount if the borrowing and repayment happen in the same block or just few seconds interval.

Consider this POC that can be copied and pasted in the test files (replace all tests and just keep the setup & NFT creation):

```js
it("Borrow then repay instantly loosing dailyCollateral", async () => {
        const amountWBTC = ethers.utils.parseUnits("0.05", 8); //token0
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

        const borrowingKey = await borrowingManager.userBorrowingKeys(bob.address, 0);

        let repayParams = {
            isEmergency: false,
            internalSwapPoolfee: 500,
            externalSwap: swapParams,
            borrowingKey: borrowingKey,
            swapSlippageBP1000: 990, //1%
        };

        const WBTC: IERC20 = await ethers.getContractAt("IERC20", WBTC_ADDRESS);
        const prevBalance = await WBTC.balanceOf(bob.address);

        //query amount of collateral available
        const borrowingsInfo = await borrowingManager.borrowingsInfo(borrowingKey);
        const dailyCollateral = borrowingsInfo.dailyRateCollateralBalance.div(COLLATERAL_BALANCE_PRECISION);
        const liquidationBonus = borrowingsInfo.liquidationBonus;
        //should be more than 0
        expect(dailyCollateral).to.be.gt(0);
        expect(liquidationBonus).to.be.gt(0);

        //BOB repay his loan but loose his dailyCollateral even tho it hasn't been a day
        await borrowingManager.connect(bob).repay(repayParams, deadline);

        const newBalance = await WBTC.balanceOf(bob.address);

        //We only got liquidation bonus back and not the dailyCollateral
        expect(newBalance).to.be.equal(prevBalance.add(liquidationBonus));
    });
```

## Impact

Medium. When borrowing and repaying quickly after, the borrower loose his collateral.

## Code Snippet

## Tool used

Manual Review

## Recommendation

If what we want to achieve here is making sure that a minimum amount of fee is paid then consider doing this in a different `if`.

```solidity
if ((currentFees + borrowing.feesOwed) / Constants.COLLATERAL_BALANCE_PRECISION < Constants.MINIMUM_AMOUNT) {
  uint256 missingFees = Constants.MINIMUM_AMOUNT - (currentFees + borrowing.feesOwed) / Constants.COLLATERAL_BALANCE_PRECISION;
  collateralBalance -= missingFees;
  currentFees += missingsFees;
}
if (collateralBalance > 0) {
  liquidationBonus += uint256(collateralBalance) / Constants.COLLATERAL_BALANCE_PRECISION;
} else {
  currentFees = borrowing.dailyRateCollateralBalance;
}
```
