Magnificent Syrup Tardigrade

Medium

# The USer will receive less amount  than user expected

## Summary
While redeeming the stack tokens, The user provides the `minRedeemAmount` to ensure they receive at least that amount. However,  within `executeRedeemStakeToken` function, the  `minRedeemAmount` check is used before deducting the fee . which could result in  the user  receive less amount than the expected amount.


## Vulnerability Detail
The Protocol allows user to specify the `minRedeemAmount` to insure that the user will receive this amount or in other case the transaction will revert. The User will first submit a request for Redemption where he also specify this `minRedeemAmount` which user expect to receive. The Issue is in the execute redemption request flow.
```solidity
function _executeRedeemStakeToken(
        LpPool.Props storage pool,
        Redeem.Request memory params,
        address baseToken
    ) internal returns (uint256) {
        ...
        cache.redeemTokenAmount = CalUtils.usdToToken(
            cache.unStakeUsd,
            cache.tokenDecimals,
            OracleProcess.getLatestUsdUintPrice(baseToken, false)
        );

        if (pool.getPoolAvailableLiquidity() < cache.redeemTokenAmount) {
            revert Errors.RedeemWithAmountNotEnough(params.account, params.redeemToken);
        }

 @>     if (params.minRedeemAmount > 0 && cache.redeemTokenAmount < params.minRedeemAmount) {
            revert Errors.RedeemStakeTokenTooSmall(cache.redeemTokenAmount);
        }
        ...
        FeeProcess.chargeMintOrRedeemFee(
            redeemFee,
            params.stakeToken,
            params.redeemToken,
            params.account,
            FeeProcess.FEE_REDEEM,
            false
        );
        VaultProcess.transferOut(
            params.stakeToken,
            params.redeemToken,
            params.receiver,
@>         cache.redeemTokenAmount - cache.redeemFee
        );
        pool.subPoolAmount(pool.baseToken, cache.redeemTokenAmount);
        StakeToken(params.stakeToken).burn(params.account, params.unStakeAmount);
        stakingAccountProps.subStakeAmount(params.stakeToken, params.unStakeAmount);

        return cache.redeemTokenAmount;
    }
```
As it can be observed from above code that we first convert the `unStkaeUsd` amount and store receive value in `cache.redeemTokenAmount`.Than we check for `minRedeemAmount`
and than we deduct the fee and transfer the remaining `redeemTokenAmount` to user.
Following case would occur due to this:
1. Bob submit a request to redeem 10e18 token and expect to receive 9e18 token.
2. the Protocol convert the amount using latest oracle price and get 9 token as redeemTokenAmount.
3. The `cache.redeemTokenAmount < params.minRedeemAmount` check will pass as 9e18 < 9e18.
4. The `RedeemFeeRate=10` and `RATE_PRECISION=100000` Now Applying these values to calculate the Fee amount is `9e18*10/100000= 9e14`.
5. The amount Bob will receive is `9e18-9e14≈8.9e17`.


This applies on both functions `_executeRedeemStakeUsd` and `_executeRedeemStakeToken`.

## Impact
The user will receive less amount than expected.


## Code Snippet
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L157](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L157)

## Tool used

Manual Review

## Recommendation
Use slippage check after deducting the Fee.
```diff
diff --git a/elfi-perp-contracts/contracts/process/RedeemProcess.sol b/elfi-perp-contracts/contracts/process/RedeemProcess.sol
index dedfe16e..eb6c84fe 100644
--- a/elfi-perp-contracts/contracts/process/RedeemProcess.sol
+++ b/elfi-perp-contracts/contracts/process/RedeemProcess.sol
@@ -200,9 +200,7 @@ library RedeemProcess {
             tokenDecimals,
             OracleProcess.getLatestUsdUintPrice(params.redeemToken, false)
         );
-        if (params.minRedeemAmount > 0 && redeemTokenAmount < params.minRedeemAmount) {
-            revert Errors.RedeemStakeTokenTooSmall(redeemTokenAmount);
-        }
         if (pool.getMaxWithdraw(params.redeemToken) < redeemTokenAmount) {
             revert Errors.RedeemWithAmountNotEnough(params.account, params.redeemToken);
         }
@@ -219,6 +217,9 @@ library RedeemProcess {
             FeeProcess.FEE_REDEEM,
             false
         );
+        if (params.minRedeemAmount > 0 && redeemTokenAmount-redeemFee < params.minRedeemAmount) {
+            revert Errors.RedeemStakeTokenTooSmall(redeemTokenAmount);
+        }
 
         StakeToken(params.stakeToken).burn(account, params.unStakeAmount);
         StakeToken(params.stakeToken).transferOut(params.redeemToken, params.receiver, redeemTokenAmount - redeemFee);
```
