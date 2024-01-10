Big Crepe Jaguar

medium

# attacker can bypass function whitelist in Diamond pattern and call fallback() function because code doesn't check data.length>=4

## Summary
Ubiquity protocol implements diamond proxy. Admins can set (address, function) whitelist in the diamond proxy that are allowed to be called in the proxy. the issue is that `fallback()` function in Diamond contract doesn't check that `data.length >= 4` so if any whitelisted function's signature ends with 0 byte, then attacker can call fallback() function of that function's facet, while admins doesn't whitelisted the fallback function.

## Vulnerability Detail
This is the `fallback()` code in Diamond contract, as you can see code doesn't check to make sure `data.length >= 4`:
```javascript
    fallback() external payable {
        LibDiamond.DiamondStorage storage ds;
        bytes32 position = LibDiamond.DIAMOND_STORAGE_POSITION;
        assembly {
            ds.slot := position
        }
        address facet = ds.selectorToFacetAndPosition[msg.sig].facetAddress;
        require(facet != address(0), "Diamond: Function does not exist");
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
        }
    }
```
the issue is that if `data.length = 3` then solidity pads it with 0 zeros when using `msg.sig` value, so if there was a whitelisted function in the diamond proxy that its signature ends 0 then attacker can call diamond proxy without that ending zero, and fallback code would bypass the facet check but in the target facet instead of the target function, the fallback would be executed. this is the POC:
1. admins sets function `evaSafesFactory()` in FACET1 as whitelisted function+facet and the function signature is `0x12d05b00`.
2. now attacker can call diamond proxy with `data = 0x12d05b`.
3. `fallback()` function in the diamond proxy would use `msg.sig` to find the facet related to the function signature.
4. solidity would pad the `0x12d05b` with zeros and `msg.sig =  0x12d05b00` and code would validate FACET1 as target facet.
5. then code would delegate call FACET1 address with `data =  0x12d05b` and in that target address's code the fallback would be executed.

I have a coded POC that can be tested in the remix IDE, if you call this contract with `calldata = 0x12d05b` then the `fallback()` will be executed and the value of the `sig_fallback` would be set as `0x12d05b00` which proves that `msg.sig` pads with 0 if `data.length < 4` and also proves that if `data.length < 4` then fallback will be executed.
```javascript
pragma solidity ^0.8.19;

contract PoC {
    bytes4 public sig_function = 0x11223344;
    bytes4 public sig_fallbak = 0x11223344;
    bytes32 public test;

    function evaSafesFactory() external {
        sig_function = msg.sig;
    }

    fallback(bytes calldata data) external payable returns(bytes memory a) {
        sig_fallbak = msg.sig;
        return data;
    }
}
```

## Impact
in some cases attacker can bypass facet+function whitelists in the diamond proxy and call the `fallback()` functions of the facets.

## Code Snippet
https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/Diamond.sol#L46-L54

## Tool used
Manual Review

## Recommendation
in `fallback()` of the diamond check that `data.length >= 4`