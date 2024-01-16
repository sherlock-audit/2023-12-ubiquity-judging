Radiant Charcoal Horse

high

# UbiquityPool::mintDollar/redeemDollar collateral depeg will encourage using UbiquityPool to swap for better collateral

## Summary
In the case of a depeg of an underlying collateral, UbiquityPool mechanism incentivises users to fill it up with the depegging collateral and taking out the better collateral. This means uAD ultimately depegs as well.

## Vulnerability Detail
Chainlink price may be slightly outdated with regards to actual Dex state, and in that case users holding a depegging asset (let's consider `DAI` depegging) will use uAD to swap for the still pegged collateral: `LUSD`. By doing that they expect a better execution than on Dexes, because they swap at the price of chainlink minus the uAD fee:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L358-L364

This in turn fills the reserves of UbiquityPool with the depegging collateral and depletes the reserves of `good` collateral.

## Impact
A depegging collateral will cause uAD to depeg because users are incentivised to use the pool to swap for the `better` asset

## Code Snippet

## Tool used
Manual Review

## Recommendation
Multiple solutions may be studied:
- Enforce a ratio between different collateral reserves (somewhat like GMX pricing algo which also enables users to swap with zero slippage using chainlink feeds)
- Use a safety minting ratio (LTV mechanism similar to borrowing protocols)
- Force chainlink feeds to stay within acceptable thresholds for stable coins (revert operations on collateral if price is out of range)