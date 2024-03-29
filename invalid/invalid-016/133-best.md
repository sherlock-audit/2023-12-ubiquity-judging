Curly Cinnamon Pony

medium

# Lack of validation for `latestRoundData`

## Summary

`priceFeed.latestRoundData` should validate all the returned data from chainlink.

## Vulnerability Detail

`priceFeed.latestRoundData` needs to validate that roundId (1st value) is smaller or equal to and answeredInRound (last value) to prevent retrieving bad values from chainlink.

## Impact

Is chainlink returns an invalid value for a collateral, e.g. LUSD, then the protocol is vulnerable to executing incorrect minting and redeeming flows since the collateral is used to price the amount of ubiquity dollar that can be minted/redeemed.

## Code Snippet

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L530-L539

## Tool used

Manual review

## Recommendation

```solidity
(
    roundId,
    int256 answer,
    /* startedAt */,
    uint256 updatedAt,
    answeredInRound
) = priceFeed.latestRoundData();
require(answeredInRound >= roundID, "Stale price");
```