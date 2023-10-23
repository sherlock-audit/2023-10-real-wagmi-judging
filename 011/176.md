Blunt Pearl Haddock

high

# `COLLATERAL_BALANCE_PRECISION` is used for each calculation of each token type without actually checking how many decimal points the token has
## Summary

`COLLATERAL_BALANCE_PRECISION` is used for each calculation of each token type without actually checking how many decimal points the token has.

## Vulnerability Detail

Wagmi used `COLLATERAL_BALANCE_PRECISION` for collateral scaling precision.

```solidity
uint256 public constant COLLATERAL_BALANCE_PRECISION = 1e18;
```

As you can see from the code the scaling is always `1e18` i.e. for 18 decimal tokens.

But `COLLATERAL_BALANCE_PRECISION` is used for all possible calculations in the protocol without checking how many decimals the token has. Some tokens, for example, have 6 decimals. This leads to totally wrong calculations everywhere in the protocol.
I'll give just a few examples, but the constant is used in many more places.

For example, `collectProtocol()` collects protocol fees for multiple tokens but the collateral scaling precision is always  `1e18`:
```solidity
    function collectProtocol(address recipient, address[] calldata tokens) external onlyOwner {  
        uint256[] memory amounts = new uint256[](tokens.length);
        for (uint256 i; i < tokens.length; ) {
            address token = tokens[i];
            uint256 amount = platformsFeesInfo[token] / Constants.COLLATERAL_BALANCE_PRECISION; 
            if (amount > 0) {
                platformsFeesInfo[token] = 0;
                amounts[i] = amount;
                Vault(VAULT_ADDRESS).transferToken(token, recipient, amount);  
            }
            unchecked {
                ++i;
            }
        }

        emit CollectProtocol(recipient, tokens, amounts);
    }
```

`checkDailyRateCollateral()` check the daily rate collateral for a specific borrowing. Collateral scaling precision again is `1e18`.
```solidity
    function checkDailyRateCollateral(
        bytes32 borrowingKey
    ) external view returns (int256 balance, uint256 estimatedLifeTime) {
        (, balance, estimatedLifeTime) = _getDebtInfo(borrowingKey);
        balance /= int256(Constants.COLLATERAL_BALANCE_PRECISION);
    }
```

And everywhere else in the protocol, instead of using hardcoded `1e18`, the token decimal should be taken dynamically.
## Impact

Incorrect calculations will lead to large financial losses and also make the protocol susceptible to attacks.

## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/libraries/Constants.sol#L15

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L188
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L237
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L319
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L351
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L359
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L380
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L422
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L449
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L490
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L567
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L572
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L598
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L939
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L1017
## Tool used

Manual Review

## Recommendation

Instead of using a fixed `COLLATERAL_BALANCE_PRECISION`, calculate precision dynamically based on the token's decimals.