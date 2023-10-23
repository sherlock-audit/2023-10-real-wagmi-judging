Ancient Malachite Jay

high

# Adversary can reenter takeOverDebt() during liquidation to steal vault funds
## Summary

Due to the lack of nonReentrant modifier on takeOverDebt() a liquidatable position can be both liquidated and transferred simultaneously. This results in LPs being repaid from the vault while the position and loans continue to be held open, effectively duplicating the liquidated position. LPs therefore get to 'double dip' from the vault, stealing funds and causing a deficit. This can be abused by an attacker who borrows against their own LP to exploit the 'double dip' for profit.

## Vulnerability Detail

First we'll walk through a high level breakdown of the issue to have as context for the rest of the report:

 1) Create a custom token that allows them to take control of the transaction and to prevent liquidation
 2) Fund UniV3 LP with target token and custom token
 3) Borrow against LP with target token as the hold token
 4) After some time the position become liquidatable
 5) Begin liquidating the position via repay()
 6) Utilize the custom token during the swap in repay() to gain control of the transaction
 7) Use control to reenter into takeOverDebt() since it lack nonReentrant modifier
 8) Loan is now open on a secondary address and closed on the initial one
 8) Transaction resumes (post swap) on repay() 
 9) Finish repayment and refund all initial LP
10) Position is still exists on new address
11) After some time the position become liquidatable
12) Loan is liquidated and attacker is paid more LP
13) Vault is at a deficit due to refunding LP twice
14) Repeat until the vault is drained of target token

[LiquidityManager.sol#L279-L287](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L279-L287)

    _v3SwapExactInput(
        v3SwapExactInputParams({
            fee: params.fee,
            tokenIn: cache.holdToken,
            tokenOut: cache.saleToken,
            amountIn: holdTokenAmountIn,
            amountOutMinimum: (saleTokenAmountOut * params.slippageBP1000) /
                Constants.BPS
        })

The control transfer happens during the swap to UniV3. Here when the custom token is transferred, it gives control back to the attacker which can be used to call takeOverDebt().

[LiquidityBorrowingManager.sol#L667-L672](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L667-L672)

        _removeKeysAndClearStorage(borrowing.borrower, params.borrowingKey, loans);
        // Pay a profit to a msg.sender
        _pay(borrowing.holdToken, address(this), msg.sender, holdTokenBalance);
        _pay(borrowing.saleToken, address(this), msg.sender, saleTokenBalance);

        emit Repay(borrowing.borrower, msg.sender, params.borrowingKey);

The reason the reentrancy works is because the actual borrowing storage state isn't modified until AFTER the control transfer. This means that the position state is fully intact for the takeOverDebt() call, allowing it to seamlessly transfer to another address behaving completely normally. After the repay() call resumes, _removeKeysAndClearStorage is called with the now deleted borrowKey. 

[Keys.sol#L31-L42](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/libraries/Keys.sol#L31-L42)

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

The unique characteristic of deleteKey is that it doesn't revert if the key doesn't exist. This allows "removing" keys from an empty array without reverting. This allows the repay call to finish successfully.

[LiquidityBorrowingManager.sol#L450-L452](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L450-L452)

        //newBorrowing.accLoanRatePerSeconds = oldBorrowing.accLoanRatePerSeconds;
        _pay(oldBorrowing.holdToken, msg.sender, VAULT_ADDRESS, collateralAmt + feesDebt);
        emit TakeOverDebt(oldBorrowing.borrower, msg.sender, borrowingKey, newBorrowingKey);

Now we can see how this creates a deficit in the vault. When taking over an existing debt, the user is only required to provide enough hold token to cover any fee debt and any additional collateral to pay fees for the newly transferred position. This means that the user isn't providing any hold token to back existing LP.

[LiquidityBorrowingManager.sol#L632-L636](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L632-L636)

            Vault(VAULT_ADDRESS).transferToken(
                borrowing.holdToken,
                address(this),
                borrowing.borrowedAmount + liquidationBonus
            );

On the other hand repay transfers the LP backing funds from the vault. Since the same position is effectively liquidated twice, it will withdraw twice as much hold token as was originally deposited and no new LP funds are added when the position is taken over. This causes a deficit in the vault since other users funds are being withdrawn from the vault.

## Impact

Vault can be drained

## Code Snippet

[LiquidityBorrowingManager.sol#L395-L453](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L395-L453)

## Tool used

Manual Review

## Recommendation

Add the `nonReentrant` modifier to `takeOverDebt()`