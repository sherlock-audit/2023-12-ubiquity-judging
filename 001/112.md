Flaky Cherry Poodle

medium

# AmoMinters can borrow collaterals without collateralDollar

## Summary
According to the logic of protocol, to borrow collaterals from protocol AmoMinters should have non zero collateralDollarBalance but AmoMinters can borrow collaterals without collateralDollar.
## Vulnerability Detail
When adding AmoMinters to the protocol using [addAmoMinter](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L608-L621) function of [LibUbiquityPool](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L21) library, there is a check if the AmoMinter has non zero collateralDollarBalance at [L612-614](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L612-L614).
But in the [amoMinterBorrow](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L574-L598) function of [LibUbiquityPool](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L21) library, there is no check if the AmoMinter has non zero collateralDollarBalance.
## Impact
AmoMinters can borrow collaterals without providing collateralDollar and I think this is against protocols logic.
## Code Snippet
```javascript
File: packages/contracts/src/dollar/libraries/LibUbiquityPool.sol
function amoMinterBorrow(uint256 collateralAmount) internal onlyAmoMinter {
    UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();

    // checks the collateral index of the minter as an additional safety check
    uint256 minterCollateralIndex = IDollarAmoMinter(msg.sender)
        .collateralIndex();

    // checks to see if borrowing is paused
    require(
        poolStorage.isBorrowPaused[minterCollateralIndex] == false,
        "Borrowing is paused"
    );

    // ensure collateral is enabled
    require(
        poolStorage.isCollateralEnabled[
            poolStorage.collateralAddresses[minterCollateralIndex]
        ],
        "Collateral disabled"
    );

    // transfer
    IERC20(poolStorage.collateralAddresses[minterCollateralIndex])
        .safeTransfer(msg.sender, collateralAmount);
}
```
[L574-589](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L574-L598)

## Tool used

Manual Review

## Recommendation

Add check for AmoMinter's collateralDollarBalance like the one in the [addAmoMinter](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L612-L614) function.

```diff
File: packages/contracts/src/dollar/libraries/LibUbiquityPool.sol
function amoMinterBorrow(uint256 collateralAmount) internal onlyAmoMinter {
    UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();

    // checks the collateral index of the minter as an additional safety check
    uint256 minterCollateralIndex = IDollarAmoMinter(msg.sender)
        .collateralIndex();

+   uint256 collatValE18 = IDollarAmoMinter(amoMinterAddress)
+           .collateralDollarBalance();
+   require(collatValE18 >= 0, "Invalid AMO");

    // checks to see if borrowing is paused
    require(
        poolStorage.isBorrowPaused[minterCollateralIndex] == false,
        "Borrowing is paused"
    );

    // ensure collateral is enabled
    require(
        poolStorage.isCollateralEnabled[
            poolStorage.collateralAddresses[minterCollateralIndex]
        ],
        "Collateral disabled"
    );

    // transfer
    IERC20(poolStorage.collateralAddresses[minterCollateralIndex])
        .safeTransfer(msg.sender, collateralAmount);
}
```