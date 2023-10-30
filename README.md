# Issue H-1: old borrowing key is used instead of `newBorrowingKey` when adding old loans to the newBorrowing in LiquidityBorrowingManager.takeOverDebt() 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/53 

## Found by 
0xMAKEOUTHILL, 0xMaroutis, 0xpep7, AuditorPraise, Nyx, ali\_shehab, jah, kutugu, p-tsanev, seeques, zraxx
when `_addKeysAndLoansInfo()` is called within LiquidityBorrowingManager.takeOverDebt(), old Borrowing Key is used and not `newBorrowingKey`  see [here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L440-L441)


## Vulnerability Detail
The old borrowing key credentials are deleted in  `_removeKeysAndClearStorage(oldBorrowing.borrower, borrowingKey, oldLoans);` see  [here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L429)

And a new borrowing key is created with the holdToken, saleToken, and the address of the user who wants to take over the borrowing in the `_initOrUpdateBorrowing()`. see [here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L430-L439)


now the old borrowing key whose credentials are already deleted is used to update the old loans in `_addKeysAndLoansInfo()` instead of the `newBorrowingKey` generated in  `_initOrUpdateBorrowing()` see [here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L440-L441)

## Impact
wrong borrowing Key is used (i.e the old borrowing key) when adding old loans to `newBorrowing` 

Therefore the wrong borrowing key (i.e the old borrowing key) will be added as borrowing key for tokenId of old Loans in `tokenIdToBorrowingKeys` in _addKeysAndLoansInfo()

(i.e when the bug of `update bool` being false, is corrected, devs should understand :))

## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L440-L441
## Tool used

Manual Review

## Recommendation
use newBorrowingKey when calling  `_addKeysAndLoansInfo()` instead of old borrowing key.



## Discussion

**fann95**

This is an inattention error related to the accelerated development of the project. I think we would have fixed this during testing, but thanks for the hint anyway. We will fix it.

# Issue H-2: Adversary can reenter takeOverDebt() during liquidation to steal vault funds 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/76 

## Found by 
0x52

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



## Discussion

**fann95**

Unfortunately, during development, we lost the nonReentrant-modifier just like checkDeadline. We will fix it.

# Issue H-3: Creditor can maliciously burn UniV3 position to permanently lock funds 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/78 

## Found by 
0x52, 0xJuda, ArmedGoose, Bauer, HHK, handsomegiraffe, mstpr-brainbot, talfao

LP NFT's are always controlled by the lender. Since they maintain control, malicious lenders have the ability to burn their NFT. Once a specific tokenID is burned the ownerOf(tokenID) call will always revert. This is problematic as all methodologies to repay (even emergency) require querying the ownerOf() every single token. Since this call would revert for the burned token, the position would be permanently locked.

## Vulnerability Detail

