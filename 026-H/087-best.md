Early Blush Yak

high

# Liquidity Position Payoff Does not Match Disassembled Payoff
## Summary

The protocol assumes that a liquidity positon can be restored as long as the amount returned `>=` the amount borrowed. This is not true. Disassembling the uniswap position does not have the same payoff as the original uniswap position. Therefore, even a full repayment of the loan will not be able to restore the loan liquidity.

## Vulnerability Detail

- Let's say ETH-USDC is $1000
- Liquidity provider provides lqiuidity at $800-$1200
- Price moves to $1100
- Now, the liquidity position has suffered some impermanant loss. It is worth less than before.
- Somebody takes a loan out, dissasembling the uniswap position
- The price moves back to $1000

If a loan was never taken, the user would have an impermanant loss of zero. However, since a loan was takn out, the Uniswap position was dissasembled. The impermanant loss became a realised loss. When repay is attempted, the attempt will revert due to the repayment not being able to return the original amount of liquidity

## Impact

- Loss of funds for lender
- Loss of funds for borrower for not being able to pay back their loan

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L223-L321

## Tool used

Manual Review

## Recommendation

There still needs to be price checks for this leverage system, that ensures that there is enough collateral to pay off price slippage. In return, the borrower should be able to get a discount on repayment if the price shifts such that restoring the liquidity position is cheaper.
