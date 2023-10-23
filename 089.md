Early Blush Yak

high

# No Incentive to Liquidate Small Positions
## Summary

`min_borrowed_amount` allows very small positions to be opened. The gas cost for liquidating these positions is lower than the profit incentive for liquidators.

## Vulnerability Details
There is a check that enforces a minimum borrowed amount. However it does not scale according to token decimals or value. The `min_borrowed_amount`, which is not decimal scaled, is less than $1 for many tokens.

```solidity
uint256 public constant MINIMUM_BORROWED_AMOUNT = 100000;
```

The reward for liquidating a position is proportional to its size. With a very small sizeThe gas costs for liquidating a position can exceed the benefit gained from a position take over or liquidation. Therefore this may force collectors of positions or cause bad debt which is passed on to other users.


## Impact

No incentive to liquidate very small loans which leads to non-repayment and potential bad debt.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/libraries/Constants.sol#L17C5-L17C62

## Tool used

Manual Review

## Recommendation

Add a gasFee to the liquidation reward to ensure that the total liquidation reward always exceeds the gas costs.
