Damp Fossilized Canary

medium

# Invalid AMO minter might be added by LibUbiquityPool.addAmoMinter()

## Summary
Invalid AMO minter might be added by LibUbiquityPool.addAmoMinter().

## Vulnerability Detail
UbiquityPool.addAmoMinter() calls IDollarAmoMinter(amoMinterAddress).collateralDollarBalance() to ensure that the collateralDollarBalance is greater than zero. 

However, the following line is always successful: 

```javascript
require(collatValE18 >= 0, "Invalid AMO");
```
Thus the require statement is useless. That also means, even when the collateralDollarBalance is zero, the require statement will still pass and the invalid AMO minter will still be added. 


## Impact
Invalid AMO minter might be added by LibUbiquityPool.addAmoMinter().

## Code Snippet
[https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L608-L621](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L608-L621)

## Tool used
VSCode
Manual Review

## Recommendation
Change the require statement to make sure ``collatValE18 > 0``.