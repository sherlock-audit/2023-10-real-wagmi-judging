Future Blue Alpaca

medium

# latestUpTimestamp isn't recorded
## Summary

lastestUpTimestamp isn't recorded in _getHOldTokenRateInfo()

## Vulnerability Detail

holdTokenRateInfo is a _getHOldTokenRateInfo() return variable containing information about the token holding rate from the holdTokenInfo array. It is declared as a memory variable, so it contains the value and not a reference to the holdTokenInfo[key] mapping.
The update is therefore not recorded. However, the protocol assumes that the last update timestamp is recorded for future calculations.

## Impact

Incorrect calculation of the accumulated loan rate per second and the collateral balance

## Code Snippet

https://github.com/sherlock-audit/2023-10-real-wagmi/blob/main/wagmi-leverage/contracts/abstract/DailyRateAndCollateral.sol#L25-L55

```solidity
    /**
     * @notice This internal view function retrieves the current daily rate for the hold token specified by `holdToken`
     * in relation to the sale token specified by `saleToken`. It also returns detailed information about the hold token rate stored
     * in the `holdTokenInfo` mapping. If the rate is not set, it defaults to `Constants.DEFAULT_DAILY_RATE`. If there are any existing
     * borrowings for the hold token, the accumulated loan rate per second is updated based on the time difference since the last update and the
     * current daily rate. The latest update timestamp is also recorded for future calculations.
     * @param saleToken The address of the sale token in the pair.
     * @param holdToken The address of the hold token in the pair.
     * @return currentDailyRate The current daily rate for the hold token.
     * @return holdTokenRateInfo The struct containing information about the hold token rate.
     */
    function _getHoldTokenRateInfo(
        address saleToken,
        address holdToken
    ) internal view returns (uint256 currentDailyRate, TokenInfo memory holdTokenRateInfo) {
        bytes32 key = Keys.computePairKey(saleToken, holdToken);
        holdTokenRateInfo = holdTokenInfo[key];
        currentDailyRate = holdTokenRateInfo.currentDailyRate;
        if (currentDailyRate == 0) {
            currentDailyRate = Constants.DEFAULT_DAILY_RATE;
        }
        if (holdTokenRateInfo.totalBorrowed > 0) {
            uint256 timeWeightedRate = (uint32(block.timestamp) -
                holdTokenRateInfo.latestUpTimestamp) * currentDailyRate;
            holdTokenRateInfo.accLoanRatePerSeconds +=
                (timeWeightedRate * Constants.COLLATERAL_BALANCE_PRECISION) /
                1 days;
        }

        holdTokenRateInfo.latestUpTimestamp = uint32(block.timestamp);
    }
```

## Tool used

Manual Review

## Recommendation
