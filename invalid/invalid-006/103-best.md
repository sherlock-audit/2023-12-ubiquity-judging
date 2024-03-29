Soaring Hazel Koala

medium

# Pausing redemptions while user is waiting for for `redemptionDelayBlocks` can freeze user's funds for unknown period of time

## Summary
When user want to claim his collateral, he should first burn all his `uAD` tokens and wait for `redemptionDelayBlocks` blocks to pass before he is able to redeem it. The problem arrises from the check 
```solidity
        require(
            poolStorage.isRedeemPaused[collateralIndex] == false,
            "Redeeming is paused"
        );
```
## Vulnerability Detail
`redemptionDelayBlocks` is a safety mechanism to prevent users from using flashloan -> minting -> redeeming in one transaction. But the logic introduces more complexity, which can lead to problems in some cases. One case is if the protocol has reached ceiling and decide to pause mints/redemptions. If the same event happens right after user calling `redeemDollar`, the protocol will burn user's dollar and don't allow him to withdraw his collateral. As the protocol is based on governance, there would be unknown time that would passed if the redemptions are unpaused again. The protocol plan to pause redemptions for long/unknown period after a ceiling is reached. In case of user trying to redeem low in general, but high enough for single user amount (lets say $1 000) the action would be unnoticed and noone would take the action to unpause all redemption for that.  
## Impact
Locked end users funds
## Code Snippet
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L481-L484
## Tool used
Manual Review
## Recommendation
- One solution is to remove the `require` statement inside `collectRedemption`. This is the correct business logic handle pausing redemptions, so user, which have burned their uAD are able to redeem their collateral, even after redemption is paused
- Another solution is to check for `poolStorage.unclaimedPoolCollateral[collateralIndex]` before pausing redemptions for given collateralIndex:
```diff
        if (toggleIndex == 0)
            poolStorage.isMintPaused[collateralIndex] = !poolStorage
                .isMintPaused[collateralIndex];
        else if (toggleIndex == 1)
            poolStorage.isRedeemPaused[collateralIndex] = !poolStorage
                .isRedeemPaused[collateralIndex];
+        if (poolStorage.isRedeemPaused[collateralIndex]) {
+           if (poolStorage.unclaimedPoolCollateral[collateralIndex] > 0)
+                revert;
        } else if (toggleIndex == 2)
            poolStorage.isBorrowPaused[collateralIndex] = !poolStorage
                .isBorrowPaused[collateralIndex];
```