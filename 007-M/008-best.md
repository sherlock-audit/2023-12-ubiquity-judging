Dancing Vinyl Sawfish

medium

# `UbiquityDollarToken` allows users to transfer tokens to themselves and receive incentives both as sender and recipient

## Summary

`UbiquityDollarToken` allows users to transfer tokens to themselves and receive incentives both as sender and recipient.

## Vulnerability Detail

`UbiquityDollarToken` transfers applies optional incentives to the sender, recipient and operator. This is performed in the `_checkAndApplyIncentives` internal function.

While there is a check to ensure that the operator is a different address than the sender and recipient, there is no check to ensure that the sender and recipient are different addresses. This means that if the sender and recipient are the same address, the incentives will be applied twice to the same address. 

While the transfer does not have any impact on the balance of the user, he will receive incentives both as sender and recipient.

## Impact

Users can transfer tokens to themselves and receive extra incentives without any impact on their balance.

## Code Snippet

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/core/UbiquityDollarToken.sol#L84-L136

## Tool used

Manual Review

## Recommendation

```diff
    function _transfer(
        address sender,
        address recipient,
        uint256 amount
    ) internal override {
        super._transfer(sender, recipient, amount);
-       _checkAndApplyIncentives(sender, recipient, amount);
+       if (sender != recipient) {
+           _checkAndApplyIncentives(sender, recipient, amount);
+       }
    }
```
