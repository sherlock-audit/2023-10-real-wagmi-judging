Colossal Tan Hyena

medium

# Revert on Large Approvals & Transfers
## Summary
Some tokens (e.g. UNI, COMP) revert if the value passed to approve or transfer is larger than uint96.


## Vulnerability Detail
Some tokens (e.g. [UNI](https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code), [COMP](https://etherscan.io/token/0xc00e94cb662c3520282e6f5717214004a7f26888#code)) have special case logic in approve that sets allowance to type(uint96).max .
```solidity
function approve(address spender, uint rawAmount) external returns (bool) {
        uint96 amount;
        if (rawAmount == uint(-1)) {
            amount = uint96(-1);
        } else {
            amount = safe96(rawAmount, "Uni::approve: amount exceeds 96 bits");
        }

        allowances[msg.sender][spender] = amount;

        emit Approval(msg.sender, spender, amount);
        return true;
    }
```
if the approval amount is type(uint256).max), which may cause issues with systems that expect the value passed to approve to be reflected in the allowances mapping.

```solidity
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
 The function's multiple attempts will fail
## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L91-L104
## Tool used

Manual Review

## Recommendation
It is advisable to dynamically adjust the approval amount.
