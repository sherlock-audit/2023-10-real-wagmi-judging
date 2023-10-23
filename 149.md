Raspy Beige Aphid

high

# "zeroForSaleToken" variable is incorrect calculated, fixing the sale direction of every pair
## Summary
`zeroForSaleToken` is a boolean value used in UniswapV3 that provides the direction of the swap. If true, it means you are selling token0 for token1. If false, it means you are selling token1 for token0. However, in `LiquidityBorrowingManager.sol`, the `zeroForSaleToken` direction is decided based on which token address is a higher value (`bool zeroForSaleToken = params.saleToken < params.holdToken;`), which makes the direction point to an address with higher hex value, so some pair swaps will always have the same direction, making that you can just borrow token0 for token1, but never token1 for token0 in these pairs.

## Vulnerability Detail
1. In `LiquidityBorrowingManager.sol`, you input the struct `BorrowParams` in the method `borrow`.  After that, the subcall `_precalculateBorrowing` is called. The subcall snippet:
```solidity
    function _precalculateBorrowing(
        BorrowParams calldata params
    ) private returns (BorrowCache memory cache) {
        {
            bool zeroForSaleToken = params.saleToken < params.holdToken;
// more code below
```
2. The line `bool zeroForSaleToken = params.saleToken < params.holdToken;` compares params.saleToken to params.holdToken from a `BorrowParams` struct. Let's see what are they in the `BorrowParams`:
```solidity
    struct BorrowParams {
        /// @notice The pool fee level for the internal swap
        uint24 internalSwapPoolfee;
        /// @notice The address of the token that will be sold to obtain the loan currency
        address saleToken;
        /// @notice The address of the token that will be held
        address holdToken;
        /// @notice The minimum amount of holdToken that must be obtained
        uint256 minHoldTokenOut;
        /// @notice The maximum amount of collateral that can be provided for the loan
        uint256 maxCollateral;
        /// @notice The SwapParams struct representing the external swap parameters
        SwapParams externalSwap;
        /// @notice An array of LoanInfo structs representing multiple loans
        LoanInfo[] loans;
    }
```
3. It's visible that `saleToken` and `holdToken` are both addresses. So, the direction of the swap is calculated by which address hexadecimal is higher, which it's not intended by the protocol or UniswapV3.

4. After in `_precalculateBorrowing`, the inherited method `_extractLiquidity` from `LiquidityManager.sol` is called and the `zeroForSaleToken` is one of the inputs. The snippet:
```solidity
    function _extractLiquidity(
        bool zeroForSaleToken,
        address token0,
        address token1,
        LoanInfo[] memory loans
    ) internal returns (uint256 borrowedAmount) {
        if (!zeroForSaleToken) {
            (token0, token1) = (token1, token0);
        }

        for (uint256 i; i < loans.length; ) {
            uint256 tokenId = loans[i].tokenId;
            uint128 liquidity = loans[i].liquidity;
            // Extract position-related details
            {
                int24 tickLower;
                int24 tickUpper;
                uint128 posLiquidity;
                {
                    address operator;
                    address posToken0;
                    address posToken1;

                    (
                        ,
                        operator,
                        posToken0,
                        posToken1,
                        ,
                        tickLower,
                        tickUpper,
                        posLiquidity,
                        ,
                        ,
                        ,

                    ) = underlyingPositionManager.positions(tokenId);
                    // Check operator approval
                    if (operator != address(this)) {
                        revert NotApproved(tokenId);
                    }
                    // Check token validity
                    if (posToken0 != token0 || posToken1 != token1) {
                        revert InvalidTokens(tokenId);
                    }
                }
                // Check borrowed liquidity validity
                if (!(liquidity > 0 && liquidity <= posLiquidity)) {
                    revert InvalidBorrowedLiquidity(tokenId);
                }
                // Calculate borrowed amount
                borrowedAmount += _getSingleSideRoundUpBorrowedAmount(
                    zeroForSaleToken,
                    tickLower,
                    tickUpper,
                    liquidity
                );
            }
            // Decrease liquidity and move to the next loan
            _decreaseLiquidity(tokenId, liquidity);

            unchecked {
                ++i;
            }
        }
    }
```
It's visible that the protocol uses `zeroForSaleToken` as a way to decide which is the token for sale in the first lines. However, as this boolean is not calculated correctly, some pairs have a fixed direction (tokenX -> tokenY) based on which address has a higher value, so not respecting this buggy direction can make the transaction revert (swapping the holdToken for the saleToken) or executing wrong swaps.

## Impact
Based on which token address of the pair is higher, some pairs will have a fixed direction of which token should be for sale and which token is not. As this is not intended, some pairs will be only possible to borrow token0 for token1 and never token1 for token0, which is a high severity denial of service that makes transactions reverts and the protocol to not work as intended.

## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L837
```solidity
    function _precalculateBorrowing(
        BorrowParams calldata params
    ) private returns (BorrowCache memory cache) {
        {
            bool zeroForSaleToken = params.saleToken < params.holdToken;
```

## Tool used
Manual Review

## Recommendation
Allow user to choose the direction or correctly calculate it comparing the amount of assets in eth, not the addresses.
