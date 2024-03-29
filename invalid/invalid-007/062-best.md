Salty Saffron Pheasant

high

# ChainLink price feed address for collateral with index 0 might be updated mistakenly

## Summary
ChainLink price feed address for collateral with index 0 might be updated mistakenly
## Vulnerability Detail
[`UbiquityPoolFacet#setCollateralChainLinkPriceFeed()`](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/facets/UbiquityPoolFacet.sol#L155-L165) is used to update chainLink price feed address and its threshold as well.
When [`LibUbiquityPool#setCollateralChainLinkPriceFeed()`](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L700-L726) is called, first it will [read the collateral index](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L707-L709), then update the chainLink price feed address and threshold based on the index:
```solidity
700:    function setCollateralChainLinkPriceFeed(
701:        address collateralAddress,
702:        address chainLinkPriceFeedAddress,
703:        uint256 stalenessThreshold
704:    ) internal {
705:        UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();
706:
707:        uint256 collateralIndex = poolStorage.collateralIndex[
708:            collateralAddress
709:        ];
710:
711:        // set price feed address
712:        poolStorage.collateralPriceFeedAddresses[
713:            collateralIndex
714:        ] = chainLinkPriceFeedAddress;
715:
716:        // set staleness threshold in seconds when chainlink answer should be considered stale
717:        poolStorage.collateralPriceFeedStalenessThresholds[
718:            collateralIndex
719:        ] = stalenessThreshold;
720:
721:        emit CollateralPriceFeedSet(
722:            collateralIndex,
723:            chainLinkPriceFeedAddress,
724:            stalenessThreshold
725:        );
726:    }
```
However, `collateralIndex` will be 0 for any collateral address which has not been supported yet. Therefore the chainlink price feed address and threshold of collateral stored in index 0 will be accidentally updated. 

Copy below codes into [UbiquityPoolFacetTest.t.sol](https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/test/diamond/facets/UbiquityPoolFacet.t.sol) and run `forge test --match-test testSetCollateralChainLinkPriceFeed_UpdateToInvalidFeedAddress()`:
```solidity
    function testSetCollateralChainLinkPriceFeed_UpdateToInvalidFeedAddress() public {
        vm.startPrank(admin);

        LibUbiquityPool.CollateralInformation memory info = ubiquityPoolFacet
            .collateralInformation(address(collateralToken));
        assertEq(
            info.collateralPriceFeedAddress,
            address(collateralTokenPriceFeed)
        );
        assertEq(info.collateralPriceFeedStalenessThreshold, 1 days);

        address newPriceFeedAddress = address(1);
        uint256 newStalenessThreshold = 2 days;
        vm.expectEmit(address(ubiquityPoolFacet));
        emit CollateralPriceFeedSet(
            0,
            newPriceFeedAddress,
            newStalenessThreshold
        );
        ubiquityPoolFacet.setCollateralChainLinkPriceFeed(
            address(collateralToken),
            newPriceFeedAddress,
            newStalenessThreshold
        );

        info = ubiquityPoolFacet.collateralInformation(
            address(collateralToken)
        );
        assertEq(info.collateralPriceFeedAddress, newPriceFeedAddress);
        assertEq(
            info.collateralPriceFeedStalenessThreshold,
            newStalenessThreshold
        );
        //@audit-info update price feed address for unsupported collateral
        ubiquityPoolFacet.setCollateralChainLinkPriceFeed(
            address(0x01),
            address(0x01),
            1
        );

        info = ubiquityPoolFacet.collateralInformation(
            address(collateralToken)
        );
        //@audit-info however, the price feed address and threshold of `collateralToken` was updated without any warning/reminding
        assertEq(info.collateralPriceFeedAddress, address(0x01));
        assertEq(
            info.collateralPriceFeedStalenessThreshold,
            1
        );
        vm.stopPrank();
    }
```
## Impact
Because `mintDallor()` and `redeemDollar()` completely dependent on price feed address to obtain collateral price, a malicious user might exploit the system by over minting uAD or redeeming collateral, potentially resulting in profit for themself.
## Code Snippet
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/facets/UbiquityPoolFacet.sol#L155-L165
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L700-L726
## Tool used

Manual Review

## Recommendation
Use `collateralIndex` as parameter directly instead of `collateralAddress` in `setCollateralChainLinkPriceFeed()`:
```diff
    function setCollateralChainLinkPriceFeed(
-       address collateralAddress,
+       uint256 collateralIndex,
        address chainLinkPriceFeedAddress,
        uint256 stalenessThreshold
    ) internal {
        UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();

-       uint256 collateralIndex = poolStorage.collateralIndex[
-           collateralAddress
-       ];

        // set price feed address
        poolStorage.collateralPriceFeedAddresses[
            collateralIndex
        ] = chainLinkPriceFeedAddress;

        // set staleness threshold in seconds when chainlink answer should be considered stale
        poolStorage.collateralPriceFeedStalenessThresholds[
            collateralIndex
        ] = stalenessThreshold;

        emit CollateralPriceFeedSet(
            collateralIndex,
            chainLinkPriceFeedAddress,
            stalenessThreshold
        );
    }
```