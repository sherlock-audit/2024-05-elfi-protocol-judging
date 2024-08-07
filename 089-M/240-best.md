Great Maroon Wasp

Medium

# The profits of LpPool  are not included in AvailableLiquidity in `getPoolAvailableLiquidity()`


## Summary
The profits of LpPool  are not included in AvailableLiquidity in `getPoolAvailableLiquidity()`
## Vulnerability Detail
```javascript
 function getPoolAvailableLiquidity(
        LpPool.Props storage pool,
        OracleProcess.OracleParam[] memory oracles
    ) public view returns (uint256) {
        int256 baseTokenAmount = pool.baseTokenBalance.amount.toInt256() + pool.baseTokenBalance.unsettledAmount;
        if (baseTokenAmount < 0) {
            return 0;
        }

        address[] memory stableTokens = pool.getStableTokens();
        if (stableTokens.length > 0) {
            uint8 baseTokenDecimals = TokenUtils.decimals(pool.baseToken);
            int256 baseTokenPrice = OracleProcess.getIntOraclePrices(oracles, pool.baseToken, true);
            for (uint256 i; i < stableTokens.length; i++) {
                LpPool.TokenBalance storage tokenBalance = pool.stableTokenBalances[stableTokens[i]];
                if (
@>>                    tokenBalance.lossAmount > 0 &&
@>>                    tokenBalance.amount.toInt256() + tokenBalance.unsettledAmount < tokenBalance.lossAmount.toInt256()
                ) {
                    int256 tokenUsd = CalUtils.tokenToUsdInt(
                        tokenBalance.lossAmount.toInt256() -
                            tokenBalance.amount.toInt256() -
                            tokenBalance.unsettledAmount,
                        TokenUtils.decimals(stableTokens[i]),
                        OracleProcess.getIntOraclePrices(oracles, stableTokens[i], true)
                    );
                    int256 stableToBaseToken = CalUtils.usdToTokenInt(tokenUsd, baseTokenDecimals, baseTokenPrice);
                    if (baseTokenAmount > stableToBaseToken) {
                        baseTokenAmount -= stableToBaseToken;
                    } else {
                        baseTokenAmount = 0;
                    }
                }
            }
        }
        int256 availableTokenAmount = CalUtils.mulRate(baseTokenAmount, pool.getPoolLiquidityLimit().toInt256());
        return
            availableTokenAmount > pool.baseTokenBalance.holdAmount.toInt256()
                ? (availableTokenAmount - pool.baseTokenBalance.holdAmount.toInt256()).toUint256()
                : 0;
    }
```
We can see that only when the pool has losses, and the losses are greater than the profits, the difference in the loss of stablecoins (i.e., the user’s profit) is calculated. However, stablecoin profits are not accounted for in cases where there are profits.

## Impact
The expected liquidity is not correctly calculated, affecting the liquidity providers’ ability to withdraw their tokens.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L151
## Tool used

Manual Review

## Recommendation
```diff
if (
-                   tokenBalance.lossAmount > 0 &&
-                   tokenBalance.amount.toInt256() + tokenBalance.unsettledAmount < tokenBalance.lossAmount.toInt256()
+                    tokenBalance.lossAmount > 0 || tokenBalance.amount > 0 || tokenBalance.unsettledAmount > 0
                )
```

