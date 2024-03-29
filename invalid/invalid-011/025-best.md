Damp Fossilized Canary

medium

# LibUbiquityPool.setCollateralChainLinkPriceFeed() fails to check that collateralAddress is valid.

## Summary
LibUbiquityPool.setCollateralChainLinkPriceFeed() fails to check that collateralAddress is valid.  The problem is that the collateralIndex of an invalid collateral address is 0. As a result, the ColllateralChainLinkPriceFeed is set for the collateral with  collateralIndex  = 0. This would be mistake, as the wrong colleteral is udpated. 

## Vulnerability Detail
LibUbiquityPool.setCollateralChainLinkPriceFeed() fails to check that collateralAddress is valid

Meanwhile the following line will return collateralIndex  = 0: 

```javascript
 uint256 collateralIndex = poolStorage.collateralIndex[
            collateralAddress
        ];
```
As a result, instead of updating the collateralPriceFeedAddresses for the given collateral address, the collateral with index 0 is updated. 


```javascript
  poolStorage.collateralPriceFeedAddresses[
            collateralIndex
        ] = chainLinkPriceFeedAddress;
```

This is a mistake since the wrong colleteral price feed is updated. 

## Impact
LibUbiquityPool.setCollateralChainLinkPriceFeed() fails to check that collateralAddress is valid.  The wrong (price feed for index 0) price feed will be set for collateral with index of zero. 

## Code Snippet
[https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L700-L726](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L700-L726)


## Tool used
VScode

Manual Review

## Recommendation
LibUbiquityPool.setCollateralChainLinkPriceFeed() should check that the collateral is valid by checking that  poolStorage.isCollateralEnabled[collateralAddress] is true.