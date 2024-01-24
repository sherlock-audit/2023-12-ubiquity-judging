# Issue H-1: UbiquityPool::redeemDollar DoS on redeeming if USDT/USDC depegs 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/59 

## Found by 
0xpiken, Arz, GatewayGuardians, KingNFT, cergyk, evmboi32, fugazzi, ilchovski, shaka
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The issue describes about the protocol insolvancy in case of collateral depeg. It's not avoidable, that's why the protocol has borrowing function to get yield, take fees on mint and redeem, these features will hedge the risk from protocol insolvancy



**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The issue describes about the protocol insolvancy in case of collateral depeg. It's not avoidable, that's why the protocol has borrowing function to get yield, take fees on mint and redeem, these features will hedge the risk from protocol insolvancy



**nevillehuang**

See also #61 for a coded PoC, both talking about UAD price reliant on 3CRV

**rndquu**

Ok, we've already discussed it [here](https://github.com/ubiquity/ubiquity-dollar/issues/259) so the issue seems to be valid and "will fix".

Not sure about the possible solution, but the following seem worth to check:
- https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/30
- https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/61
- https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/141
- https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/165
- https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/192


# Issue H-2: UbiquityPool::mintDollar/redeemDollar Sandwich chainlink oracle update can enable riskless arbitrage on uAD 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/72 

## Found by 
Bauer, Coinstein, FastTiger, almurhasan, cergyk, fugazzi
## Summary
When chainlink updates the oracle price, a malicious actor can use own funds to sandwich the transaction and profit at the expense of the collateral reserve of the protocol

## Vulnerability Detail

We can see that the `LibUbiquityPool::update` function updates the price of a collateral, but can do so multiple times in one block:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L519-L562

Then we can deduce that calling this function before and after the underlying chainlink update inside one block will set the price two times, and enable the sandwicher to redeem more collateral than deposited, effectively risklessly extracting collateral from the protocol  

### Scenario

Chainlink aggregator update transaction is broadcasted to the mempool, initial price is 0.98 and will become 1.02 after the transaction. 

Alice sandwiches the transaction:

- Front-run by calling `UbiquityPool::mintDollar` for 1000 uAD by providing 980 DAI
- Back-run by calling `UbiquityPool::redeemDollar` for 1000 uAD and withdrawing 1020 DAI

Note: the actual withdraw of collateral is delayed, but this is not preventing this vector of attack

## Impact
Some collateral may be extracted from the protocol at each oracle update

## Code Snippet

## Tool used
Manual Review

## Recommendation
Add a delay mechanism to prevent redeeming in the same block as minting, or use current delay mechanism but adapt it to ensure that the price at time of collecting is used



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> TWAP price does not change in same block



**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> TWAP price does not change in same block



**rndquu**

For me this is normal arbitrage. If chainlink reports the `DAI` token to equal `$1.02` then there should be less `DAI` tokens than `Dollar` tokens in the end which is basically the backrun part of the sandwich bundle described.

If you take the same example described in the original issue but without the sandwich frontrun and backrun transactions you get the same logic:
1. Chainlink updates DAI to `$1.02` (it means that the Dollar token price should be <1$ over time)
2. Some other user redeems 1 Dollar token for 1 DAI worth `$1.02` pushing the Dollar token peg back to 1 USD

**rndquu**

I don't understand how exactly it harms the protocol, need a detailed example

**nevillehuang**

@CergyK Maybe you would want to comment and provide an example/PoC?

**molecula451**

@CergyK  please provide us with a reproducible PoC! with good price queries

**molecula451**

there is an hypothetical lack of sufficient liquidity to redeem by an honest actor (depositor) due to the nature of the arbitrage and the front run and external user could have taken the collateral, making thus the initial depositor would be unavailable to redeem his/her deposit, argument 0 fees

what do you think @rndquu @gitcoindev 

# Issue M-1: AmoMinter can borrow collateral more than what's free/available 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/1 

## Found by 
0xAlix2, 0xadrii, 0xpiken, 14si2o\_Flint, ArmedGoose, Aymen0909, Coinstein, DMoore, Drynooo, GatewayGuardians, KupiaSec, avoloder, b0g0, blutorque, brgltd, cats, cergyk, coffiasd, ilchovski, infect3d, nobody2018, r0ck3tz, rvierdiiev, ydlee, zraxx
## Summary

