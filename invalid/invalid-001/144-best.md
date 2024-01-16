Muscular Sapphire Rook

medium

# Oracle price boundaries not checked

## Summary
Users cannot burn their dollar tokens for collateral if collateral depegs or oracle reports an incorrect price.

This happened just last month when the chainlink reported an incorrect price
https://twitter.com/spreekaway/status/1733358874269249578

- `USDC` depegged to `~$0.83` last year
- `USDT` depegged a few times
- `DAI` also fell during the `USDC` depeg
- `LUSD` is generally quite volatile

The depegging of the stable coin is rare but it happens. But when it does it can cause unforeseen consequences.
## Vulnerability Detail
When updating the oracle price for collateral it is not checked if the price is within expected bounds
```solidity
function updateChainLinkCollateralPrice(uint256 collateralIndex) internal {
    ...
    // fetch latest price
    (
        ,
        // roundId
        int256 answer, // startedAt
        ,
        uint256 updatedAt,

    ) = // answeredInRound
        priceFeed.latestRoundData();
    ...
    // validation
    require(answer > 0, "Invalid price");
...
}
```

Since the collateral should be `DAI` or `LUSD` from the beginning i don't think it is safe to assume that the price should be `~$1` at all times. If a de-peg happens the price could fall sharply or if the oracle reports an incorrect price.

Let's look at the following scenario:
- `LUSD` is the collateral and its current price is `$1`
- `User1` mints dollar calling `mintDollar()` and mints `1000 dollar` tokens.
- `User2` mints dollar calling `mintDollar()` and mints `1000 dollar` tokens.
- `LUSD` depegs to `$0.5`
- `User2` decides to redeem his dollars to collateral (maybe unaware of the fall in price or he speculates that the token will repeg) calling the `redeemDollar()` with `1000` dollar tokens
- This will calculate the amount of collateral owed to the user as `1000 / 0.5 = 2000 LUSD`, which is the whole collateral on the contract.
```solidity
function getDollarInCollateral(
    uint256 collateralIndex,
    uint256 dollarAmount
) internal view returns (uint256) {
    UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();
    // 1000e18 * 1e6 / 1e12 / 0.99e6 = 1010e6
    return
        dollarAmount
            .mul(UBIQUITY_POOL_PRICE_PRECISION)
            .div(10 ** poolStorage.missingDecimals[collateralIndex])
            .div(poolStorage.collateralPrices[collateralIndex]);
}
```
- User1 cannot redeem his tokens.
- This can lead to a scenario where the price would be < `0.99` but users are unable to burn tokens to raise the price as there is no collateral to redeem.

#### Coded POC
Add this file to the `./packages/contracts/test/diamond/facets/Depeg.t.sol` folder   
And run with `forge test --match-path ./test/diamond/facets/Depeg.t.sol -vv
v --fork-url RPC_URL`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {TWAPOracleDollar3poolFacet} from "../../../src/dollar/facets/TWAPOracleDollar3poolFacet.sol";

import "forge-std/Test.sol";
import "../DiamondTestSetup.sol";
import "forge-std/console.sol";

contract MockERC20 is ERC20 {
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals
    ) ERC20(_name, _symbol) {}

    function mint(address to, uint256 value) public virtual {
        _mint(to, value);
    }

    function burn(address from, uint256 value) public virtual {
        _burn(from, value);
    }
}

interface IMetaPool {
    // Add liquidity to the pool
    function add_liquidity(uint256[2] calldata amounts, uint256 min_mint_amount) external;

    // Remove liquidity from the pool
    function remove_liquidity(uint256 _amount, uint256[2] calldata min_amounts) external;

    // Remove liquidity in a single token
    function remove_liquidity_one_coin(uint256 _token_amount, int128 i, uint256 min_amount) external;

    // Perform a swap from one token to another
    function exchange(int128 from, int128 to, uint256 _from_amount, uint256 _min_to_amount) external returns(uint256);

    // Get the current balance of a token in the pool
    function balances(int128 i) external view returns (uint256);

    // Get the current price of a token in the pool
    function get_virtual_price() external view returns (uint256);

    function coins(uint256 arg0) external view returns (address);

    function get_price_cumulative_last() external view returns (uint256[2] memory);

    function block_timestamp_last() external view returns (uint256);
}

interface ICurveFactory {
    // The function and parameters depend on the specific Curve Factory interface
    function deploy_metapool(address base_pool, string calldata name, string calldata symbol, address token, uint256 A, uint256 fee) external returns (address);
}

