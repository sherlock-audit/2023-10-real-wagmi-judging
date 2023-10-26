Ancient Malachite Jay

high

# Adversary can overwrite function selector in _patchAmountAndCall due to inline assembly lack of overflow protection
## Summary

When using inline assembly, the standard [overflow/underflow protections do not apply](https://faizannehal.medium.com/how-solidity-0-8-protect-against-integer-underflow-overflow-and-how-they-can-still-happen-7be22c4ab92f). This allows an adversary to specify a swapAmountInDataIndex which after multiplication and addition allows them to overwrite the function selector. Using a created token in a UniV3 LP pair they can manufacture any value for swapAmountInDataValue.

## Vulnerability Detail

```The use of YUL or inline assembly in a solidity smart contract also makes integer overflow/ underflow possible even if the compiler version of solidity is 0.8. In YUL programming language, integer underflow & overflow is possible in the same way as Solidity and it does not check automatically for it as YUL is a low-level language that is mostly used for making the code more optimized, which does this by omitting many opcodes. Because of its low-level nature, YUL does not perform many security checks therefore it is recommended to use as little of it as possible in your smart contracts.``` 

[Source](https://faizannehal.medium.com/how-solidity-0-8-protect-against-integer-underflow-overflow-and-how-they-can-still-happen-7be22c4ab92f)

Inline assembly lacks overflow/underflow protections, which opens the possibility of this exploit.

[ExternalCall.sol#L27-L38](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/libraries/ExternalCall.sol#L27-L38)

            if gt(swapAmountInDataValue, 0) {
                mstore(add(add(ptr, 0x24), mul(swapAmountInDataIndex, 0x20)), swapAmountInDataValue)
            }
            success := call(
                maxGas,
                target,
                0, //value
                ptr, //Inputs are stored at location ptr
                data.length,
                0,
                0
            )

In the code above we see that `swapAmountInDataValue` is stored at `ptr + 36 (0x24) + swapAmountInDataIndex * 32 (0x20)`. The addition of 36 (0x24) in this scenario should prevent the function selector from being overwritten because of the extra 4 bytes (using 36 instead of 32). This is not the case though because `mul(swapAmountInDataIndex, 0x20)` can overflow since it is a uint256. This allows the attacker to target any part of the memory they choose by selectively overflowing to make it write to the desired position.

As shown above, overwriting the function selector is possible although most of the time this value would be a complete nonsense since swapAmountInDataValue is calculated elsewhere and isn't user supplied. This also has a work around. By creating their own token and adding it as LP to a UniV3 pool, swapAmountInDataValue can be carefully manipulated to any value. This allows the attacker to selectively overwrite the function selector with any value they chose. This bypasses function selectors restrictions and opens calls to dangerous functions. 

## Impact

Attacker can bypass function restrictions and call dangerous/unintended functions

## Code Snippet

[ExternalCall.sol#L14-L47](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/libraries/ExternalCall.sol#L14-L47)

## Tool used

Manual Review

## Recommendation

Limit `swapAmountInDataIndex` to a reasonable value such as uint128.max, preventing any overflow.