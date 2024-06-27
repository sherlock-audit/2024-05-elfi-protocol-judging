Magic Black Crane

High

# Function `OracleFacet.setOraclePrices()` is missing access control

## Summary

The function `setOraclePrices()` is missing access control and consequently any user can call it to update the price of any asset.

## Vulnerability Detail

The `OracleFacet` implements the function `setOraclePrices()`:
```solidity
function setOraclePrices(OracleProcess.OracleParam[] calldata params) external {
    OracleProcess.setOraclePrice(params);
}
```

This function sets the price of the different assets supported by the protocol. As the function, lacks access control, any user can call it to manipulate the price of any asset. **This can be abused to drain the protocol**. 

While it is true that all the execute functions in the protocol include a `OracleProcess.OracleParam[] calldata oracles` parameter to set the price of the assets and these functions can only be called by a keeper, if the keeper does not update the assets that are part of the request trusting that their price is already correct the protocol could be drained.

## Impact

High, prices can be manipulated.

## Code Snippet

- [OracleFacet.sol#L11-L13](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/OracleFacet.sol#L11-L13)

Note that even if this facet is not directly listed in the scope the other facets interact with it constantly by calling the function `getLatestUsdPrice()` so it can be considered in scope.

## Tool used

Manual Review

## Recommendation

Enforce that the `OracleFacet.setOraclePrices()` can only be called by a keeper.
