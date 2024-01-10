Sparkly Taffy Shark

medium

# Stale Data issue in Oracle Update and Consult Functions

## Summary
see vulnerability details 
## Vulnerability Detail
the contract is serves as a TWAP oracle for a Curve MetaPool involving Dollar and 3CRV LP tokens.
and we have these functions  :
     - setPool is to Sets the Curve MetaPool for TWAP calculations.
     - update toUpdates state variables based on the latest values from the MetaPool.
     - consult is returns the price quote for a specified token.
the vulnerability  is exists in both the `update ` and `consult ` functions, so the first function it's has Absence of a mechanism to ensure the oracle data is timely updated. and the consult function has no verification to ensure that the consulted data is fresh and not stale.
- here is the vulnerable part :
```solidity
   */
    function update() external {
        LibTWAPOracle.update();
    }

    /**
     * @notice Returns the quote for the provided `token` address
     * @notice If the `token` param is Dollar then returns 3CRV LP / Dollar quote
     * @notice If the `token` param is 3CRV LP then returns Dollar / 3CRV LP quote
     * @dev This will always return 0 before update has been called successfully for the first time
     * @param token Token address
     * @return amountOut Token price, Dollar / 3CRV LP or 3CRV LP / Dollar quote
     */
    function consult(address token) external view returns (uint256 amountOut) {
        return LibTWAPOracle.consult(token);
    }
}
```
- if the actual market rate is more favorable than the stale rate, the attacker could use stale data in the oracle, to buy or sell large quantities of Dollar or 3CRV LP at non-market favorable prices, resulting in financial gain for the attacker and potential losses for other uninformed users.
- The attacker could perform trades that take advantage of the stale data before others notice, or before the oracle is updated, and then manipulating the market in their favor.
## Impact
- let's Assume the update function hasn't been called for an extended period, and the market prices have significantly changed. A user consulting the oracle via the consult function during this period receives outdated price information. This incorrect information can be used to exploit market positions or execute trades at non-market favorable prices,  and then leading to potential financial losses for uninformed users.
## Code Snippet
- https://github.com/sherlock-audit/2023-12-ubiquity/blob/d9c39e8dfd5601e7e8db2e4b3390e7d8dff42a8e/ubiquity-dollar/packages/contracts/src/dollar/facets/TWAPOracleDollar3poolFacet.sol#L31C3-L47C2
## Tool used
Manual Review
## Recommendation
-it's need a timestamp check in the update function to ensure that the oracle data is not stale as an example to fix it is to is to define a maximum time interval within which data must be updated and revert the transaction if the current timestamp exceeds this interval since the last update.