AmoMinters can borrow collateral from the protocol to earn yield in external protocols like Compound, and Curve, ... This can be done using the `amoMinterBorrow` function, which sends the "amount" of collateral to the targeted AmoMinter. However, this function doesn't check the validity of the borrowed amount, i.e. it doesn't check if the protocol has enough "free" amount before going ahead and sending the collateral.

## Vulnerability Detail

AmoMinter can borrow a collateral amount even if it is included in an unclaimed collateral storage. So if a user calls `redeemDollar` on certain collateral, and an AmoMinter borrows the full amount of collateral, that user won't be able to call `collectRedemption`, forcing him to lose his dollar tokens for nothing in return. As the transferred "borrowed" amount won't consider the unclaimed collateral `unclaimedPoolCollateral[collateralIndex]`.

```solidity
function test_bug_noFreeCollateralCheck() public {
    uint256 _100ETH = 100e18;

    vm.prank(admin);
    ubiquityPoolFacet.setPriceThresholds(1000000, 1000000);

    assertEq(collateralToken.balanceOf(address(ubiquityPoolFacet)), 0);

    vm.prank(user);
    ubiquityPoolFacet.mintDollar(0, _100ETH, 99e18, _100ETH);

    assertEq(
        collateralToken.balanceOf(address(ubiquityPoolFacet)),
        _100ETH
    );

    vm.prank(user);
    ubiquityPoolFacet.redeemDollar(0, 99e18, 90e18);

    assertEq(
        collateralToken.balanceOf(address(ubiquityPoolFacet)),
        _100ETH
    );
    assertEq(ubiquityPoolFacet.freeCollateralBalance(0), 2.98e18);

    vm.prank(address(dollarAmoMinter));
    ubiquityPoolFacet.amoMinterBorrow(_100ETH);

    vm.roll(3);

    vm.prank(user);
    vm.expectRevert("ERC20: transfer amount exceeds balance");
    ubiquityPoolFacet.collectRedemption(0);
}
```

## Impact

Loss of funds (dollar tokens).

## Code Snippet

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L574-L598

## Tool used

Manual Review + vscode

## Recommendation

Add the following `require` statement in the `amoMinterBorrow` function in the `LibUbiquityPool` library.
```solidity
require(
    collateralAmount <= freeCollateralBalance(minterCollateralIndex),
    "Not enough free collateral"
);
```



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**0xLogos** commented:
> amo minter is trusted, imho correctness of amo behavior is vital to protocol solvency and there's no way to safeguard against its bad behavior and keep its functionality because allowing him to withdraw all free collateral barely only protects pending withdrawls



**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**0xLogos** commented:
> amo minter is trusted, imho correctness of amo behavior is vital to protocol solvency and there's no way to safeguard against its bad behavior and keep its functionality because allowing him to withdraw all free collateral barely only protects pending withdrawls



**rndquu**

@gitcoindev @molecula451 

AMO minters are trusted so it's basically our job to check that AMO minters don't borrow more than available.

Anyway, this seems to be a quick fix one-liner (in the "recommendation" section) so I think we should fix it. 

**gitcoindev**

@rndquu I agree with your comment. Since we do not use AMO minters yet but can simulate and add the check. Adding Will Fix label.

# Issue M-2: LibUbiquityPool::mintDollar/redeemDollar reliance on outdated TWAP oracle may be inefficient for preventing depeg 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/13 

## Found by 
0xadrii, ArmedGoose, XDZIBEC, cergyk, osmanozdemir1, rvierdiiev, the-first-elder
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The issue describes about TWAP can be manipulated because `update` function can be called anytime and by anyone, thus TWAP period can be as short as 1 block. It seems like a valid issue but after caeful consideration, it's noticed that the TWAP issue does not come from its period but the logic itself is incorrect, thus marking this as Invalid



**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The issue describes about TWAP can be manipulated because `update` function can be called anytime and by anyone, thus TWAP period can be as short as 1 block. It seems like a valid issue but after caeful consideration, it's noticed that the TWAP issue does not come from its period but the logic itself is incorrect, thus marking this as Invalid



**gitcoindev**

> 1 comment(s) were left on this issue during the judging contest.
> 
> **auditsea** commented:
> 
> > The issue describes about TWAP can be manipulated because `update` function can be called anytime and by anyone, thus TWAP period can be as short as 1 block. It seems like a valid issue but after caeful consideration, it's noticed that the TWAP issue does not come from its period but the logic itself is incorrect, thus marking this as Invalid

Shouldn't the issue be marked with 'Sponsor disputed' label then if it is invalid? @rndquu @pavlovcik @molecula451  

**pavlovcik**

