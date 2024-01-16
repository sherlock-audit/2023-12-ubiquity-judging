Soft Coconut Mongoose

medium

# TWAP oracle is incompatible with current Curve metapool implementation

## Summary

The current TWAP oracle implementation is compatible with older versions of Curve pools which are no longer used. Since the protocol plans to redeploy the pool used for uAD with the new version of the token (see https://github.com/ubiquity/ubiquity-dollar/discussions/846), this will create a compatibility conflict with the currently used revisions of Curve pools.

## Vulnerability Detail

The TWAPOracle contract relies on older versions of the Curve metapool implementation that expose the following functions `get_price_cumulative_last()`, `block_timestamp_last()`, `get_twap_balances()`. This implementation is really old and is not used anymore, not even the non-ng stablepool factory uses these functions anymore (see https://github.com/curvefi/curve-factory/blob/master/contracts/implementations/meta/MetaUSD.vy).

The current updated supported versions of Curve pools are the new stablepool NG implementations. These are the ones recommended by Curve and the ones used by the UI at https://curve.fi/. The factory address and current metapool blueprint are detailed below. 

Factory: https://etherscan.io/address/0x6a8cbed756804b16e05e741edabd5cb544ae21bf#code

Metapool implementation: https://etherscan.io/address/0xede71F77d7c900dCA5892720E76316C6E575F0F7#code

As we can see, the new metapool contract (CurveStableSwapMetaNG) no longer provides the TWAP utility functions. The updated implementation now bundles a moving average oracle given by the function `price_oracle()`.

## Impact

The TWAP oracle implementation is incompatible with. If the protocol plans to deploy a new Curve pool to use the new uAD token, then the current TWAP oracle implementation will not work.

## Code Snippet

https://github.com/sherlock-audit/2023-12-ubiquity/blob/main/ubiquity-dollar/packages/contracts/src/deprecated/TWAPOracle.sol#L15-L39

## Tool used

Manual Review

## Recommendation

As the protocol plans to deploy a new Curve pool, and since the updated pool implementation now bundles an oracle, it is recommended to drop the current TWAPOracle contract and directly consult the oracle of the pool.