Obedient Misty Tiger

medium

# ApproveSwapAndPay.sol is vulnerable to address collission
## Summary
In ApproveSwapAndPay.sol, the uniswapV3SwapCallback() never checks with the factory that the pool exists or if any of the inputs are valid in any way.
## Vulnerability Detail
In the uniswapV3SwapCallback(), only a check is performed to verify if msg.sender has been computed via computePoolAddress(), but never check with the factory that the pool exists or any of the inputs are valid in any way. 
```solidity
(computePoolAddress(tokenIn, tokenOut, fee) != msg.sender).revertError(
            ErrLib.ErrorCode.INVALID_CALLER
        );
```
According to the [UniswapV3 Doc](https://docs.uniswap.org/contracts/v3/reference/core/interfaces/callback/IUniswapV3SwapCallback), when using uniswapV3SwapCallback, the caller of this method must be verified to be a UniswapV3Pool deployed by the canonical UniswapV3Factory.
In the 2023-07-kyber-swap contest, there is a similar valid finding that provides a detailed explanation and verification for this issue. You can review it here:
https://github.com/sherlock-audit/2023-07-kyber-swap-judging/issues/90
## Impact
The pool address check in the the callback function isn't strict enough and can suffer issues with collision.
Address collision can cause all allowances to be drained.
Although this would require a large amount of compute it is already possible to break with current computing. Given the enormity of the value potentially at stake it would be a lucrative attack to anyone who could fund it. In less than a decade this would likely be a fairly easily attained amount of compute, nearly guaranteeing this attack.
## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L242
## Tool used

Manual Review

## Recommendation
Verify with the factory that msg.sender is a valid pool
You can refer to the use of verifyCallback in Uniswap V3:
https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol#L65
https://github.com/Uniswap/v3-periphery/blob/main/contracts/libraries/CallbackValidation.sol#L21