Fun Magenta Aardvark

high

# The `FullMath` library used in `LiquidityBorrowingManager.sol` and `DailyRateAndCollateral.sol` is unable to handle intermediate overflows due to overflow
## Summary
The FullMath.sol library used in `LiquidityBorrowingManager.sol` and `DailyRateAndCollateral.sol` contracts doesn't correctly handle the case when an intermediate value overflows 256 bits. This happens because an overflow is desired in this case but it's never reached.

## Vulnerability Detail
The FullMath.sol library was taken from Uniswap. However, the original solidity version that was used was in this FullMath.sol library was `< 0.8.0`, Which means that the execution didn't revert when an overflow was reached. This effectively means that when a phantom overflow (a multiplication and division where an intermediate value overflows 256 bits) occurs the execution will revert and the correct result won't be returned. 

`mulDivRoundingUp()` from the `FullMath.sol` has been used in below in scope contracts,

1) In `LiquidityBorrowingManager.sol` having functions `calculateCollateralAmtForLifetime()`, `_precalculateBorrowing()` and `_getDebtInfo()`

2) In `DailyRateAndCollateral.sol` having function `_calculateCollateralBalance()`

The values returned from these functions wont be correct and the execution of these functions will be reverted when phantom overflow occurs.

## Impact
The correct result isn't returned in this case and the execution gets reverted when a phantom overflows occurs.

## Code Snippet
Code reference: https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/vendor0.8/uniswap/FullMath.sol#L119-L129

The `FullMath.sol` having `mulDivRoundingUp()` function used `LiquidityBorrowingManager.sol` and `DailyRateAndCollateral.sol`  doesn't use an unchecked block and it is shown as below(_removed comments to shorten the code_),

```Solidity
File: wagmi-leverage/contracts/vendor0.8/uniswap/FullMath.sol

// SPDX-License-Identifier: MIT

pragma solidity ^0.8.4;

library FullMath {

    function mulDiv(
        uint256 a,
        uint256 b,
        uint256 denominator
    ) internal pure returns (uint256 result) {
        unchecked {                                 @audit // used unchecked block here but not used in mulDivRoundingUp()
            uint256 prod0; // Least significant 256 bits of the product
            uint256 prod1; // Most significant 256 bits of the product
            assembly {
                let mm := mulmod(a, b, not(0))
                prod0 := mul(a, b)
                prod1 := sub(sub(mm, prod0), lt(mm, prod0))
            }


           // some code


            result = prod0 * inv;
            return result;
        }
    }


    function mulDivRoundingUp(
        uint256 a,
        uint256 b,
        uint256 denominator
    ) internal pure returns (uint256 result) {
        result = mulDiv(a, b, denominator);                @audit // does not use unchecked block similar to mulDiv()
        if (mulmod(a, b, denominator) > 0) {
            require(result < type(uint256).max);
            result++;
        }
    }
}
```

## Tool used
Manual Review

## Recommendation
It is advised to put the entire function bodies of `mulDivRoundingUp()` similar to `mulDiv()` in an unchecked block. A modified version of the original Fullmath library that uses unchecked blocks to handle the overflow, can be found in the 0.8 branch of the [Uniswap v3-core repo](https://github.com/Uniswap/v3-core/blob/0.8/contracts/libraries/FullMath.sol).

Per `Uniswap-v3-core`, Do below changes in `FullMath.sol`

```diff

    function mulDivRoundingUp(
        uint256 a,
        uint256 b,
        uint256 denominator
    ) internal pure returns (uint256 result) {
+       unchecked {
            result = mulDiv(a, b, denominator);
            if (mulmod(a, b, denominator) > 0) {
                require(result < type(uint256).max);
                result++;
           }
+       }
    }
```