Stale Raisin Whale

medium

# _pay() will always revert in some functions for not approving VAULT_ADDRESS address to spend the tokens of the borrower.
## Summary
**`_pay()`** is employed to transfer tokens from the borrower to the **VAULT_ADDRESS**. To achieve this, a **`transferFrom()`** operation is used, which transfer the tokens from the borrower's address (EOA) indicated by **msg.sender()** to the **VAULT_ADDRESS**,  but the **VAULT_ADDRESS** is never approved to spend the tokens of the borrower.
## Vulnerability Detail
In several functions **`_pay()`**  is called. These functions involve two addresses: **msg.sender**, representing the borrower (EOA) and **VAULT_ADDRESS**, which is where the tokens are intended to be transferred
```Solidity
       _pay(               
            params.holdToken,
            msg.sender,    
            VAULT_ADDRESS,
            borrowingCollateral + liquidationBonus + cache.dailyRateCollateral + feesDebt
        );
```
When the payer is not the contract, the **`safeTransferFrom()`** function is employed to do the token transfers. However, the crucial step in this process is approving **VAULT_ADDRESS** to enable to expend the tokens belonging to the payer (borrower). This approval is managed by **`_maxApproveIfNecessary()`**. 

The issue arises because the **`_maxApproveIfNecessary()`** is never invoked. This omission causes all transactions to revert.
```Solidity
function _pay(address token, address payer, address recipient, uint256 value) public {
        if (value > 0) {
            if (payer == address(this)) {   
                IERC20(token).safeTransfer(recipient, value);   
            } else {
                IERC20(token).safeTransferFrom(payer, recipient, value);    
            }
        }
    }
```
```Solidity
function _maxApproveIfNecessary(address token, address spender, uint256 amount) internal {
        if (IERC20(token).allowance(address(this), spender) < amount) {
            if (!_tryApprove(token, spender, type(uint256).max)) {
                if (!_tryApprove(token, spender, type(uint256).max - 1)) {
                    require(_tryApprove(token, spender, 0));
                    if (!_tryApprove(token, spender, type(uint256).max)) {
                        if (!_tryApprove(token, spender, type(uint256).max - 1)) {
                            true.revertError(ErrLib.ErrorCode.ERC20_APPROVE_DID_NOT_SUCCEED);
                        }
                    }
                }
            }
        }
    }
```
 
## Impact
In **`increaseCollateralBalance()`**, **`takeOverDebt()`** and **`borrow()`** will always revert.
## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L381
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L451
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L498-L503
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L91-L104
## Tool used

Manual Review

## Recommendation
Before call **`_pay()`** approve the **VAULT_ADDRESS** to spend the tokens of the borrower using **`_maxApproveIfNecessary()`**.