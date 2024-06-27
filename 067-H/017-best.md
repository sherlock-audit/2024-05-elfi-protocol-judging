Rough Emerald Gazelle

High

# OracleProcess.getOraclePrices() might retrive the wrong price for a pair, leading to loss of funds due to wrong price calculation.

## Summary
OracleProcess.getOraclePrices() might retrive the wrong price for a pair, leading to loss of funds due to wrong price calculation. 

## Vulnerability Detail

OracleProcess.getOraclePrices() is supposed to return the max/min price of a pair. 

[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OracleProcess.sol#L70-L82](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OracleProcess.sol#L70-L82)

However, due to failing to match the target token (it only matches the ``token``), it will match the first pair whose ``token`` will match but ``targetToken`` might not match. As a result, it can return the wrong price for a pair. 

Consider the following POC with their pair prices:

1) (token1, token2): (300, 400);
2) (token2, token3): (400, 500);
3) (token1, token3): (500, 600). 
4) When we try to retrive the max price for (token1, token3), instead of returning 600, it returns 400, the wrong value. 

```javascript
function testOracle() public{
        address token1 = makeAddr("token1");
        address token2 = makeAddr("token2");
        address token3 = makeAddr("token3");

        OracleProcess.OracleParam[] memory oracleParams = new OracleProcess.OracleParam[](3); 
        oracleParams[0] = OracleProcess.OracleParam({
            token: token1,
            targetToken: token2,
            minPrice: 300,
            maxPrice: 400
        });

        oracleParams[1] = OracleProcess.OracleParam({
            token: token2,
            targetToken: token3,
            minPrice: 400,
            maxPrice: 500
        });


        oracleParams[2] = OracleProcess.OracleParam({
            token: token1,
            targetToken: token3,
            minPrice: 500,
            maxPrice: 600
        });


        console2.log("max price of pair of token1, token3: ", OracleProcess.getOraclePrices(oracleParams, token1, token3, false));
    }
```

## Impact
OracleProcess.getOraclePrices() might retrive the wrong price for a pair, leading to loss of funds due to wrong price calculation. 
## Code Snippet

## Tool used
Foundry

Manual Review

## Recommendation
We need to compare both "token" and "targetToken":

```diff
 function getOraclePrices(
        OracleParam[] memory params,
        address token,
        address targetToken,
        bool isMin
    ) external view returns (uint256) {
        for (uint256 i; i < params.length; i++) {
-            if (params[i].token == token) {
+            if (params[i].token == token && params[i].targetToken == targetToken) {
                return isMin ? uint256(params[i].minPrice) : uint256(params[i].maxPrice);
            }
        }
        return getLatestUsdUintPrice(token, targetToken, isMin);
    }
```