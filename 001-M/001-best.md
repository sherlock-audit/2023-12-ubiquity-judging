Mythical Oily Goldfish

medium

# AmoMinter can borrow collateral more than what's free/available

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