Suave Wintergreen Donkey

medium

# PoolCeiling invariant can easily be broken due to use of `balanceOf`, allowing minting when the total collateral deposited is greater than the poolceiling.

## Summary

The protocol defines a poolCeiling for each type of collateral and implements a require check in mintDollar to make sure that the function reverts if adding the collateral would surpasses the poolCeiling. 

However, since the `balanceOf` used in the calculation can be temporarily reduced due to regular AMO activity, it is extremely likely that the poolceiling invariant will be broken. 

NOTE TO JUDGES:  I would like to argue that this finding should not be considered as a duplicate of findings related to the absence of limits in the amoMinterBorrow function. The root of this finding lies in the use balancOf and will occur even with extremely small uses of the amoMinterBorrow function (1-10%). Since the root and effects are different, this should a separate finding. 

## Vulnerability Detail

Example:

poolCeiling = 1000
Contract balance = 925

Alice tries to mint with 100 collateral: 
`mintDollar`: check 925 + 100  >  1000 =>  reverts with "Pool ceiling" message

NOW However: AMO borrows 50 through `amoMinterBorrow` 
Contract balance = 925 - 50 = 875

Bob tries to mint with 100 collateral:  
`mintDollar`check 875 + 100 < 1000 => check passes and 100 collateral is added. 
Contract balance = 875 + 100 = 975
 
AMO returns borrow of 50
Contract balance = 975 +50 = 1025 **poolCeiling** is broken 


## Impact

The protocol has defined the poolCeiling invariant in order to limit potential losses of funds in case of issues with the code. If this invariant can be broken, it would mean additional potential lost funds.  

## Code Snippet

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L268-L276

## Tool used

Manual Review

## Recommendation

The root of this issue lies in the use of `balanceOf`. Taking the current balance of a collateral as a representation of all deposits is prone to error since it is influenced by many factors.

Instead, it would be better to use a `mapping (address => uint256) private collateralBalance` which tracks the total deposits for a type of collateral and which is incremented in mintDollar and decremented in redeemDollar. In this way, it cannot be influenced by amo borrows, nor by accumulated fees or direct transfers.  