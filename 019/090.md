Macho Heather Mammoth

medium

# AccessControlFacet doesn't have ability to set admin for the role

## Summary
AccessControlFacet doesn't have ability to set admin for the role.
## Vulnerability Detail
`AccessControlFacet` is created to allow protocol manage different roles. Facet extends `AccessControlInternal`, which has different methods and uses `LibAccessControl` library to store executed roles actions.

While `LibAccessControl` library [has `setRoleAdmin` function](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibAccessControl.sol#L140-L144), both `AccessControlFacet` and `AccessControlInternal` don't use it. And as result it's not possible to set admin for the role.

Because of that only default admin will be used as parent role and protocol will not be able to granulary manage their roles.
## Impact
Admin for the role can't be set.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Add function on the facet that will set admin for the role.