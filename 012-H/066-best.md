Overt Fiery Starfish

Medium

# Uninitialized cache.redeemFee cause 0 redeem fee

## Summary
`cache.redeemFee` is not initialized correctly, which cause LP holders don't need to pay the redeem fee. This is not expected behavior.
 
## Vulnerability Detail
When LP holders redeem liquidity, LP holders need to pay some redeem fees. In function `_executeRedeemStakeToken `, the actual redeem fee is calculated and save into the variable `redeemFee`. The vulnerability is that contract use `cache.redeemTokenAmount - cache.redeemFee` to calculate the final amount that LP holders can redeem. However, `cache.redeemFee` is not initialised and the default value is 0. 

This means that the LP holders don't need to pay the redeem fee. This is not one expected behavior. What's more, this redeem fee is charged by fee rewards. In the future, these redeem fees may be transferred out to reward contract. However, in fact, LP holders don't leave any redeem fees. This could cause the LP pool's account into a mess.

```javascript
    function _executeRedeemStakeToken(
        LpPool.Props storage pool,
        Redeem.Request memory params,
        address baseToken
    ) internal returns (uint256) {
       ......
@==> actual redeem Fee.
        uint256 redeemFee = FeeQueryProcess.calcMintOrRedeemFee(cache.redeemTokenAmount, poolConfig.redeemFeeRate);
        FeeProcess.chargeMintOrRedeemFee(
            redeemFee,
            params.stakeToken,
            params.redeemToken,
            params.account,
            FeeProcess.FEE_REDEEM,
            false
        );
@==> cache.redeemFee is not initialized to redeemFee.
        VaultProcess.transferOut(
            params.stakeToken,
            params.redeemToken,
            params.receiver,
            cache.redeemTokenAmount - cache.redeemFee
        );
```

## Impact
- LP holders don't need to pay the redeem fees.
- The redeem fees are charged by fee rewards. However, LP holders don't leave any redeem fees in the pool, which will cause the pool's account into a mess.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L133-L183

## Tool used

Manual Review

## Recommendation
Initialized `cache.redeemFee`