It probably makes more sense to ask @auditsea (not sure if this is the corresponding GitHub handle.)

**rndquu**

> A malicious user can use this to mint/burn heavily in order to depeg the coin further

Economically it doesn't make sense for a malicious user to do so. User is incentivised to arbitrage Dollar tokens hence even when metapool price is stale and a huge trade happens (i.e. a huge amount of Dollar tokens is minted in the Ubiquity pool) all secondary markets (uniswap, curve, etc...) will be on parity (in terms of Dollar-ANY_STABLECLON pair) after some time hence arbitrager will have to call `_update()` (when adding or removing liquidity) in the curve's metapool to make a profit.

Meanwhile I agree that the metapool's price can be stale but I don't understand how exactly minting "too many" Dollar tokens (keeping in mind that we get collateral in return) or burning "too many" Dollar tokens (keeping in mind that it reduces the `Dollar/USD` quote) may harm the protocol.

Anyway fresh data is better than stale one and the fix looks like a one liner so perhaps it makes sense to implement it. So [here](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L69) we could remove liquidity from the curve's metapool with 0 values (as mentioned in the "recommendation" section) which updates cumulative balances under the hood.

@gitcoindev @molecula451 What do you think?

**gitcoindev**

> > A malicious user can use this to mint/burn heavily in order to depeg the coin further
> 
> Economically it doesn't make sense for a malicious user to do so. User is incentivised to arbitrage Dollar tokens hence even when metapool price is stale and a huge trade happens (i.e. a huge amount of Dollar tokens is minted in the Ubiquity pool) all secondary markets (uniswap, curve, etc...) will be on parity (in terms of Dollar-ANY_STABLECLON pair) after some time hence arbitrager will have to call `_update()` (when adding or removing liquidity) in the curve's metapool to make a profit.
> 
> Meanwhile I agree that the metapool's price can be stale but I don't understand how exactly minting "too many" Dollar tokens (keeping in mind that we get collateral in return) or burning "too many" Dollar tokens (keeping in mind that it reduces the `Dollar/USD` quote) may harm the protocol.
> 
> Anyway fresh data is better than stale one and the fix looks like a one liner so perhaps it makes sense to implement it. So [here](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L69) we could remove liquidity from the curve's metapool with 0 values (as mentioned in the "recommendation" section) which updates cumulative balances under the hood.
> 
> @gitcoindev @molecula451 What do you think?

I have the same impression. Since this is easy to remediate with updating of cumulative balances I agree we could accept 
 and implement it there. 


# Issue M-3: LibTWAPOracle::setPool Dos on setPool by donating a few wei of 3CRV directly to the metapool 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/14 

## Found by 
0xadrii, ArmedGoose, Tendency, bareli, bitsurfer, cergyk, fugazzi, hancook, infect3d, nmirchev8, nobody2018, rvierdiiev, unforgiven
## Summary
`LibTWAPOracle::setPool` relies on a perfect match between the initial balance of uAD and 3CRV in the pool, which means that a malicious user can front-run the admin call to `setPool` and donate a few wei of 3CRV to the pool in order to make the call revert. 

## Vulnerability Detail
We can see the the admin function `LibTWAPOracle::setPool`, checks that the balances of uAD and 3CRV are perfectly equal:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L51

This means that a malicious user can front-run the call to setPool by donating a few wei of 3CRV directly to the metapool, in which case the balances will not match. The admin will not be able to set the pool as the function will revert

## Impact
The feature of migrating to a new pool will be DoSed temporarily, as long as the attacker keeps sending a few wei to the metapool contract

## Code Snippet

## Tool used
Manual Review

## Recommendation
Accept a slight imbalance between the balances of uAD and 3CRV (by a few dollars), in order to make this kind of DoS economically impractical for the attacker



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The issue describes about DOSing setPool function by manipulating the Curve pool, but it's assumed that the Curve pool deployment, LP deposit, and setPool will be handled in one tx using multicall structure



**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The issue describes about DOSing setPool function by manipulating the Curve pool, but it's assumed that the Curve pool deployment, LP deposit, and setPool will be handled in one tx using multicall structure



**rndquu**

@gitcoindev @molecula451 

The recommendation states to "Accept a slight imbalance between the balances of uAD and 3CRV" but this can also lead to potential DOS.

I think we can simply remove [this](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L51) line.

**gitcoindev**

> @gitcoindev @molecula451
> 
> The recommendation states to "Accept a slight imbalance between the balances of uAD and 3CRV" but this can also lead to potential DOS.
> 
> I think we can simply remove [this](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L51) line.

