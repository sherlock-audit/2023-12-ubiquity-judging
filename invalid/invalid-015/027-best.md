Damp Fossilized Canary

medium

# LibUbiquityPool.addCollateralToken() might add duplicate collateral to the protocol.

## Summary
LibUbiquityPool.addCollateralToken() might add duplicate collateral to the protocol, which leads to some confusions in the protocoL:
1) freeCollateralBalance will return the wrong value;
2) For each collateral, isBorrowPaused, isMintedPaused and isRedeemPaused might have two states. 

## Vulnerability Detail()
LibUbiquityPool.addCollateralToken() might add duplicate collateral to the protocol since it does not check the given collateral is already in the protocol or not. 

As a result, there might be duplicate indices for the same collateral. For example, a collateral C might have two indices: 3 and 5, such that both of them are mapped to the same collateral C. Meanwhile, collateralIndex(c) = 5.

As a result,  freeCollateralBalance(5) will return the wrong result since it does not consider unclaimedPoolCollateral[3] which also for collateral C.

Meanwhile, isBorrowPaused(3) and isBorrowPaused(5) might be inconsitent (one is true and the other is false), similary, isMintedPaused(3) and isMintedPaused(5) might be inconsistent;, isRedeemPaused  and isRedeemPaused might be inconsistent also. 

That means, for the same collateral, depending on which index to use, one user can redeeem and another user cannot. One user can mint and another user cannot. This is not a desirable result. 

## Impact
LibUbiquityPool.addCollateralToken() might add duplicate collateral to the protocol, leading to consistent states of the protocol and wrong returned value from  freeCollateralBalance().

## Code Snippet

[https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L629-L680](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L629-L680)

## Tool used

VScode

Manual Review

## Recommendation
Check and ensure that no duplicate collateral is added. 