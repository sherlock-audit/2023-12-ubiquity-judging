Radiant Charcoal Horse

medium

# UbiquityPool::mintDollar/redeemDollar Sandwich chainlink oracle update can enable riskless arbitrage on uAD

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