contract MetaPoolTest is DiamondTestSetup {
    // This is the Curve.fi DAI/USDC/USDT pool
    address constant private threeCurvePoolAddress  = 0xbEbc44782C7dB0a1A60Cb6fe97d0b483032FF1C7;

    // This is the Curve.fi DAI/USDC/USDT (3Crv) token
    address constant private threeCurveTokenAddress = 0x6c3F90f043a72FA612cbac8115EE7e52BDe6E490;

    // This is the address of curve factory used to deploy a metapool
    address constant private curveFactoryAddress    = 0x0959158b6040D32d04c301A72CBFD6b39E21c9AE;

    ICurveFactory private curveFactory;
    address private metaPoolAddress;
    
    ERC20 private threeCRVToken;
    IMetaPool private metaPool;

    // Mock token used as collateral, should represent LUSD or DAI
    MockERC20 collateralToken;

    // Collateral price in USD with 8 decimals
    int256 collateralPrice = 1e8;

    // Pool ceiling
    uint256 poolCeiling = 50_000e18; // max 50_000 of collateral tokens is allowed

    function setUp() public override{
        super.setUp();

        // Set the Curve Factory address
        curveFactory = ICurveFactory(curveFactoryAddress); 
        threeCRVToken = ERC20(threeCurveTokenAddress);
        collateralToken = new MockERC20("COLLATERAL", "CLT", 18);

        // Deal tokens used for initial liquidity in the pool
        deal(address(dollarToken), address(this), 1000 ether);
        deal(address(threeCRVToken), address(this), 1000 ether);

        // Deploy the MetaPool through Curve Factory
        metaPoolAddress = curveFactory.deploy_metapool(
            threeCurvePoolAddress,  // Address of the 3CRV pool
            "MetaPool Token",       // MetaPool Token Name
            "MPT",                  // MetaPool Token Symbol
            address(dollarToken),
            100,                    // A parameter
            0.003 * 1e10            // Fee (example: 0.3%)
        );

        // Create metapool instance
        metaPool = IMetaPool(metaPoolAddress);

        // Add liquidity to the pool
        dollarToken.approve(address(metaPool), 1000000 ether);
        threeCRVToken.approve(address(metaPool), 1000000 ether);

        // Add liquidity to meta pool
        uint256[2] memory amounts = [uint256(1000 ether), uint256(1000 ether)];
        metaPool.add_liquidity(amounts, 0);


        // Setup a pool
        vm.prank(owner);
        twapOracleDollar3PoolFacet.setPool(metaPoolAddress, address(threeCRVToken));  

        // Setup ubiquityPoolFacet
        // Oracle address set as address(this)
        vm.startPrank(admin);
        ubiquityPoolFacet.addCollateralToken(
            address(collateralToken),
            address(this), 
            poolCeiling
        );

        ubiquityPoolFacet.toggleCollateral(0);

        // Set mint and redeem fees
        ubiquityPoolFacet.setFees(
            0, // collateral index
            0, // 1% mint fee
            0  // 2% redeem fee
        );
        // Set redemption delay to 2 blocks
        ubiquityPoolFacet.setRedemptionDelayBlocks(2);

        // Set mint price threshold to $1.01 and redeem price to $0.99
        ubiquityPoolFacet.setPriceThresholds(1010000, 990000);
        vm.stopPrank();
    }

    // This is a helper function to manipulate the price of the dollar token with large amounts
    // and by passing time so the TWAP reports price > 1.01 so we can mint tokens
    // not important - only used so we can mint dollar tokens
    function raiseDollarPrice() internal {
        address user = address(0x456);
        deal(address(threeCRVToken), user, 1000 ether);

        vm.startPrank(user);
        threeCRVToken.approve(address(metaPool), 1000 ether);
        metaPool.exchange(1, 0, 670 ether, 0);
        twapOracleDollar3PoolFacet.update();
        vm.stopPrank();

        vm.warp(block.timestamp + 13);
        twapOracleDollar3PoolFacet.update();

        vm.startPrank(user);
        metaPool.exchange(1, 0, 1 ether, 0);
        twapOracleDollar3PoolFacet.update();
        vm.stopPrank();

        vm.warp(block.timestamp + 13);
        // twapOracleDollar3PoolFacet.update();
    }

    function basicTrading() internal {
        address user = address(0x456);

        deal(address(dollarToken), user, 100000 ether);

        vm.startPrank(user);
        dollarToken.approve(address(metaPool), 0);
        dollarToken.approve(address(metaPool), 10000 ether);
        metaPool.exchange(0, 1, 200 ether, 0);
        twapOracleDollar3PoolFacet.update();
        vm.stopPrank();

        vm.warp(block.timestamp + 13);
        twapOracleDollar3PoolFacet.update();

        vm.startPrank(user);
        metaPool.exchange(0, 1, 200 ether, 0);
        twapOracleDollar3PoolFacet.update();
        vm.stopPrank();

        vm.warp(block.timestamp + 13);
        twapOracleDollar3PoolFacet.update();
    }

    // We are using the chainlink price feed as address this to easy change the collateralPrice
    function latestRoundData() external view returns (
      uint80 roundId,
      int256 answer,
      uint256 startedAt,
      uint256 updatedAt,
      uint80 answeredInRound
    ) {
        return (0, collateralPrice, 0, block.timestamp - 13 , 0);
    }

    // Decimals of price feed
    function decimals() external view returns (uint8) {
        return 8;
    }

    function testDepeg() public {
        address user = address(0xabc);
        address user2 = address(0x678);

        // Deal tokens to user and user2
        deal(address(collateralToken), user, 1000 ether);
        deal(address(collateralToken), user2, 1000 ether);

        // Raise dollar price to > 1.01 so the mintDollar function can be called
        raiseDollarPrice();

        // Get the current price of the dollar token
        uint256 amount0Out = twapOracleDollar3PoolFacet.consult(address(dollarToken));
        uint256 amount1Out = twapOracleDollar3PoolFacet.consult(address(threeCRVToken));
        console.log("Dollar Token Price :", amount0Out);
        console.log("3CRV LP Token Price:", amount1Out);
        console.log();

        // User mint some dollar tokens and deposit collateral
        vm.startPrank(user);
        collateralToken.approve(address(ubiquityPoolFacet), 1000 ether);
        ubiquityPoolFacet.mintDollar(
            0,       // collateral index
            1000e18, // Dollar amount - pool ceiling
            0,       // min amount of Dollars to mint
            1000 ether
        );
        vm.stopPrank();

        // User2 mint some dollar tokens and deposit collateral
        vm.startPrank(user2);
        collateralToken.approve(address(ubiquityPoolFacet), 1000 ether);
        ubiquityPoolFacet.mintDollar(
            0,       // collateral index
            1000e18, // Dollar amount - pool ceiling
            0,       // min amount of Dollars to mint
            1000 ether
        );
        vm.stopPrank();

        // Some time passes and trades happen that bring TWAP pricei in < 0.99 so users can redeem
        basicTrading();

        uint256 collateralBalanceUbiquityBefore = collateralToken.balanceOf(address(ubiquityPoolFacet));

        // Collateral price falls to $0.5
        collateralPrice = 0.5e8;

        // User2 redeems his dollar tokens and gets all the collateral
        vm.startPrank(user2);
        ubiquityPoolFacet.redeemDollar(0, 1000 ether, 0);

        // Pass the redemptionDelayBlocks delay
        vm.roll(block.number + 2);

        // Collect the redemption
        ubiquityPoolFacet.collectRedemption(0);
        vm.stopPrank();

        //User1 tries to redeem but fails due to insufficient collateral
        vm.startPrank(user);
        vm.expectRevert("Insufficient pool collateral");
        ubiquityPoolFacet.redeemDollar(0, 1000 ether, 0);
        vm.stopPrank();

        uint256 collateralBalanceUbiquityAfter = collateralToken.balanceOf(address(ubiquityPoolFacet));

        // Get the current price of the dollar token
        amount0Out = twapOracleDollar3PoolFacet.consult(address(dollarToken));
        amount1Out = twapOracleDollar3PoolFacet.consult(address(threeCRVToken));
        console.log("Dollar Token Price :", amount0Out);
        console.log("3CRV LP Token Price:", amount1Out);

        console.log("---------------------------------------");
        console.log("collateralBalanceUbiquityBefore:", collateralBalanceUbiquityBefore);
        console.log("collateralBalanceUbiquityAfter :", collateralBalanceUbiquityAfter);
    }
}
```

```solidity
[PASS] testDepeg() (gas: 1596989)
Logs:
  Dollar Token Price : 1014999605046437077
  3CRV LP Token Price: 979034204935612062
  
  Dollar Token Price : 985079916536919819
  3CRV LP Token Price: 1008988932820357703
  ---------------------------------------
  collateralBalanceUbiquityBefore: 2000000000000000000000
  collateralBalanceUbiquityAfter : 0
