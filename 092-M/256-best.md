Magnificent Syrup Tardigrade

High

# While Calculating the value for availableLiquidity we need to subtract the holdAmount as it is not part of Liquidity yet.

## Summary
While Calculating the Liquidity for `stableToken` we did not care about `holdAmount` , `holdAmount` is not part of Current Liquidity.In case of `baseToken` we subtract `holdAmount` from `availableLiquidity`


## Vulnerability Detail
The `getPoolAvailableLiquidity` function is used to check for available liquidity in terms of `baseTokens`. 
```solidity
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
 @>                   tokenBalance.amount.toInt256() + tokenBalance.unsettledAmount < tokenBalance.lossAmount.toInt256()
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
@>                ? (availableTokenAmount - pool.baseTokenBalance.holdAmount.toInt256()).toUint256()
                : 0;
    }
``` 
The if condition only check for ` tokenBalance.amount.toInt256() + tokenBalance.unsettledAmount<tokenBalance.lossAmount.toInt256()` which means that all the token balance in amount and unsettled is available for liquidity which is not correct we also `holdAmount` from these balances. We do not subtract the `holdAmount` from Liquidity in case of `stableToken` while in  `baseToken` case we do.

## Impact
The amount calculated for AvailableLiquidity in stableCoin is not correct it will impact the case where we subtract from available liquidity.

## Code Snippet
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L151-L191](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L151-L191)
## Tool used

Manual Review

## Recommendation
Also handle the `holdAmount` in case of `stableToken` balances.
