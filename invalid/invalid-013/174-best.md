Sticky Wintergreen Peacock

medium

# Pool ceiling check does not take into account AMO

## Summary

When minting Ubiquity dollar, there is a check to avoid minting too many dollar based on a single collateral. This verifies that after minting the collateral held by the UbiquityPool minus the collateral pending user withdrawal does not exceed the ceiling limit. However, it does not take into account the collateral that has been withdrawn by AMOs. As such, there is no real limit to the amount of collateral the Ubiquity dollar may rely on for any collateral if they are processed by AMOs.

## Vulnerability Detail

The `mintDollar()` function checks the following:

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
        ...
        
        collateralNeeded = getDollarInCollateral(collateralIndex, dollarAmount);
        
        ...
        
        // check the pool ceiling
        require(
            freeCollateralBalance(collateralIndex).add(collateralNeeded) <=
                poolStorage.poolCeilings[collateralIndex],
            "Pool ceiling"
        );
        ...
    }
```

The `freeCollateralBalance()` value is calculated as:

```solidity
    function freeCollateralBalance(
        uint256 collateralIndex
    ) internal view returns (uint256) {
        UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();
        return
            IERC20(poolStorage.collateralAddresses[collateralIndex])
                .balanceOf(address(this))
                .sub(poolStorage.unclaimedPoolCollateral[collateralIndex]);
    }
```

We can see that the check relies on the current token balance of the contract and does not take into account the amount withdrawn by the AMOs.

## Impact

Ubiquity dollar could have higher reliance on a single collateral token than anticipated as the ceiling values do not properly take into account the amount of Ubiquity dollar minted from one collateral. This can destabilize the Ubiquity dollar price in case the price of the underlying collateral fluctuates.

## Code Snippet

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L326-L386

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L268-L276

## Tool used

Manual Review

## Recommendation

Calculate how much collateral has been deposited to mint Ubiquity dollar for each token and enforce that value to be lower than the ceiling instead of relying on the current balance of the contract.