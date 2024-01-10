Fit Tawny Crow

medium

# Mint and Redeem function don't have the functionality of adding deadline check.

## Summary
The mint- redeem- trade transaction lacks of expiration timestamp check (DeadLine check)
## Vulnerability Detail
`mintDollar()` function and `redeemDollar()` function have the following signature :

```soldity
    function mintDollar(
        uint256 collateralIndex,
        uint256 dollarAmount,
        uint256 dollarOutMin,
        uint256 maxCollateralIn
    )
    function redeemDollar(
        uint256 collateralIndex,
        uint256 dollarAmount,
        uint256 collateralOutMin
    )
```

As we can see there is no deadline check option.  The deadline check ensures that the transaction can be executed on time and the expired transaction revert.

## Impact
The transaction can be pending in mempool for a long and the trading activity is very time sensitive. Without a deadline check, the trade transaction can be executed for a long time after the user submits the transaction, at that time, the trade can be done at a sub-optimal price, which harms the user's position.

The deadline check ensures that the transaction can be executed on time and the expired transaction revert.

As we have seen in the past stable coins can depeg, like usdc and ust, so the risk is there for DAI and LUSD too. 
## Code Snippet
https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L326-L465
## Tool used
cAtS


## Recommendation
Add option to add a deadline check. 