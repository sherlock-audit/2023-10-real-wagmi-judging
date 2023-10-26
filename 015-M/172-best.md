Blunt Pearl Haddock

medium

# Some tokens must approve by zero first
## Summary

Protocol specifically mentions :

> Q: Do you expect to use any of the following tokens with non-standard behavior with the smart contracts?
> - Whatever uniswap v3 supports for their pools can interact with our contracts

Some tokens will revert when updating the allowance. They must first be approved by zero and then the actual allowance must be approved.

## Vulnerability Detail

`_tryApprove()` function attempts to approve a specific amount of tokens for a spender:

```solidity
    function _tryApprove(address token, address spender, uint256 amount) private returns (bool) {
        (bool success, bytes memory data) = token.call(
            abi.encodeWithSelector(IERC20.approve.selector, spender, amount)   
        );
        return success && (data.length == 0 || abi.decode(data, (bool)));
    }
```

Some ERC20 tokens (like USDT) do not work when changing the allowance from an existing non-zero allowance value. For example, Tether (USDT)'s `approve()` function will revert if the current approval is not zero, to protect against front-running changes of approvals.

## Impact

The protocol will be impossible to use with certain tokens.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L75-L80
## Tool used

Manual Review

## Recommendation

For some tokens, you need to set the allowance to zero before increasing the allowance.