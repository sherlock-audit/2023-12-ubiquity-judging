Radiant Charcoal Horse

medium

# UbiquityPool::redeemDollar No guarantees of pool solvency

## Summary
The ubiquity pool relies on chainlink feeds for minting/redeeming but as the price fluctuates, there is no guarantee that there is enough collateral in the pool to serve all redeems. That can cause a bank run in which the last user may not be able to redeem.

## Vulnerability Detail

### Scenario
Only DAI is used as a collateral in the pool

Alice and Bob deposit 10 DAI when price of uAD against DAI is 0.9, so they both get 11 uAD. Collateral in the pool is 20 DAI.

Chainlink feed now returns 1.1 for the same pair.

Alice can withdraw 12 DAI by redeeming all of her uAD. Bob should be able to redeem 12 DAI as well, but there is only 8 DAI left in the pool.

## Impact
Last users may be unable to redeem full value of their uAD, and thus lose funds. 

## Code Snippet
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L284-L294

## Tool used

Manual Review

## Recommendation
Socialize `losses` when chainlink feed returns a price which may not allow all users to redeem.