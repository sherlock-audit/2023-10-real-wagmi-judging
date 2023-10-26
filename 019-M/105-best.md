Big Charcoal Cod

medium

# The liquidation bonus is not distributed fairly during emergency liquidations, leading to backrunning.
## Summary
During an emergency repayment (liquidation), the liquidation bonus is currently awarded to only the last liquidator. This behaviour leads to backrunning. For fairness and accuracy, the liquidation bonus should be distributed on a per-loan basis.

## Vulnerability Detail
The issue arises when owners of loans initiate an emergency repayment. In the current implementation, only the final repayment, which closes the entire borrowing position, receives the liquidation bonus. This is evident in the code snippet below:
```Solidity
if (completeRepayment) {
                LoanInfo[] memory empty;
                _removeKeysAndClearStorage(borrowing.borrower, params.borrowingKey, empty);
                feesAmt += liquidationBonus; // @audit-issue this not good as the liqudation bonus gets just to the last emergency liquidator 
```
However, the liquidation bonus should be distributed individually for each loan, as indicated in the calculation of the bonus:
```Solidity
function getLiquidationBonus(
        address token,
        uint256 borrowedAmount,
        uint256 times
    ) public view returns (uint256 liquidationBonus) {
       ...
        liquidationBonus *= times; //@audit-info here makes liquidation bonus *= loans.length
    }
```
Therefore, during emergency repayments, the bonus should be distributed based on the number of loans to ensure fairness. Failure to do so may result in backrunning among position owners, with the last liquidator receiving the entire bonus.

## Impact
This vulnerability leads to the loss of the "liquidation bonus" for the lenders who were the first to liquidate (repay) their positions based on the incorrect model of asset distribution.

## Code Snippet
Just the last liquidator(owner of position) gets a liqudation bonus: [Liquidation bonus distribution](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L605)
However, the calculation of the bonus is done like this: [Liquidation bonus calculation](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L702)

## Tool used

Manual Review

## Recommendation
To address this issue, it is recommended to distribute the liquidation bonus evenly among all loans during an emergency repayment. Modify the code to divide the bonus by the number of loans, ensuring that each loan's liquidator receives a fair share of the bonus.