[NonfungiblePositionManager](https://etherscan.io/address/0xC36442b4a4522E871399CD717aBDD847Ab11FE88#code#F41#L114)

    function ownerOf(uint256 tokenId) public view virtual override returns (address) {
        return _tokenOwners.get(tokenId, "ERC721: owner query for nonexistent token");
    }

When querying a nonexistent token, ownerOf will revert. Now assuming the NFT is burnt we can see how every method for repayment is now lost.

[LiquidityManager.sol#L306-L308](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L306-L308)

            address creditor = underlyingPositionManager.ownerOf(loan.tokenId);
            // Increase liquidity and transfer liquidity owner reward
            _increaseLiquidity(cache.saleToken, cache.holdToken, loan, amount0, amount1);

If the user is being liquidated or repaying themselves the above lines are called for each loan. This causes all calls of this nature to revert.

[LiquidityBorrowingManager.sol#L727-L732](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L727-L732)

        for (uint256 i; i < loans.length; ) {
            LoanInfo memory loan = loans[i];
            // Get the owner address of the loan's token ID using the underlyingPositionManager contract.
            address creditor = underlyingPositionManager.ownerOf(loan.tokenId);
            // Check if the owner of the loan's token ID is equal to the `msg.sender`.
            if (creditor == msg.sender) {

The only other option to recover funds would be for each of the other lenders to call for an emergency withdrawal. The problem is that this pathway will also always revert. It cycles through each loan causing it to query ownerOf() for each token. As we know this reverts. The final result is that once this happens, there is no way possible to close the position.

## Impact

Creditor can maliciously lock all funds

## Code Snippet

[LiquidityBorrowingManager.sol#L532-L674](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532-L674)

## Tool used

Manual Review

## Recommendation

I would recommend storing each initial creditor when a loan is opened. Add try-catch blocks to each `ownerOf()` call. If the call reverts then use the initial creditor, otherwise use the current owner.

# Issue H-4: Incorrect calculations of borrowingCollateral leads to DoS for positions in the current tick range due to underflow 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/86 

## Found by 
0xReiAyanami, Bauer, Japy69, ali\_shehab, lil.eth, lucifero, peanuts, seeques
The `borrowingCollateral` is the amount of collateral a borrower needs to pay for his leverage. It should be calculated as the difference of holdTokenBalance (the amount borrowed + holdTokens received after saleTokens are swapped) and the amount borrowed and checked against user-specified maxCollateral amount which is the maximum the borrower wishes to pay. However, in the current implementation the `borrowingCollateral` calculation is most likely to underflow.
## Vulnerability Detail
This calculation is most likely to underflow
```solidity
uint256 borrowingCollateral = cache.borrowedAmount - cache.holdTokenBalance;
```
The `cache.borrowedAmount` is the calculated amount of holdTokens based on the liquidity of a position. `cache.holdTokenBalance` is the balance of holdTokens queried after liquidity extraction and tokens transferred to the `LiquidityBorrowingManager`. If any amounts of the saleToken are transferred as well, these are swapped to holdTokens and added to `cache.holdTokenBalance`. 

So in case when liquidity of a position is in the current tick range, both tokens would be transferred to the contract and saleToken would be swapped for holdToken and then added to `cache.holdTokenBalance`. This would make `cache.holdTokenBalance > cache.borrowedAmount` since `cache.holdTokenBalance == cache.borrowedAmount + amount of sale token swapped` and would make the tx revert due to underflow.
## Impact
Many positions would be unavailable to borrowers. For non-volatile positions like that which provide liquidity to stablecoin pools the DoS could last for very long period. For volatile positions that provide liquidity in a wide range this could also be for more than 1 year.

## Code Snippet
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L492-L503
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L470
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L848-L896
## Tool used

Manual Review

## Recommendation
The borrowedAmount should be subtracted from holdTokenBalance
```solidity
uint256 borrowingCollateral = cache.holdTokenBalance - cache.borrowedAmount;
```

# Issue H-5: `repay()` is prone to sandwich attacks 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/103 

## Found by 
0x52, 0xJuda, 0xMaroutis, Bandit, HHK, IceBear, Nyx, kaysoft, lucifero, pks\_

The [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) function of the LiquidityBorrowingManager contract is used to close a borrowing position.

When called by the borrower itself or by a liquidator bot, the amount of token received by the caller can be less than it should if the transaction is sandwiched. Greatly reducing potential profits for a borrower or liquidator.

## Vulnerability Detail

When a position is at profit, the lender can call [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) to close it. If the collateral if negative a liquidator can also call this function to reimburse collateral and profit from the `liquidationBonus` as well as the position's profits.

To close the position and realize the profits, if a the position is not out of range, a swap is made. Some `holdToken` are swapped for `saleToken` to be able to add the liquidity back to the position in the function [`_restoreLiquidity()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L223).

The current issue is that the minimum amount out of the swap is determined onchain which makes sandwich attacks possible as an MEV bot could bundle our transaction with 2 swaps one before and one after to affect the amount we receive on our swap effectively sandwiching out transaction.

The bot would make profits from our unrealized profits leading to the borrower or the liquidator to make less or even 0 profits.

## Impact

High. If the `repay()` transaction is sandwiched the profits will be greatly reduced.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L223

## Tool used

Manual Review

## Recommendation

Consider adding an array of `minOut` to the parameters that will be used for each loan swap, reducing the gas cost as we won't need to get a quote anymore as well as protect the users from sandwich attacks.



## Discussion

**fann95**

duplicate https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/109

# Issue H-6: No slippage protection during repayment due to dynamic slippage params and easily influenced `slot0()` 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/109 

## Found by 
0xblackskull, Bandit, Bauer, IceBear, Kral01, MohammedRizwan, kaysoft, lil.eth, lucifero, p-tsanev, peanuts, pks\_, talfao, tsvetanovv
The repayment function lacks slippage protection. It relies on slot0() to calculate sqrtLimitPrice, which in turn determines amounts for restoring liquidation. The dynamic calculation of slippage parameters based on these values leaves the function without adequate slippage protection, potentially reducing profit for the repayer.

## Vulnerability Detail
The absence of slippage protection can be attributed to two key reasons. Firstly, the `sqrtPrice` is derived from `slot0()`, **which can be easily manipulated:**
```Solidity
     function _getCurrentSqrtPriceX96(
        bool zeroForA,
        address tokenA,
        address tokenB,
        uint24 fee
    ) private view returns (uint160 sqrtPriceX96) {
        if (!zeroForA) {
            (tokenA, tokenB) = (tokenB, tokenA);
        }
        address poolAddress = computePoolAddress(tokenA, tokenB, fee);
        (sqrtPriceX96, , , , , , ) = IUniswapV3Pool(poolAddress).slot0(); //@audit-issue can be easily manipulated
    }
```
The calculated `sqrtPriceX96` is used to determine the amounts for restoring liquidation and the number of holdTokens to be swapped for saleTokens:
```Solidity
(uint256 holdTokenAmountIn, uint256 amount0, uint256 amount1) = _getHoldTokenAmountIn(
                params.zeroForSaleToken,
                cache.tickLower,
                cache.tickUpper,
                cache.sqrtPriceX96,
                loan.liquidity,
                cache.holdTokenDebt
            );
``` 
After that, the number of `SaleTokemAmountOut` is gained based on the sqrtPrice via QuoterV2.

Then, the slippage params are calculated 
`amountOutMinimum: (saleTokenAmountOut * params.slippageBP1000) /
                                Constants.BPS
                        })`
However, the `saleTokenAmountOut` is a dynamic number calculated on the current state of the blockchain, based on the calculations mentioned above. This will lead to the situation that the swap will always satisfy the `amountOutMinimum`.

As a result, if the repayment of the user is sandwiched (frontrunned), the profit of the repayer is decreased till the repayment satisfies the restored liquidity.

### Proof of concept
A Proof of Concept (PoC) demonstrates the issue with comments. Although the swap does not significantly impact a strongly founded pool, it does result in a loss of a few dollars for the repayer.

```Javascript
       let amountWBTC = ethers.utils.parseUnits("0.05", 8); //token0
        const deadline = (await time.latest()) + 60;
        const minLeverageDesired = 50;
        const maxCollateralWBTC = amountWBTC.div(minLeverageDesired);

        const loans = [
            {
                liquidity: nftpos[3].liquidity,
                tokenId: nftpos[3].tokenId,
            },
        ];

        const swapParams: ApproveSwapAndPay.SwapParamsStruct = {
            swapTarget: constants.AddressZero,
            swapAmountInDataIndex: 0,
            maxGasForCall: 0,
            swapData: swapData,
        };

let  params = {
            internalSwapPoolfee: 500,
            saleToken: WETH_ADDRESS,
            holdToken: WBTC_ADDRESS,
            minHoldTokenOut: amountWBTC,
            maxCollateral: maxCollateralWBTC,
            externalSwap: swapParams,
            loans: loans,
        };

await borrowingManager.connect(bob).borrow(params, deadline);

const borrowingKey = await borrowingManager.userBorrowingKeys(bob.address, 0);
        const swapParamsRep: ApproveSwapAndPay.SwapParamsStruct = {
            swapTarget: constants.AddressZero,
            swapAmountInDataIndex: 0,
            maxGasForCall: 0,
            swapData: swapData,
        };

       
        amountWBTC = ethers.utils.parseUnits("0.06", 8); //token0

let swapping: ISwapRouter.ExactInputSingleParamsStruct = {
            tokenIn: WBTC_ADDRESS,
            tokenOut: WETH_ADDRESS,
            fee: 500,
            recipient: alice.address,
            deadline: deadline,
            amountIn: ethers.utils.parseUnits("100", 8),
            amountOutMinimum: 0,
            sqrtPriceLimitX96: 0
        };
        await router.connect(alice).exactInputSingle(swapping);
        console.log("Swap success");

 let paramsRep: LiquidityBorrowingManager.RepayParamsStruct = {
            isEmergency: false,
            internalSwapPoolfee: 500,
            externalSwap: swapParamsRep,
            borrowingKey: borrowingKey,
            swapSlippageBP1000: 990, //<=slippage simulated
        };
 await borrowingManager.connect(bob).repay(paramsRep, deadline);
        // Without swap
// Balance of hold token after repay:  BigNumber { value: "993951415" }
// Balance of sale token after repay:  BigNumber { value: "99005137946252426108" }
// When swap
// Balance of hold token after repay:  BigNumber { value: "993951415" }
// Balance of sale token after repay:  BigNumber { value: "99000233164653177505" }
```

The following table shows difference of recieved sale token:
| Swap before repay transaction | Token |  Balance of user after Repay |
|---------------------------|---------------------|----------------------|
| No                        | WETH  | 99005137946252426108 |
| Yes                       | WETH  |99000233164653177505 |

The difference in the profit after repayment is 4904781599248603 weis, which is at the current market price of around 8 USD. The profit loss will depend on the liquidity in the pool, which depends on the type of pool and related tokens.

## Impact
The absence of slippage protection results in potential profit loss for the repayer.

## Code Snippet
[Slot0 is used here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L341)
[Dynamic slippage params are created here](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L265) - saleTokenAmount is dynamic variable calculated on the state of blockchain.

## Tool used

Manual Review

## Recommendation
To address this issue, avoid relying on slot0 and instead utilize Uniswap TWAP. Additionally, consider manually setting values for amountOutMin for swaps based on data acquired before repayment.



## Discussion

**fann95**

I think the severity level is medium since exchange within the pool is not the only way, there is also an external swap.

# Issue H-7: Borrower collateral that they are owed can get stuck in Vault and not sent back to them after calling `repay` 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/122 

## Found by 
HHK, detectiveking

There's a case where a borrower calls `borrow`, perhaps does a bunch of intermediate actions like calling `increaseCollateralBalance`, and then calls `repay` a short while later (so fees haven't had a time to increase), but the collateral they are owed is stuck in the `Vault` instead of being sent back to them after they repay. 

## Vulnerability Detail

First, let's say that a borrower called `borrow` in `LiquidityBorrowingManager`. Then, they call increase `increaseCollateralBalance` with a large collateral amount. A short time later, they decide they want to repay so they call `repay`. 

In `repay`, we have the following code:

```solidity
            if (
                collateralBalance > 0 &&
                (currentFees + borrowing.feesOwed) / Constants.COLLATERAL_BALANCE_PRECISION >
                Constants.MINIMUM_AMOUNT
            ) {
                liquidationBonus +=
                    uint256(collateralBalance) /
                    Constants.COLLATERAL_BALANCE_PRECISION;
            } else {
                currentFees = borrowing.dailyRateCollateralBalance;
            }
```

Notice that if we have `collateralBalance > 0` BUT `!((currentFees + borrowing.feesOwed) / Constants.COLLATERAL_BALANCE_PRECISION >
                Constants.MINIMUM_AMOUNT)` (i.e. the first part of the if condition is fine but the second is not. It makes sense the second part is not fine because the borrower is repaying not long after they borrowed, so fees haven't had a long time to accumulate), then we will still go to `currentFees = borrowing.dailyRateCollateralBalance;` but we will not do:

```solidity
                liquidationBonus +=
                    uint256(collateralBalance) /
                    Constants.COLLATERAL_BALANCE_PRECISION;
```

However, later on in the code, we have:

```solidity
            Vault(VAULT_ADDRESS).transferToken(
                borrowing.holdToken,
                address(this),
                borrowing.borrowedAmount + liquidationBonus
            );
``` 

So, the borrower's collateral will actually not even be sent back to the LiquidityBorrowingManager from the Vault (since we never incremented `liquidationBonus`). We later do:

```solidity
            _pay(borrowing.holdToken, address(this), msg.sender, holdTokenBalance);
            _pay(borrowing.saleToken, address(this), msg.sender, saleTokenBalance);
```

So clearly the user will not receive their collateral back. 

## Impact
User's collateral will be stuck in Vault when it should be sent back to them. This could be a large amount of funds if for example `increaseCollateralBalance` is called first. 

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L565-L575

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L632-L670

## Tool used

Manual Review

## Recommendation

You should separate:

```solidity
            if (
                collateralBalance > 0 &&
                (currentFees + borrowing.feesOwed) / Constants.COLLATERAL_BALANCE_PRECISION >
                Constants.MINIMUM_AMOUNT
            ) {
                liquidationBonus +=
                    uint256(collateralBalance) /
                    Constants.COLLATERAL_BALANCE_PRECISION;
            } else {
                currentFees = borrowing.dailyRateCollateralBalance;
            }
```

Into two separate if statements. One should check if `collateralBalance > 0`, and if so, increment liquidationBonus. The other should check `(currentFees + borrowing.feesOwed) / Constants.COLLATERAL_BALANCE_PRECISION >
                Constants.MINIMUM_AMOUNT` and if not, set `currentFees = borrowing.dailyRateCollateralBalance;`. 

# Issue M-1: DoS of lenders and gas griefing by packing tokenIdToBorrowingKeys arrays 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/15 

## Found by 
0xDetermination
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

# Issue M-2: No deadline and slippage check on `takeOverDebt()` can lead to unexpected results 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/51 

## Found by 
HHK, kutugu

The function [`takeOverDebt()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L395) in the LiquidityBorrowingManager contract doesn't have any deadline check like [`borrow`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L465) or [`repay`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532).

Additionally it also doesn't have any "slippage" check that would make sure the position hasn't changed between the moment a user calls [`takeOverDebt()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L395) and the transaction is confirmed.

## Vulnerability Detail

Blockchains are asynchronous by nature, when sending a transaction, the contract targeted can see its state changed affecting the result of our transaction.

When a user wants to call [`takeOverDebt()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L395) he expects that the debt he is gonna take over will be the same (or very close) as when he signed the transaction.

By not providing any deadline check, if the transaction is not confirmed for a long time then the user might end up with a position that is not as interesting as it was.

Take this example:

- A position on Ethereum is in debt of collateral but the borrowed tokens are at profit and the user think the token's price is going to keep increasing, it makes sense to take over.
- The user makes a transaction to take over.
- The gas price rise and the transaction takes longer than he thoughts to be confirmed.
- Eventually the transaction is confirmed but the position is not in profit anymore because price changed during that time.
- User paid the debt of the position for nothing as he won't be making profits.

Additionally when the collateral of a position is negative, lenders can call [`repay()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) as part of the "emergency liquidity restoration mode" which will reduce the size of the position. If this happens while another user is taking over the debt then he might end up with a position that is not as interesting as he thoughts.

Take this second example:

- A position on Ethereum with 2 loans is in debt of collateral but the borrowed tokens are at profit and the user think the token's price is going to keep increasing, it makes sense to take over.
- The user makes a transaction to take over.
- While the user's transaction is in the MEMPOOL a lender call ['repay()'](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L532) and get back his tokens.
- The user's transaction is confirmed and he take over the position but it only has 1 loan now as one of the 2 loans was sent back to the lender. the position might not be at profit anymore or less than it was supposed to be.
- User paid the debt of the position for nothing as he won't be making profits.

## Impact

Medium. User might pay collateral debt of a position for nothing.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L395

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L581

## Tool used

Manual Review

## Recommendation

Consider adding the modifier `checkDeadline()` as well as a parameter `minBorrowedAmount` and compare it to the current `borrowedAmount` to make sure no lender repaid their position during the take over.



## Discussion

**fann95**

we will add a deadline check.

# Issue M-3: Adversary can overwrite function selector in _patchAmountAndCall due to inline assembly lack of overflow protection 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/82 

## Found by 
0x52

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



## Discussion

**fann95**

I don't think this is a realistic scenario. You won't be able to get the offset to the function selector by overflowing the results of the multiplication or addition here. Using which index can you get the mload(0x40) ?

**Czar102**

I think overwriting the selector could be done if the `swapAmountInDataIndex` was `k * 2 ** 251 - 2` for any integer `k`. This is because `swapAmountInDataIndex * 0x20 =  k * 2 ** 251 * 2 ** 5 - 2 * 2 ** 5 = k * 2 ** 256 - 64 = - 64` mod `2 ** 256`. Changing memory at a pointer with such modification would collide with the selector at 4 least significant bytes and would write 28 bytes to previously allocated memory (memory corruption).
My recommendation would be to limit `swapAmountInDataIndex` to `div(data.length, 0x20)`. Then, one would be able to write `swapAmountInDataValue` at any correct index in the calldata or right behind the calldata, if that amount is not to show up in the function call.

# Issue M-4: Blacklisted creditor can block all repayment besides emergency closure 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/83 

## Found by 
0x52, ArmedGoose, Bauer, tsvetanovv

After liquidity is restored to the LP, accumulated fees are sent directly from the vault to the creditor. Some tokens, such as USDC and USDT, have blacklists the prevent users from sending or receiving tokens. If the creditor is blacklisted for the hold token then the fee transfer will always revert. This forces the borrower to defualt. LPs can recover their funds but only after the user has defaulted and they request emergency closure.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L306-L315

            address creditor = underlyingPositionManager.ownerOf(loan.tokenId);
            // Increase liquidity and transfer liquidity owner reward
            _increaseLiquidity(cache.saleToken, cache.holdToken, loan, amount0, amount1);
            uint256 liquidityOwnerReward = FullMath.mulDiv(
                params.totalfeesOwed,
                cache.holdTokenDebt,
                params.totalBorrowedAmount
            ) / Constants.COLLATERAL_BALANCE_PRECISION;

            Vault(VAULT_ADDRESS).transferToken(cache.holdToken, creditor, liquidityOwnerReward);

The following code is executed for each loan when attempting to repay. Here we see that each creditor is directly transferred their tokens from the vault. If the creditor is blacklisted for holdToken, then the transfer will revert. This will cause all repayments to revert, preventing the user from ever repaying their loan and forcing them to default. 

## Impact

Borrowers with blacklisted creditors are forced to default

## Code Snippet

[LiquidityManager.sol#L223-L321](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L223-L321)

## Tool used

Manual Review

## Recommendation

Create an escrow to hold funds in the event that the creditor cannot receive their funds. Implement a try-catch block around the transfer to the creditor. If it fails then send the funds instead to an escrow account, allowing the creditor to claim their tokens later and for the transaction to complete.

# Issue M-5: No check on liquidation and daily rates update while borrowing 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/97 

## Found by 
HHK

When a user calls the [`borrow()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L465) function of the LiquidityBorrowingManager contract. He expect that the `currentDailyRate` and `liquidationBonusForToken` are the same as what was displayed by the UI.

But given asynchronous nature of blockchains, an admin could update these values for the tokens the borrower is about to borrow and result in more token charged than expected.

## Vulnerability Detail

During the [`borrow()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L465) function, the token to be sent by the borrower are computed using multiple state variables that define the rates and default amounts for the tokens to borrow.

These variables can be modified by the `owner/dailyRateOperator`. While the function check the `deadline` and `maxCollateral` given by the borrower. It doesn't check that the `currentDailyRate` nor `liquidationBonusForToken` changes.

If these values are changed while the transaction is being confirmed, it could result in more tokens charged to the user than he expected.

## Impact

Medium. If some state parameters are changed while a borrower's transaction is being confirmed it could result in more token charged and a different daily fee than expected.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L465
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/LiquidityBorrowingManager.sol#L211
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/OwnerSettings.sol#L65

## Tool used

Manual Review

## Recommendation

Consider adding `maxDailyRate` and `maxLiquidationBonus` parameters and check them.

# Issue M-6: `computePoolAddress()` will not work on ZkSync Era 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/104 

## Found by 
0x52, HHK, shogoki

When using the wagmi protocol, multiple swap can happen when borrowing or repaying a position. When the swap uses Uniswap v3 it checks that the callback is a pool by computing the address but the computation won't match on ZkSync Era.

## Vulnerability Detail

When borrowing or repaying a position a user can either use a custom router that was approved by the wagmi team to make the swaps required or can use Uniswap v3 as a fallback.

When using the Uniswap v3 as a fallback the [`_v3SwapExactInput()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L204) internal function is being called. This function uses [`computePoolAddress()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L271) to find the pool address to use. [`computePoolAddress()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L271) is also used during the [`uniswapV3SwapCallback()`](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L242) to make sure the `msg.sender` is a valid pool.

On ZkSync Era the `create2` addresses are not computed the same way see [here](https://era.zksync.io/docs/reference/architecture/differences-with-ethereum.html#address-derivation).

This will result in the swaps on Uniswapv3 to revert. If a user was able to open a position using a custom router but the custom router is removed later on by the team or if the liquidity was one sided so no swap happened. The borrower and liquidators could find themself not able to close the positions until a new router is whitelisted.

The borrower could be forced to pay collateral for a longer time as he won't be able to close his position.

## Impact

Medium. Unlikely to happen but would result in short-term DOS and more fees paid by the borrower.

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L146
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L204
https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/ApproveSwapAndPay.sol#L271

## Tool used

Manual Review

## Recommendation

Consider calling the Uniswap factory getter `getPool()` to get the address of the pool.



## Discussion

**fann95**

This is too obvious a problem with which this project simply will not work. We know about this, so we will make changes before deployment in ZkSync Era.

**Czar102**

As the contest readme states, watsons were to consider zkSync as one of the chains the code in scope was to be deployed on. If watsons couldn't have known that a modification of the code in scope would be deployed on zkSync, I don't see a reason to invalidate this issue, even if it was previously considered by the protocol team and/or is trivial.

**fann95**

I’m pasting the solution for Sherlock, but we don’t plan to make any fixes to this issue right now.

# Issue M-7: The `FullMath` library used in `LiquidityBorrowingManager.sol` and `DailyRateAndCollateral.sol` is unable to handle intermediate overflows due to overflow 

Source: https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/138 

## Found by 
MohammedRizwan
The FullMath.sol library used in `LiquidityBorrowingManager.sol` and `DailyRateAndCollateral.sol` contracts doesn't correctly handle the case when an intermediate value overflows 256 bits. This happens because an overflow is desired in this case but it's never reached.

## Vulnerability Detail
The FullMath.sol library was taken from Uniswap. However, the original solidity version that was used was in this FullMath.sol library was `< 0.8.0`, Which means that the execution didn't revert when an overflow was reached. This effectively means that when a phantom overflow (a multiplication and division where an intermediate value overflows 256 bits) occurs the execution will revert and the correct result won't be returned. 

`mulDivRoundingUp()` from the `FullMath.sol` has been used in below in scope contracts,

1) In `LiquidityBorrowingManager.sol` having functions `calculateCollateralAmtForLifetime()`, `_precalculateBorrowing()` and `_getDebtInfo()`

2) In `DailyRateAndCollateral.sol` having function `_calculateCollateralBalance()`

The values returned from these functions wont be correct and the execution of these functions will be reverted when phantom overflow occurs.

## Impact
The correct result isn't returned in this case and the execution gets reverted when a phantom overflows occurs.

## Code Snippet
Code reference: https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/vendor0.8/uniswap/FullMath.sol#L119-L129

The `FullMath.sol` having `mulDivRoundingUp()` function used `LiquidityBorrowingManager.sol` and `DailyRateAndCollateral.sol`  doesn't use an unchecked block and it is shown as below(_removed comments to shorten the code_),

```Solidity
File: wagmi-leverage/contracts/vendor0.8/uniswap/FullMath.sol

// SPDX-License-Identifier: MIT

pragma solidity ^0.8.4;

library FullMath {

    function mulDiv(
        uint256 a,
        uint256 b,
        uint256 denominator
    ) internal pure returns (uint256 result) {
        unchecked {                                 @audit // used unchecked block here but not used in mulDivRoundingUp()
            uint256 prod0; // Least significant 256 bits of the product
            uint256 prod1; // Most significant 256 bits of the product
            assembly {
                let mm := mulmod(a, b, not(0))
                prod0 := mul(a, b)
                prod1 := sub(sub(mm, prod0), lt(mm, prod0))
            }


           // some code


            result = prod0 * inv;
            return result;
        }
    }


    function mulDivRoundingUp(
        uint256 a,
        uint256 b,
        uint256 denominator
    ) internal pure returns (uint256 result) {
        result = mulDiv(a, b, denominator);                @audit // does not use unchecked block similar to mulDiv()
        if (mulmod(a, b, denominator) > 0) {
            require(result < type(uint256).max);
            result++;
        }
    }
}
```

## Tool used
Manual Review

## Recommendation
It is advised to put the entire function bodies of `mulDivRoundingUp()` similar to `mulDiv()` in an unchecked block. A modified version of the original Fullmath library that uses unchecked blocks to handle the overflow, can be found in the 0.8 branch of the [Uniswap v3-core repo](https://github.com/Uniswap/v3-core/blob/0.8/contracts/libraries/FullMath.sol).

Per `Uniswap-v3-core`, Do below changes in `FullMath.sol`

```diff

    function mulDivRoundingUp(
        uint256 a,
        uint256 b,
        uint256 denominator
    ) internal pure returns (uint256 result) {
+       unchecked {
            result = mulDiv(a, b, denominator);
            if (mulmod(a, b, denominator) > 0) {
                require(result < type(uint256).max);
                result++;
           }
+       }
    }
```

