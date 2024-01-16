Damp Fossilized Canary

medium

# LibUbiquityPool.collectRedemption() lacks modifier collateralEnabled(collateralIndex).

## Summary
LibUbiquityPool.collectRedemption() lacks modifier collateralEnabled(collateralIndex). As a result, even when a collateral is disabled, a user can still collected the redeemed collateral. 

## Vulnerability Detail
The redeeming of collateral takes two steps:  1. `redeemDollar()` and 2 `collectRedemption()`.

These two steps cannot be be performed in the same block to avoid flashload manipulation. 

While the `redeemDollar()` has the collateralEnabled(collateralIndex) modifier, the `collectRedemption()` does not. 
As a result, if the collateral is disabled between the two steps to disable redeeming. The disnabling will fail since the second does not have the modifier  collateralEnabled(collateralIndex). The user will still be able to complete the redemption process. 


## Impact
LibUbiquityPool.collectRedemption() lacks modifier collateralEnabled(collateralIndex). The admin/owner cannot stop the redeeming process of two steps once the first step is completed. Even a collateral is disabled, a user can still call LibUbiquityPool.collectRedemption() to collect the collateral.  Stopping the collateral transfer will not succeed.

## Code Snippet

[https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L476-L517](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L476-L517)

## Tool used
VScode

Manual Review

## Recommendation

Add the modifier collateralEnabled(collateralIndex) to LibUbiquityPool.collectRedemption()