```
## Impact
Users can be left unable to burn their tokens for collateral as there would soon be no collateral left. The process of burning collateral is meant to raise the price of the dollar token. But if the tokens cannot be burned then the price cannot go back up.
## Code Snippet
https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L523-L562

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/dollar/libraries/LibUbiquityPool.sol#L284-L294
## Tool used
Manual Review

## Recommendation
Don't allow mints or redemptions if the price falls out of range. Add the `minPrice` and `maxPrice` to the pool storage for each collateral and then do the checks. This could be set for example `minPrice = 0.99` and `maxPrice = 1.01`. If you plan to support assets such as `WBTC/USD` it is a good idea to check the `WBTC/BTC` feed to make sure that it is still pegged to the `BTC`.

```diff
function updateChainLinkCollateralPrice(uint256 collateralIndex) internal {
    UbiquityPoolStorage storage poolStorage = ubiquityPoolStorage();

    AggregatorV3Interface priceFeed = AggregatorV3Interface(
        poolStorage.collateralPriceFeedAddresses[collateralIndex]
    );
    
+   uint256 minPrice = poolStorage.minPrice[0];
+   uint256 maxPrice = poolStorage.maxPrice[0];

    // fetch latest price
    (
        ,
        // roundId
        int256 answer, // startedAt
        ,
        uint256 updatedAt,

    ) = // answeredInRound
        priceFeed.latestRoundData();
        
+   require(answer > minPrice && answer < maxPrice, "Invalid price");
    ...
}
```