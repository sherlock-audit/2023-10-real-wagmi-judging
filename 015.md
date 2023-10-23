High Chartreuse Hedgehog

high

# DoS of lenders and gas griefing by packing tokenIdToBorrowingKeys arrays
## Summary
In `LiquidityBorrowingManager`, `tokenIdToBorrowingKeys` arrays can be packed to gas grief and cause DoS of specific loans for an arbitrary period of time.
## Vulnerability Detail
`LiquidityBorrowingManager.borrow()` calls the function `_addKeysAndLoansInfo()`, which adds user keys to the `tokenIdToBorrowingKeys` array of the borrowed-from LP position:
```solidity
    function _addKeysAndLoansInfo(
        bool update,
        bytes32 borrowingKey,
        LoanInfo[] memory sourceLoans
    ) private {
        // Get the storage reference to the loans array for the borrowing key
        LoanInfo[] storage loans = loansInfo[borrowingKey];
        // Iterate through the sourceLoans array
        for (uint256 i; i < sourceLoans.length; ) {
            // Get the current loan from the sourceLoans array
            LoanInfo memory loan = sourceLoans[i];
            // Get the storage reference to the tokenIdLoansKeys array for the loan's token ID
            bytes32[] storage tokenIdLoansKeys = tokenIdToBorrowingKeys[loan.tokenId];
            // Conditionally add or push the borrowing key to the tokenIdLoansKeys array based on the 'update' flag
            update
                ? tokenIdLoansKeys.addKeyIfNotExists(borrowingKey)
                : tokenIdLoansKeys.push(borrowingKey);
    ...
```
A user key is calculated in the `Keys` library like so:
```solidity
    function computeBorrowingKey(
        address borrower,
        address saleToken,
        address holdToken
    ) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(borrower, saleToken, holdToken));
    }
```
So every time a new user borrows some amount from a LP token, a new `borrowKey` is added to the `tokenIdToBorrowingKeys[LP_Token_ID]` array. The problem is that this array is iterated through by calling iterating methods (`addKeyIfNotExists()` or `removeKey()`) in the `Keys` library when updating a borrow (as seen in the first code block). Furthermore, emergency repays call `removeKey()` in `_calculateEmergencyLoanClosure()`, non-emergency repays call `removeKey()` in `_removeKeysAndClearStorage()`, and `takeOverDebt()` calls `removeKey()` in `_removeKeysAndClearStorage()`. The result is that all exit/repay/liquidation methods must iterate through the array. Both of the iterating methods in the `Keys` library access storage to compare array values to the key passed as argument, so every key in the array before the argument key will increase the gas cost of the transaction by (more than) a cold `SLOAD`, which costs 2100 gas (https://eips.ethereum.org/EIPS/eip-2929). Library methods below:
```solidity
    function addKeyIfNotExists(bytes32[] storage self, bytes32 key) internal {
        uint256 length = self.length;
        for (uint256 i; i < length; ) {
            if (self.unsafeAccess(i).value == key) {
                return;
            }
            unchecked {
                ++i;
            }
        }
        self.push(key);
    }

    function removeKey(bytes32[] storage self, bytes32 key) internal {
        uint256 length = self.length;
        for (uint256 i; i < length; ) {
            if (self.unsafeAccess(i).value == key) {
                self.unsafeAccess(i).value = self.unsafeAccess(length - 1).value;
                self.pop();
                break;
            }
            unchecked {
                ++i;
            }
        }
    }
```
Let's give an example to see the potential impact and cost of the attack:
1. An LP provider authorizes the contract to give loans from their large position. Let's say USDC/WETH pool.
2. The attacker sees this and takes out minimum borrows of USDC using different addresses to pack the position's `tokenIdToBorrowingKeys` array. In `Constants.sol`, `MINIMUM_BORROWED_AMOUNT = 100000` so the minimum borrow is $0.1 dollars since USDC has 6 decimal places. Add this to the estimated gas cost of the borrow transaction, let's say $3.9 dollars. The cost to add one key to the array is approx. $4. The max block gas limit on ethereum mainnet is `30,000,000`,  so divide that by 2000 gas, the approximate gas increase for one key added to the array. The result is 15,000, therefore the attacker can spend 60000 dollars to make any new borrows from the LP position unable to be repaid, transferred, or liquidated. Any new borrow will be stuck in the contract.
3. The attacker now takes out a high leverage borrow on the LP position, for example $20,000 in collateral for a $1,000,000 borrow. The attacker's total expenditure is now $80,000, and the $1,000,000 from the LP is now locked in the contract for an arbitrary period of time.
4. The attacker calls `increaseCollateralBalance()` on all of the spam positions. Default daily rate is .1% (max 1%), so over a year the attacker must pay 36.5% of each spam borrow amount to avoid liquidation and shortening of the array. If the gas cost of increasing collateral is $0.5 dollars, and the attacker spends another $0.5 dollars to increase collateral for each spam borrow, then the attacker can spend $1 on each spam borrow and keep them safe from liquidation for over 10 years for a cost of $15,000 dollars. The total attack expenditure is now $95,000. The protocol cannot easily increase the rate to hurt the attacker, because that would increase the rate for all users in the USDC/WETH market. Furthermore, the cost of the attack will not increase that much even if the daily rate is increased to the max of 1%. The attacker does not need to increase the collateral balance of the $1,000,000 borrow since repaying that borrow is DoSed. 
6. The result is that $1,000,000 of the loaner's liquidity is locked in the contract for over 10 years for an attack cost of $95,000.
## Impact
Array packing causes users to spend more gas on loans of the affected LP token. User transactions may out-of-gas revert due to increased gas costs. An attacker can lock liquidity from LPs in the contract for arbitrary periods of time for asymmetric cost favoring the attacker. The LP will earn very little fees over the period of the DoS.
## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L100-L101
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L790-L826
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/libraries/Keys.sol
## Tool used
Manual Review
## Recommendation
`tokenIdToBorrowingKeys` tracks borrowing keys and is used in view functions to return info (getLenderCreditsCount() and getLenderCreditsInfo()). This functionality is easier to implement with arrays, but it can be done with mappings to reduce gas costs and prevent gas griefing and DoS attacks. For example the protocol can emit the borrows for all LP tokens and keep track of them offchain, and pass borrow IDs in an array to a view function to look them up in the mapping. Alternatively, OpenZeppelin's EnumerableSet library could be used to replace the array and keep track of all the borrows on-chain.