I did a research on GitHub and indeed other pool implementations just use 

`require(_reserve0 > 0 && _reserve1 > 0, "No reserves");`

also [some pools check for overflow](https://github.com/PatrickAlphaC/Web3Bugs/blob/00259ca226c5e470e520d3b4705b46aafe4a89ec/contracts/29/trident/contracts/pool/franchised/FranchisedHybridPool.sol#L271)

therefore since 2 lines above  [this](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L51) line we check for reserves it is safe to remove this line.






**rndquu**

> > @gitcoindev @molecula451
> > The recommendation states to "Accept a slight imbalance between the balances of uAD and 3CRV" but this can also lead to potential DOS.
> > I think we can simply remove [this](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L51) line.
> 
> I did a research on GitHub and indeed other pool implementations just use
> 
> `require(_reserve0 > 0 && _reserve1 > 0, "No reserves");`
> 
> also [some pools check for overflow](https://github.com/PatrickAlphaC/Web3Bugs/blob/00259ca226c5e470e520d3b4705b46aafe4a89ec/contracts/29/trident/contracts/pool/franchised/FranchisedHybridPool.sol#L271)
> 
> therefore since 2 lines above [this](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L51) line we check for reserves it is safe to remove this line.

[This](https://github.com/PatrickAlphaC/Web3Bugs/blob/00259ca226c5e470e520d3b4705b46aafe4a89ec/contracts/29/trident/contracts/pool/franchised/FranchisedHybridPool.sol#L271) overflow check is needed because of downcasting from `uint256` to `uint128` [here](https://github.com/PatrickAlphaC/Web3Bugs/blob/00259ca226c5e470e520d3b4705b46aafe4a89ec/contracts/29/trident/contracts/pool/franchised/FranchisedHybridPool.sol#L272). I guess this is not the case for us since we don't use `uint128` in the `LibUbiquityPool`.

# Issue M-4: UbiquityPool::mintDollar/redeemDollar collateral depeg will encourage using UbiquityPool to swap for better collateral 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/17 

## Found by 
cducrest-brainbot, cergyk, fugazzi, ge6a, shaka
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The issue describes about the protocol insolvancy in case of collateral depeg. It's not avoidable, that's why the protocol has borrowing function to get yield, take fees on mint and redeem, these features will hedge the risk from protocol insolvancy



**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The issue describes about the protocol insolvancy in case of collateral depeg. It's not avoidable, that's why the protocol has borrowing function to get yield, take fees on mint and redeem, these features will hedge the risk from protocol insolvancy



**gitcoindev**

> 1 comment(s) were left on this issue during the judging contest.
> 
> **auditsea** commented:
> 
> > The issue describes about the protocol insolvancy in case of collateral depeg. It's not avoidable, that's why the protocol has borrowing function to get yield, take fees on mint and redeem, these features will hedge the risk from protocol insolvancy

Should we also add 'Sponsor Disputed' label in this issue as well? @rndquu @pavlovcik @molecula451 

**pavlovcik**

It probably makes more sense to ask @auditsea (not sure if this is the corresponding GitHub handle.)

**rndquu**

As far as I understand this issue describes the following scenario:
1. User1 mints 100 Dollar tokens and provides 100 DAI collateral
2. User2 mints 100 Dollar tokens and provides 100 LUSD collateral
3. DAI depegs
4. User1 (who initially deposited DAI) redeems 100 Dollar tokens for 100 LUSD
5. User2 (who initially deposited LUSD) is left only with depegged DAI pool

If DAI depegs then the Dollar token will also depeg mainly because DAI is used as an underlying collateral in the `Dollar-3CRVLP` curve's metapool. We shouldn't limit users in redeeming (i.e. burning) Dollar tokens anyhow because Dollar redeems bring back the `Dollar/USD` quote to `$1.00` peg. So it seems to be fine that users should be able to burn Dollars until the pools are empty no matter where they initially deposited to because this is the only way to maintain the Dollar token's USD peg.

In case of a collateral depeg the only way to hedge Dollar depeg to some extent is to acquire fees and yield from AMO minters. This is not a 100% guarantee but I guess that if chainlink works fine (i.e. provides not too stale data) and we have 5% overcollateralization (from fees and yield) the Dollar token should not depeg too much.

> Force chainlink feeds to stay within acceptable thresholds for stable coins (revert operations on collateral if price is out of range)

I don't understand how reverting on minting and redeeming helps. Reverting in this case means acquiring bad debt since all operations are paused and abritragers are not able to bring the Dollar token back to the USD peg by burning Dollars which makes the Dollar token to depeg with greater force.

**pavlovcik**

@rndquu as a heads up we should only start with accepting `LUSD` and then maybe, eventually, add `sUSD` next in case of future liquidity issues. `DAI` isn't part of the plan yet. 

**rndquu**

Initially we plan to use only `LUSD` as collateral so there won't be the case with bad (i.e. depegging) and good collateral.

I think we should:
1. Create a separate issue for this one in our [repo](https://github.com/ubiquity/ubiquity-dollar/) and fix it later since it is not critical
2. Mark the current issue as valid and "won't fix"

@molecula451 @gitcoindev Help


# Issue M-5: LibUbiquityPool::mintDollar/redeemDollar reliance on arbitrarily short TWAP oracle may be inefficient for preventing depeg 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/20 

## Found by 
0xLogos, Arz, Coinstein, Krace, b0g0, cducrest-brainbot, cergyk, evmboi32, infect3d, nirohgo, rvierdiiev, the-first-elder
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The issue describes about TWAP can be manipulated because `update` function can be called anytime and by anyone, thus TWAP period can be as short as 1 block. It seems like a valid issue but after caeful consideration, it's noticed that the TWAP issue does not come from its period but the logic itself is incorrect, thus marking this as Invalid



**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The issue describes about TWAP can be manipulated because `update` function can be called anytime and by anyone, thus TWAP period can be as short as 1 block. It seems like a valid issue but after caeful consideration, it's noticed that the TWAP issue does not come from its period but the logic itself is incorrect, thus marking this as Invalid



**gitcoindev**

> 1 comment(s) were left on this issue during the judging contest.
> 
> **auditsea** commented:
> 
> > The issue describes about TWAP can be manipulated because `update` function can be called anytime and by anyone, thus TWAP period can be as short as 1 block. It seems like a valid issue but after caeful consideration, it's noticed that the TWAP issue does not come from its period but the logic itself is incorrect, thus marking this as Invalid

Also invalid then and 'Sponsor Disputed' label should be added? @rndquu @pavlovcik @molecula451 

**pavlovcik**

It probably makes more sense to ask @auditsea (not sure if this is the corresponding GitHub handle.)

**rndquu**

Yes, it is possible (especially in the early Dollar token stage when market activity is low) to skew the curve's TWAP value since our TWAP (in some cases) may simply take the latest block price thus TWAP can be manipulated with low effort.

This is not critical (i.e. medium severity seems to be valid) since TWAP's price is used only in `require()` statements on mint and redeem operations but must be fixed.

Not sure what's the best possible solution but the following similar issues are worth to check:
- https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/138
- https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/175



**gitcoindev**

@rndquu just to double check, the solution is to set a constant time window for the TWAP price. 

A question to Sherlock: should the window be set to 15, 30 min or any other value?

**pavlovcik**

In my experience with software development, anything time based is better implemented to be event based. Although unfortunately I'm not sure what events would make a great substitute in this case. What if we checked that they aren't consecutive blocks?

**rndquu**

> @rndquu just to double check, the solution is to set a constant time window for the TWAP price.
> 
> A question to Sherlock: should the window be set to 15, 30 min or any other value?

 I think the solution is to use the latest curve's metapool with built-in TWAP oracle and adjustable time window as described [here](https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/201). 
 
 > the solution is to set a constant time window for the TWAP price
 
 You're right, the solution is to increase it to 15 or 30 minutes.

**rndquu**

Since there is no direct loss of funds the "high" severity doesn't seem to be correct.

# Issue M-6: `mintingFee` and `redemptionFee` are not used and are not withdrawable causing protocol to miss profit off it 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/36 

## Found by 
0xnirlin, ArmedGoose, Tendency, cergyk, rvierdiiev
## Summary
In `UbiquityPoolFacet`, implemented in `LibUbiquityPool`, there are variables and some logic for collecting `mintingFee` and `redemptionFee`. However this logic is nonfunctional due to a fact, that these fees are not accounted anywhere and there is also no logic allowing for withdrawing / utilizing these fees. In fact, these fees just increase the contract's collateral balance, which causes the protocol to miss any profit that may come off the fees. 

## Vulnerability Detail
`UbiquityPoolFacet` allows for core operations of [minting](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L326) and [redeeming](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L399) Ubiquity dollars. Each of those operations includes a fee, which is deducted off the amount user inputs, for mint, the minted amount is decreased by `mintingFee`, which means that fee part of deposited collateral is received by the protocol without minting dollars in exchange, and in `redeem`, similarly, the amount redeemed is diminished by the fee amount, which also means, that fee part of the collateral stays in the contract instead of being fully returned for ubiquity dollars. There is also no withdraw() or similar function that could allow for extraction of the user fees - it simply stays on the contract.

As an effect, the protocol gains surplus collateral stored on the contract, but this fee is not utilized or even accounted, and thus protocol cannot benefit off it.

## Impact
The protocol misses the revenue from fees which is a financial loss.

## Code Snippet

```solidity
    function mintDollar(
        uint256 collateralIndex,
        uint256 dollarAmount,
        uint256 dollarOutMin,
        uint256 maxCollateralIn
    )
        internal
        collateralEnabled(collateralIndex)
        returns (uint256 totalDollarMint, uint256 collateralNeeded)
    {
        UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();

        require(
            poolStorage.isMintPaused[collateralIndex] == false,
            "Minting is paused"
        );

        // update Dollar price from Curve's Dollar Metapool
        LibTWAPOracle.update();
        // prevent unnecessary mints
        require(
            getDollarPriceUsd() >= poolStorage.mintPriceThreshold,
            "Dollar price too low"
        );

        // update collateral price
        updateChainLinkCollateralPrice(collateralIndex);

        // get amount of collateral for minting Dollars
        collateralNeeded = getDollarInCollateral(collateralIndex, dollarAmount);

        // subtract the minting fee
        totalDollarMint = dollarAmount
            .mul(
                UBIQUITY_POOL_PRICE_PRECISION.sub(
                    poolStorage.mintingFee[collateralIndex] //@audit nothing happens with the fee
                )
            )
            .div(UBIQUITY_POOL_PRICE_PRECISION);
[...]
```

```solidity
    function redeemDollar(
        uint256 collateralIndex,
        uint256 dollarAmount,
        uint256 collateralOutMin
    )
        internal
        collateralEnabled(collateralIndex)
        returns (uint256 collateralOut)
    {
        UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();

        require(
            poolStorage.isRedeemPaused[collateralIndex] == false,
            "Redeeming is paused"
        );

        // update Dollar price from Curve's Dollar Metapool
        LibTWAPOracle.update();
        // prevent unnecessary redemptions that could adversely affect the Dollar price
        require(
            getDollarPriceUsd() <= poolStorage.redeemPriceThreshold,
            "Dollar price too high"
        );

        uint256 dollarAfterFee = dollarAmount
            .mul(
                UBIQUITY_POOL_PRICE_PRECISION.sub(
                    poolStorage.redemptionFee[collateralIndex] //@audit nothing happens with the fee
                )
            )
            .div(UBIQUITY_POOL_PRICE_PRECISION);
[...]
```

## Tool used

Manual Review

## Recommendation
1) Implement a variable(s) that could track accumulated fees, after each deduction add the newly deducted fees to these variables for accountability and 2) consider implementing a logic to administer these fees e.g. send them to a treasury, allow withdrawal, etc. Make sure to treat them separately from the contract balance so they are not used for minting, borrowing, or lost in any other way.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The goal of fee taken for mint/redeem is to hedge the risk of price deviation



**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> The goal of fee taken for mint/redeem is to hedge the risk of price deviation



**rndquu**

@pavlovcik 

There are mint and redeem fees in the Ubiquity pool contract. Initially (as far as I remember) we planned to set those fees to 0. Right now there is no way for admin to withdraw mint and redeem fees so these fees serve (as some sherlock judge already stated) to "hedge the risk of price deviation" (i.e. locked in the pool). Should the admin be able to withdraw mint and redeem fees?

**pavlovcik**

> Should the admin be able to withdraw mint and redeem fees?

I hadn't thought through this before. Following the approach of all of our other contracts, I think that having the flexibility to do this makes sense. However, given that I am just now hearing/thinking about this, perhaps the fee profits are abstracted away by lowering how much the "protocol owes." 

A stablecoin is just a balance sheet of assets and liabilities. Assets means collateral in the bank, liabilities means how many Dollar tokens (think of these as just "I owe you" tokens to redeem $1.00 of underlying collateral.)

Example: if we burn 100% of the circulating supply of "I owe you" tokens, then we "profited" 100% of the collateral, as nobody (except for us) can redeem the collateral anymore. 

So I am not 100% up-to-speed on how the mint/redeem fees are implemented, but if they burn more Dollars, then technically we are making a profit. In which case, perhaps we don't need to implement "fee withdrawal". 

**rndquu**

> So I am not 100% up-to-speed on how the mint/redeem fees are implemented, but if they burn more Dollars, then technically we are making a profit. In which case, perhaps we don't need to implement "fee withdrawal".

Technically those fees are not burning anything in terms of `ERC20.burn()` but "burning" in terms of fees:
1. On `mintDollar()` user gets less Dollars for provided collateral with mint fee enabled
2. On `redeemDollar` user gets less collateral for provided Dollars with redeem fee enabled

So both those actions reduce the amount of "I owe you" tokens hence this is equal to "burning" them which lowers the `Dollar/USD` quote in the end.

Since the contracts are upgradeable and we don't plan to withdraw fees I think we should save time and mark this issue as valid and "won't fix".

@gitcoindev @molecula451 What do you guys think?

**gitcoindev**

> > So I am not 100% up-to-speed on how the mint/redeem fees are implemented, but if they burn more Dollars, then technically we are making a profit. In which case, perhaps we don't need to implement "fee withdrawal".
> 
> Technically those fees are not burning anything in terms of `ERC20.burn()` but "burning" in terms of fees:
> 
> 1. On `mintDollar()` user gets less Dollars for provided collateral with mint fee enabled
> 2. On `redeemDollar` user gets less collateral for provided Dollars with redeem fee enabled
> 
> So both those actions reduce the amount of "I owe you" tokens hence this is equal to "burning" them which lowers the `Dollar/USD` quote in the end.
> 
> Since the contracts are upgradeable and we don't plan to withdraw fees I think we should save time and mark this issue as valid and "won't fix".
> 
> @gitcoindev @molecula451 What do you guys think?

I am fine with that, let's mark as valid but won't fix.

# Issue M-7: LibTWAPOracle::update Providing large liquidity will manipulate TWAP, DOSing redeem of uADs 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/56 

## Found by 
GatewayGuardians, KupiaSec, cducrest-brainbot, cergyk, evmboi32
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



## Discussion

**gitcoindev**

@rndquu @pavlovcik @molecula451 what do you think about this one? Technically possible, but would likely require lots of staked ETH. I am wondering if this is just hypothetical and controlling 2 blocks would require funds of e.g. Coinbase size. If this is the case, then perhaps impact is high but probability is very low. There are currently ~900k ETH validators https://beaconcha.in/charts/validators with each staked at least 32 ETH. 

**pavlovcik**

I spent some time looking through our old issues and discussions but unfortunately I couldn't find my writeup on this. Basically as I recall, after The Merge there was a research report that proved how a two-consecutive-block TWAP attack could be performed with a realistic amount of funds. 

Unfortunately I don't remember the conclusion of how to mitigate.

**rndquu**

> uAD cumulative reserve difference = 12*110+12*40*10 = 6120
> 3CRV cumulative reserve difference = 12*210+12*40*10 = 7320

TWAP window here is ~8 minutes. If we take a 30 minutes TWAP window (suggested everywhere) we get `uAD cumulative reserves = 19320 & 3CRV cumulative reserves = 20520` which is much closer to a balanced pool.

Overall this seems to be a duplicate of https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/20 since increasing the TWAP window fixes the issue (to some extent).


**rndquu**

So, as far as I understand, the core issue is that the [curve metapool](https://etherscan.io/address/0x20955CB69Ae1515962177D164dfC9522feef567E#code) uses accumulated balances instead of accumulated spot prices hence the TWAP is not as accurate as it could be.

I agree that it could be improved but disagree with the "high" severity.

# Issue M-8: AccessControlFacet doesn't have ability to set admin for the role 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/90 

## Found by 
0xnirlin, Coinstein, rvierdiiev
## Summary
AccessControlFacet doesn't have ability to set admin for the role.
## Vulnerability Detail
`AccessControlFacet` is created to allow protocol manage different roles. Facet extends `AccessControlInternal`, which has different methods and uses `LibAccessControl` library to store executed roles actions.

While `LibAccessControl` library [has `setRoleAdmin` function](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibAccessControl.sol#L140-L144), both `AccessControlFacet` and `AccessControlInternal` don't use it. And as result it's not possible to set admin for the role.

Because of that only default admin will be used as parent role and protocol will not be able to granulary manage their roles.
## Impact
Admin for the role can't be set.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Add function on the facet that will set admin for the role.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> Owner is set as a default admin, it should work



**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> Owner is set as a default admin, it should work



**molecula451**

this looks like an invalid because the admin can [set new roles](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/facets/AccessControlFacet.sol#L20):

by "granting" its just the function its named grantRoles instead of set!

the submission claims that the Facet don't use it, the admin can indeed use it!

@rndquu @gitcoindev 

**gitcoindev**

Granting roles is tested at https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/test/diamond/facets/AccessControlFacet.t.sol , could we please get feedback from the Watson who submitted this issue directly or from the @auditsea perhaps something was misunderstood here?

**rndquu**

> this looks like an invalid because the admin can [set new roles](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/facets/AccessControlFacet.sol#L20):
> 
> by "granting" its just the function its named grantRoles instead of set!
> 
> the submission claims that the Facet don't use it, the admin can indeed use it!
> 
> @rndquu @gitcoindev

This issue is about setting a role admin for some other role, not about granting roles.

Here is the `setRoleAdmin()` [method](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibAccessControl.sol#L140-L144) which is missing in the [AccessControlFacet](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/facets/AccessControlFacet.sol) contract. Should we want to set some role to be admin for another role we would need to upgrade the contracts by adding the `setRoleAdmin()` method to facets.

I would consider this valid and "will fix".

**molecula451**

good catch, yeah we missed the setRoleAdmin external function on  the facet

**molecula451**

PR Fix Confirmation: https://github.com/ubiquity/ubiquity-dollar/pull/880#event-11541346796 by @gitcoindev 

# Issue M-9: LibTWAPOracle::update temporary redemption DOS if uAD/3CRV metapool has very low liquidity 

Source: https://github.com/sherlock-audit/2023-12-ubiquity-judging/issues/105 

## Found by 
cergyk
## Summary
If the metapool 3CRV/uAD has very low liquidity, the spot price may be deviated, which means that as a consequence the TWAP value computed is deviated, and when TWAP of uAD is too high, users withdrawals will revert. This causes a DOS on withdrawals until liquidity in the metapool becomes sufficient again

## Vulnerability Detail
We can see in `LibTWAPOracle::update`, the function `IMetapool::get_dy()` is used to retrieve the spot price on averaged balances. The amount in, for which to retrieve amount out is hardcoded to `1 ether`:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L84-L89

This means that if the liquidity in the pool is less than 0.9 ether of 3CRV, the value returned by `get_dy` will be less than 0.9 ether, and the `redeemThreshold` will not be verified:
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L418-L421

Temporarily blocking withdrawals

## Impact
Withdrawals are temporarily blocked until enough liquidity is added to the metapool

## Code Snippet

## Tool used

Manual Review

## Recommendation
Use an adaptive parameter for amount in for this estimation, or do not update if the liquidity is too low



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> get_dy is called with TWAP balances, does not depend on actual balance



**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**auditsea** commented:
> get_dy is called with TWAP balances, does not depend on actual balance



**rndquu**

So if there is 0.9 Dollars and 0.9 3CRV-LP tokens in the pool then `IMetapool::get_dy()` returns 0.9 which is meant to be the `Dollar/USD` quote. The 0.9 value is still less than [redemption threshold](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L418-L421) so redemption should not be blocked unless I'm missing smth. 

**rndquu**

As stated by the lead senior watson: "I made a mistake, the redeem function is indeed not Dosed, the mint function can be DoSed until liquidity is sufficient. I think it's an issue because there are conditions blocking the mint/redeem based on a pool which has no overlap with the accepted collaterals. So this means that there is virtually no mechanism to incentivise providing liquidity to this pool or arbitraging it properly.".

Overall if liquidity is less than `1 ether` then [this line](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L84-L89) reports an incorrect value.

This seems to be a quick-fix so it makes sense to handle this case.

@gitcoindev @molecula451 What do you think?

**gitcoindev**

> As stated by the lead senior watson: "I made a mistake, the redeem function is indeed not Dosed, the mint function can be DoSed until liquidity is sufficient. I think it's an issue because there are conditions blocking the mint/redeem based on a pool which has no overlap with the accepted collaterals. So this means that there is virtually no mechanism to incentivise providing liquidity to this pool or arbitraging it properly.".
> 
> Overall if liquidity is less than `1 ether` then [this line](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibTWAPOracle.sol#L84-L89) reports an incorrect value.
> 
> This seems to be a quick-fix so it makes sense to handle this case.
> 
> @gitcoindev @molecula451 What do you think?

Hi @rndquu yes, in my opinion it seems reasonable to handle this case. 

