Swift Rose Spider

medium

# `mintingFee` and `redemptionFee` are not used and are not withdrawable causing protocol to miss profit off it

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