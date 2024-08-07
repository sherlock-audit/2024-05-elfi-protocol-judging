Great Maroon Wasp

Medium

# Liquidity providers are unreasonably restricted by pool.getPoolAvailableLiquidity() when redeeming Stake Tokens


## Summary
Liquidity providers are unreasonably restricted by pool.getPoolAvailableLiquidity() when redeeming Stake Tokens
## Vulnerability Detail
```javascript
function _executeRedeemStakeToken(
        LpPool.Props storage pool,
        Redeem.Request memory params,
        address baseToken
    ) internal returns (uint256) {
        ExecuteRedeemCache memory cache;
        cache.poolValue = pool.getPoolValue();
        cache.totalSupply = TokenUtils.totalSupply(pool.stakeToken);
        cache.tokenDecimals = TokenUtils.decimals(baseToken);
        if (cache.poolValue == 0 || cache.totalSupply == 0) {
            revert Errors.RedeemWithAmountNotEnough(params.account, baseToken);
        }

        cache.unStakeUsd = params.unStakeAmount.mul(cache.poolValue).div(cache.totalSupply);
        cache.redeemTokenAmount = CalUtils.usdToToken(
            cache.unStakeUsd,
            cache.tokenDecimals,
            OracleProcess.getLatestUsdUintPrice(baseToken, false)
        );

@>>        if (pool.getPoolAvailableLiquidity() < cache.redeemTokenAmount) {
            revert Errors.RedeemWithAmountNotEnough(params.account, params.redeemToken);
        }
        //skip ......
       
    }
```

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
                    tokenBalance.lossAmount > 0 &&
                    tokenBalance.amount.toInt256() + tokenBalance.unsettledAmount < tokenBalance.lossAmount.toInt256()
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
@>>        int256 availableTokenAmount = CalUtils.mulRate(baseTokenAmount, pool.getPoolLiquidityLimit().toInt256());
        return
@>>            availableTokenAmount > pool.baseTokenBalance.holdAmount.toInt256()
                ? (availableTokenAmount - pool.baseTokenBalance.holdAmount.toInt256()).toUint256()
                : 0;
    }
```
The above calculation formula is incorrect. 
For example:
 a user stakes 100 ETH into the pool, and balance.holdAmount is 0 (there are no positions at this time). 
 The user wants to withdraw all their stake, but according to the above formula, with a limit of 0.8, 
 availableTokenAmount = 100 eth * 0.8 = 80 ETh.
 the user is allowed to withdraw only  80 ETH, However, the user should be able to withdraw 100 ETH..

The correct calculation formula should be the same as in `getMaxWithdraw()`.
```javascript
function getMaxWithdraw(Props storage self, address stableToken) public view returns (uint256) {
        TokenBalance storage balance = self.stableTokenBalances[stableToken];
        uint256 poolLiquidityLimit = getPoolLiquidityLimit();
        if (poolLiquidityLimit == 0) {
            return balance.amount - balance.holdAmount;
        } else {
@>>            uint256 holdNeedAmount = CalUtils.divRate(balance.holdAmount, poolLiquidityLimit);
@>>            return balance.amount > holdNeedAmount ? balance.amount - holdNeedAmount : 0;
        }
    }
```

## Impact
The incorrect calculation affects users’ ability to redeem Stake Tokens.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L133

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L151

## Tool used

Manual Review

## Recommendation
Modify the calculation formula.
