Radiant Charcoal Horse

high

# LibTWAPOracle::update Providing large liquidity will manipulate TWAP, DOSing redeem of uADs

## Summary
The twap oracle used by ubiquity does not compute a TWAP (Time weighted average price) but an instant price based on time weighted average of balances, and thus is more vulnerable to manipulation by an actor temporarily allocating a large amount to the metapool.

## Vulnerability Detail
We can see the `LibTWAPOracle::update()` calls on 
`IMetapool::get_twap_balances(uint256[2] memory _first_balances, uint256[2] memory _last_balances, uint256 _time_elapsed)` which gets the time-weighted average of the pool reserves given time_elapsed and cumulative balances.

Then the time-weighted average is deduced by using the average of reserves over the `time_elapsed`. However we can see that a user with a large amount of capital and controlling 2 consecutive blocks can either add large liquidity in an imbalanced way in block N, and remove in block N+1.
In that case the `higher` reserves ratio will have a higher weight in the computation and will last for many blocks.

### Scenario
1/ In block N-1:

Metapool contains 10 of uAD and 10 of 3CRV, the price is balanced, and withdrawals of uAD are accepted.

2/ In block N: 

Alice updates both metapool twap and ubiquityPool twap. 

Alice provides 100 uAD and 200 3CRV to the pool, now reserves are 110 uAD and 210 3CRV

3/ In block N+1:

Alice removes the previously added liquidity putting the reserves back to 10 uAD and 10 3CRV, but does not updates UbiquityPool right away.

4/ In block N+40:

Alice updates both Metapool and ubiquity TWAP oracle, 

Let's see the value which is computed for the returned TWAP: 

uAD cumulative reserve difference = 12\*110+12\*40\*10 = 6120
3CRV cumulative reserve difference = 12\*210+12\*40\*10 = 7320

We can see that the reserves are still very imbalanced even 40 blocks after the 2 blocks manipulation, blocking withdrawals, because uAD price versus 3CRV is too high:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L418-L421

## Impact
A malicious user with access to considerable capital and controlling two consecutive blocks (feasible since the merge), can DOS the withdraw functionality of UbiquityPool for many blocks.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Find a way to compute the twap of the price (as Uniswap v3 does), instead of using twap of balances as a proxy 