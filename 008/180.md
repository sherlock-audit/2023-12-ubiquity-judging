Acrobatic Boysenberry Kitten

medium

# Increase in the Redemption Delay Blocks postpones the redemption of the user beyond necessary

## Summary
if a user initiated a withdrawal before a change, preferrably, an increase in redemption delay block, he will be subjected to the new value not the one in use when he initiated withdrawal causing his redemption to be delay a couple more blocks.

## Vulnerability Detail
Ubiquity uses a push-pull method for Ubiquity Dollar Token(UDT) Redemption. Therefore, when a user chooses to redeem his UDT for collateral, he first needs to call `LibUbiquityPool::redeemDollar()` to push hsi request into the redemption pile. By calling this function, the lastRedeemBlock for the user is recorded to be the current block number.

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
        //All of the other preceeding code will be here

        poolStorage.lastRedeemedBlock[msg.sender] = block.number;

        IERC20Ubiquity ubiquityDollarToken = IERC20Ubiquity(
            LibAppStorage.appStorage().dollarTokenAddress
        );

        ubiquityDollarToken.burnFrom(msg.sender, dollarAmount);
    }
```

Now that he has initiated the redemption process, he needs to call `LibUbiquityPool::collectRedemption()` for the actual token transfer to happen. For this to happen, at least the redemption delay blocks needs to have passed since the last redeem Block.

```solidity
    require(
            (
                poolStorage.lastRedeemedBlock[msg.sender].add(
                    poolStorage.redemptionDelayBlocks
                )
            ) <= block.number,
            "Too soon to collect redemption"
        );
```

The problem with this system is that if a user initiated a withdrawal before a change, preferrably, an increase in redemption delay block, he will be subjected to the new value not the one in use when he initiated withdrawal.

## Impact
The severity of this vulnerability is MEDIUM. The impact of this is low since the user is delayed a couple more blocks before he can received his collateral which he redeemed his UDT for however, the guarantee that this will happen when the redemption delay blocks is changed is 100%, hence, this likelihood of this occuring is HIGH

## Code Snippet
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L458

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L485

## Tool used

Manual Review

## Recommendation
Use a mapping to track when the user can collect his collateral. This mappings takes in the lastRedeemBlock + current redemption delay blocks and the user will only be able to withdraw at that time. This way, when the redemption delay block changes, only new redemption will be subjected to that current value not users who have initiated a withdrawal before.