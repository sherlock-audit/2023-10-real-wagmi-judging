Early Blush Yak

high

# Uniswap Fees Are Sent to Liquidity Manager Without Being Attributed to the LP Owner
## Summary

Creditors who lend Uniswap positions can lose their LP fees that belong to their liquidity position after depositing to the vault.

## Vulnerability Details

Creditors lend Uniswap positions to the vault to get enchanced yield compared to holding the tokens themselves. However, the when `_decreaseLiquidty`  is called, the fees are sent to the liquidity manager contract - `recipient: address(this)`. Although the collected fees are recorded in `amount0` and `amount1`, these values are not passed up the function chain and do not get attributed to the LP depositors, and thus they lose their uniswap LP fees.

```solidity
(amount0, amount1) = underlyingPositionManager.collect(

INonfungiblePositionManager.CollectParams({

tokenId: tokenId,

recipient: address(this),

amount0Max: uint128(amount0),

amount1Max: uint128(amount1)

})

);
```

## Impact

Uniswap LP lenders lose their fees

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L368-L376

## Tool used

Manual Review

## Recommendation

The address of the creditor should be stored, and when `collect` is called, the fees should be sent to `creditor`.
