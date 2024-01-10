Radiant Charcoal Horse

medium

# LibUbiquityPool::mintDollar/redeemDollar reliance on outdated TWAP oracle may be inefficient for preventing depeg

## Summary
The ubiquity pool used for minting/burning uAD relies on a twap oracle which can be outdated because the underlying metapool is not updated when calling the ubiquity pool. This would mean that minting/burning will be enabled based on an outdated state when it should have been reverted and inversely 

## Vulnerability Detail
We can see that LibTWAPOracle has an update function to keep its values up to date according to the underlying metapool:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L61-L102

And that this function is called when minting/burning uADs:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L344

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L416


But the function update is not called on the underlying metapool, so current values fetched for it may be stale:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L134-L136

## Impact
A malicious user can use this to mint/burn heavily in order to depeg the coin further

## Code Snippet

## Tool used

Manual Review

## Recommendation
Call the function:
```vyper
def remove_liquidity(
    _burn_amount: uint256,
    _min_amounts: uint256[N_COINS],
    _receiver: address = msg.sender
)
```

On the underlying metapool the twap is based on, with only zero values, to ensure that the values of the pool are up to date when consulted