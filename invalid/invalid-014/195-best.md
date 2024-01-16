Soft Coconut Mongoose

medium

# Collateral balance in USD may use stale oracle price

## Summary

The function `collateralUsdBalance()` calculates its result using the prices stored in the `collateralPrices` array, which will likely be outdated.

## Vulnerability Detail

The `collateralUsdBalance()` function calculates the total value of all collaterals by multiplying the balance of each one by its price. Its implementation is given by:

```solidity
    function collateralUsdBalance()
        internal
        view
        returns (uint256 balanceTally)
    {
        UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();
        uint256 collateralTokensCount = poolStorage.collateralAddresses.length;
        balanceTally = 0;
        for (uint256 i = 0; i < collateralTokensCount; i++) {
            balanceTally += freeCollateralBalance(i)
                .mul(10 ** poolStorage.missingDecimals[i])
                .mul(poolStorage.collateralPrices[i])
                .div(UBIQUITY_POOL_PRICE_PRECISION);
        }
    }
```

These prices are fetched from the internal array `poolStorage.collateralPrices` that stores collateral prices in contract storage. However these prices are only updated whenever the `updateChainLinkCollateralPrice()` is called.

Since `collateralUsdBalance()` doesn't trigger the update of prices (in fact, it can't as it is a view function), the return value of this function will be likely incorrect as it potentially uses stale prices.

Note that the same happens with `getDollarInCollateral()` when called externally (not the internal usages through `mintDollar()` or `redeemDollar()` as both of these execute an update before calling the function).

## Impact

The result of `collateralUsdBalance()` and `getDollarInCollateral()` will be inaccurate due to the staleness of the collateral prices.

## Code Snippet

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L247-L261

## Tool used

Manual Review

## Recommendation

Refactor the logic to fetch the price from a Chainlink feed into a view function in order to use this in `collateralUsdBalance()` and the external variant of `getDollarInCollateral()`, instead of using the prices stored in `poolStorage.collateralPrices`.