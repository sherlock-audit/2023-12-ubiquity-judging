Radiant Charcoal Horse

medium

# UbiquityPool::redeemDollar DoS on redeeming if USDT/USDC depegs

## Summary
uAD is algorithmically dependent on its accepted collateral, which are supposed to be `LUSD` and `DAI` at first. However there is a mechanism blocking withdrawal in case the TWAP price of uAD in the metapool `uAD/3CRV` is too high. Since 3CRV also contains USDT and USDC, in case any of USDC or USDT depegging, 3CRV price against uAD will be lower. Redeeming uAD may end up temporarily locked.

## Vulnerability Detail
We can see that calling `redeemDollar` reverts if the price of uAD is above a given threshold:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L418-L420

This means that an event which causes the price of 3CRV to be below the threshold such as USDT or USDC depegging will cause redeeming to revert effectiely causing a DOS on withdrawals. 

## Impact
Temporary DOS on withdrawal because of a depeg of USDT/USDC which should be unrelated.

## Code Snippet
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L418-L420

## Tool used

Manual Review

## Recommendation
Remove the reliance on 3CRV, create a tricrypto pool which contains collateral tokens