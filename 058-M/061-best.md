Overt Fiery Starfish

Medium

# Incorrect settleFee process for cross-margin account

## Summary
Settle fee is processed twice when the closed/decreased position is cross-margin position.

## Vulnerability Detail
When the traders want to close or decrease their cross-margin positions, settle fees are generated. Settle fees include borrow fee, funding fee, and close fee.  When the settle fee is positive, the traders need to pay for the settle fee, otherwise, the traders will receive settle fees.
The vulnerability exists in function `_settleCrossAccount`. In this function, we will update the trader's cross-margin account via `subTokenWithLiability` or `addToken` at the first time. After that, if Pnl is larger than 0, settle fee will be process again, which is in wrong direction.

For example, if the settle fee is 10. The function will decrease this settle fees from cross-margin account via `subTokenWithLiability`, and then increase this settle fees to the cross-margin account via `addToken`. This means that the traders don't need to pay for the settle fee. Of course, the traders cannot gain the settle fee profit if the settle fee is negative.

What's more, considering that when `cache.recordPnlToken` is positive and `cache.settledFee` is negative, and the sum of `cache.recordPnlToken` and `cache.settledFee` is negative, this could cause reverted because `(cache.recordPnlToken + cache.settledFee).toUint256()` cast failure.

```javascript
    function _settleCrossAccount(
        uint256 requestId,
        Account.Props storage accountProps,
        Position.Props storage position,
        DecreasePositionCache memory cache
    ) internal returns (uint256 addLiability) {
@==> process settle fee at the first time
        if (cache.settledFee > 0) {
            accountProps.subTokenWithLiability(
                cache.position.marginToken,
                cache.settledFee.toUint256(),
                Account.UpdateSource.SETTLE_FEE
            );
        } else {
            //Add some settled fee in cross-margin account
            accountProps.addToken(
                cache.position.marginToken,
                (-cache.settledFee).toUint256(),
                Account.UpdateSource.SETTLE_FEE
            );
        }
        // decrease used_amount
        accountProps.unUseToken(
            cache.position.marginToken,
            cache.decreaseMargin,
            Account.UpdateSource.DECREASE_POSITION
        );
        address portfolioVault = IVault(address(this)).getPortfolioVaultAddress();
        // trader wins in cross-margin mode
        if (cache.recordPnlToken >= 0) {
       @==> process the settle fee again.
            accountProps.addToken(
                cache.position.marginToken,
                (cache.recordPnlToken + cache.settledFee).toUint256(),
                Account.UpdateSource.SETTLE_PNL
            );
```
## Impact
- Settle fees are not processed correctly, traders may pay less fee than they should, may gain less fee than they deserve.
- Decrease order may be reverted when `cache.recordPnlToken + cache.settledFee` is negative.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L338-L368

## Tool used

Manual Review

## Recommendation
Don't process the settle fee twice.