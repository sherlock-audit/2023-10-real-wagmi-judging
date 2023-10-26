Big Eggshell Mink

high

# Drain vault through abuse of `takeOverDebt` function
## Summary

The `takeOverDebt` function incorrectly increases `newBorrowing.dailyRateCollateralBalance`, which allows a malicious attacker to drain specific `holdToken` funds from the vault. 

## Vulnerability Detail

In `takeOverDebt` in LiquidityBorrowingManager.sol, we increase the `newBorrowing.dailyRateCollateralBalance` by:

```solidity
newBorrowing.dailyRateCollateralBalance +=
            (collateralAmt - minPayment) *
            Constants.COLLATERAL_BALANCE_PRECISION;
```

However, this is incorrect. `minPayment` here is the amount that the old position was underwater: `minPayment = (uint256(-collateralBalance) / Constants.COLLATERAL_BALANCE_PRECISION) + 1;`. 

I believe the reason that the devs increased the `newBorrowing.dailyRateCollateralBalance` by `(collateralAmt - minPayment)` is because they believed that the new borrower would be forced to send the amount the old position was underwater to the vault (i.e. in this line):

`_pay(oldBorrowing.holdToken, msg.sender, VAULT_ADDRESS, collateralAmt + feesDebt);`

However, this is actually incorrect:

```solidity
        (
            uint256 feesDebt,
            bytes32 newBorrowingKey,
            BorrowingInfo storage newBorrowing
        ) = _initOrUpdateBorrowing(
                oldBorrowing.saleToken,
                oldBorrowing.holdToken,
                accLoanRatePerSeconds
            );
```

`feesDebt` here is the amount that the NEW position is underwater (i.e. the one just created by the new borrower), which is likely 0 if this is the new borrower's first borrow for that saleToken/holdToken. This is because `_initOrUpdateBorrowing` relies on `msg.sender` to compute the borrowing key, from which it pulls the data used to calculate the `feesDebt`: `borrowingKey = Keys.computeBorrowingKey(msg.sender, saleToken, holdToken);`. 

So, we are increasing `newBorrowing.dailyRateCollateralBalance` by `-minPayment` but not actually sending `-minPayment` tokens over to the vault, which is quite bad. Note that this `newBorrowing.dailyRateCollateralBalance` influences the amount that the user is paid when `repay` is called, so this essentially means that they will get around `-minPayment` extra tokens when `repay` is called (to be extremely precise, perhaps minus some small fee). 

Now let's see how to exploit this. 

First, a malicious user borrows around $200 worth of some hold token (exact sale / hold token don't really matter) and lets it just sit there over time. On mainnet, gas fees are pretty high, so even when this goes underwater we won't really see liquidations (default liquidation bonus is max of 0.69%, which will be very low, and the min liquidation bonus). It's possible that high min liquidation bonuses will be set for most of the set of hold tokens, but the contracts allow `holdTokens` where the min liquidation bonus is not set (in that case it just uses `Constants.MINIMUM_AMOUNT` as the min liquidation bonus which is a really low value). This attack pertains to `holdTokens` where the min liquidation bonus is not set to an extremely high value. Eventually, the user's $200 borrow will go very underwater (let's say **$50** underwater after a few months). The malicious user can then take over the loans with `takeOverDebt`, which will grant them an extra **$50** when they call `repay` because of the vulnerability we described. There's one problem though -- the malicious user who takes over the debt still has to pay the fees, which might land them in the net negative. The trick to bypass this is to just become the creditor -- when the malicious user initially takes out the borrow, they will take out the borrow from their own V3 LP position (so they are the creditor). They will then get back 80% of the fees while 20% will go to the protocol, so their profit margin here is still 80% of the amount the position goes underwater, which can still be quite high past gas fees.

Note that this procedure can be repeated as many times as the malicious user wants until all the hold token is drained from the vault. 

## Impact

We can drain funds from the vault, but only of specific `holdTokens` where min liquidation bonus is not set to an extremely high amount. Also, anyone who calls `takeOverDebt` (even non maliciously) will get more `holdToken` funds than they deserve. 

## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L395-L453

## Tool used

Manual Review

## Recommendation
Just calculate `feesDebt` correctly above for the old borrower, instead of for the new borrower. Or just use `-minPayment` instead of `feesDebt` all together. 