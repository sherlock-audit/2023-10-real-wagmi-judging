Rough Pearl Wombat

medium

# Lender is stuck as long a borrower is willing to pay
## Summary

Wagmi allows lenders to lend their Uniswap v3 LPs to borrowers against a daily fee. If a borrower stops paying the daily fee then the positon can be liquidated or lenders can do an emergency liquidity restoration and get their liquidity back.

The current architecture doesn't have any loan duration limit. This means that as long as a borrower is willing to pay a fee he can keep borrowing the tokens from the lender.

The lender does not receive the fees until the position has been closed or is in default of collateral that allow him to do an emergency liquidity restoration by calling [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) with `isEmergency = true`.

## Vulnerability Detail

This is in itselft not a vulnerability but an architecture issue as there is no way for a lender to know how long his tokens are gonna be borrowed for.

While in most of the case it will probably be a few days to a few months, this could potentially be way more especially if the borrower decides to go rogue and is not looking to make profit but just DOS the lender.

Take this example:

If a borrower borrows 500 USDC at 0.1% daily rate (default value). He can hold the position for a year, costing him 36.5% of the position in collateral to pay.

If he decides to go rogue and use all his money to deny lender withdrawal as long as possible he can just keep paying 36.5% of collateral every year.

Other lending market usually allow one of these features:

- Fixed duration for the loan.
- Allow new lenders to come and add liquidity so previous lenders can withdraw (ex: AAVE, COMPOUND).
- High increase in rate after a certain time, increasing so much that it's unlikely a borrower can sustain such position.

## Impact

Info to medium. While this look more like an architecture choice than a vulnerability, Sherlock rules consider a DOS if the period of locked funds goes above 1 year. In our case it could be multiple years and no way to predict the duration.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532

## Tool used

Manual Review

## Recommendation

Consider adding a maximum duration for a loan before it can go into liquidation or increase the daily rate significantly after a certain duration. 
