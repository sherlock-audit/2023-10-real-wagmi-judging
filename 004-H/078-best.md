Ancient Malachite Jay

high

# Creditor can maliciously burn UniV3 position to permanently lock funds
## Summary

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