Sunny Black Anteater

medium

# Unhandled chainlink revert would lock all price oracle access

## Summary
The collateral token price will fail to update once the ChainLink price feed is blocked from visiting.
## Vulnerability Detail
A call to `latestRoundData` could potentially revert and make it impossible to query prices. Feeds cannot be changed after they are configured through the function `addCollateralToken()` so this would result in a permanent denial of service;

Refer to https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles for more information regarding potential risks to account for when relying on external price feed providers.

## Impact
The collateral token price will fail to update.
## Code Snippet
https://github.com/sherlock-audit/2023-12-ubiquity/tree/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L526-L539
## Tool used

Manual Review

## Recommendation
Surround the call to `latestRoundData()` with `try/catch` instead of calling it directly. When the call reverts, the catch block can be used to call a fallback oracle or handle the error in any other suitable way.