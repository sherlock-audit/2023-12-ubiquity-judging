Radiant Charcoal Horse

medium

# LibTWAPOracle::update temporary redemption DOS if uAD/3CRV metapool has very low liquidity

## Summary
If the metapool 3CRV/uAD has very low liquidity, the spot price may be deviated, which means that as a consequence the TWAP value computed is deviated, and when TWAP of uAD is too high, users withdrawals will revert. This causes a DOS on withdrawals until liquidity in the metapool becomes sufficient again

## Vulnerability Detail
We can see in `LibTWAPOracle::update`, the function `IMetapool::get_dy()` is used to retrieve the spot price on averaged balances. The amount in, for which to retrieve amount out is hardcoded to `1 ether`:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L84-L89

This means that if the liquidity in the pool is less than 0.9 ether of 3CRV, the value returned by `get_dy` will be less than 0.9 ether, and the `redeemThreshold` will not be verified:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L418-L421

Temporarily blocking withdrawals

## Impact
Withdrawals are temporarily blocked until enough liquidity is added to the metapool

## Code Snippet

## Tool used

Manual Review

## Recommendation
Use an adaptive parameter for amount in for this estimation, or do not update if the liquidity is too low