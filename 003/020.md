Radiant Charcoal Horse

medium

# LibUbiquityPool::mintDollar/redeemDollar reliance on arbitrarily short TWAP oracle may be inefficient for preventing depeg

## Summary
The ubiquity pool used for minting/burning uAD relies on a twap oracle which can be outdated because the underlying metapool is not updated when calling the ubiquity pool. This would mean that minting/burning will be enabled based on an outdated state when it should have been reverted and inversely 

## Vulnerability Detail
We can see that `LibTWAPOracle::consult` computes the average price for uAD on the metapool vs 3CRV. However since it uses the duration since last update as a TWAP duration, it will always get only the value of price at the previous block it was updated at;
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L80

Let's consider the following example:

metapool initial state at block N:
reserveA: 1000
reserveB: 1000

metapool state at block N+1 (+12 seconds):
reserveA: 1500
reserveB: 500

if we have executed the update at each of these blocks, this means that if we consult the twap at block N+2,
we have:
ts.priceCumulativeLast: `[A, B]` 
priceCumulative: `[A+1500*12, B+500*12]` (reserve * time_elapsed); 
blockTimestamp - ts.pricesBlockTimestampLast = 12;

which means that when we call `get_twap_balances` the values returned are simply [1500, 500], which are the values of the previous block.

## Impact
A malicious user which can control two successive blocks (it is relatively feasible since the merge), can put the twap in any state for the next block:
- Can Dos any minting/burning in the block after the ones he controls
- Can unblock minting/burning for the next block, and depeg uAD further  

## Code Snippet

## Tool used

Manual Review

## Recommendation
Use a fixed duration for the TWAP:
```solidity
uint256[2] memory twapBalances = IMetaPool(ts.pool)
    .get_twap_balances(
        ts.priceCumulativeLast,
        priceCumulative,
        15 MINUTES
    );
```