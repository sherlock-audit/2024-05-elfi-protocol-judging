Active Punch Jellyfish

High

# Excess fromBalance removal not added to other positions fromBalance's when leveraging up

## Summary
When new deposits or withdrawals are requested by users in cross accounts, it will change the from balances of all positions. This is crucial for Elfi to maintain accurate cross-available values and borrowed amounts. Deleveraging also functions as a form of withdrawal since it moves capital back to the portfolio vault. If the amount to be pulled is enough to cover the deleveraged position's margin, it is from balance is capped at that. However, the excess amount is not used to cover other positions from balances. 
## Vulnerability Detail
Assume Alice has two positions:
1. SHORT BTC with margin token USDC: margin is $100 and fromBalance is $100.
2. SHORT SOL with margin token USDC: margin is $100 and fromBalance is $40.

After some time, Alice decides to withdraw 10 USDC. Since order 1 is first in the queue, the fromBalance decreases to $90 as we can observe in below code snippet:

```solidity
function withdraw(uint256 requestId, WithdrawParams memory params) public {
    //.
    accountProps.subTokenIgnoreUsedAmount(params.token, params.amount, Account.UpdateSource.WITHDRAW);
    -> PositionMarginProcess.updateAllPositionFromBalanceMargin(
        requestId,
        params.account,
        params.token,
        -(params.amount.toInt256()),
        ""
    );
}
```

Now Alice's positions are:
- SHORT BTC: margin $100, fromBalance $90
- SHORT SOL: margin $100, fromBalance $40

If Alice's SOL position is 10x, her quantity (QTY) is $1000. She decides to increase the leverage to 50x, reducing the required margin to $20. Alice needs to unhold $80 worth of USDC, reducing the "fromBalance" accordingly.

```solidity
function _executeReduceMargin(
    Position.Props storage position,
    Symbol.Props memory symbolProps,
    uint256 reduceMargin,
    bool needUpdateLeverage
) internal returns (uint256) {
    //.
    uint256 reduceMarginAmount = CalUtils.usdToToken(reduceMargin, decimals, marginTokenPrice);
    if (position.isCrossMargin &&
        position.initialMarginInUsd - position.initialMarginInUsdFromBalance < reduceMargin
    ) {
        -> position.initialMarginInUsdFromBalance -= (reduceMargin -
            (position.initialMarginInUsd - position.initialMarginInUsdFromBalance)).max(0);
    }
    position.initialMargin -= reduceMarginAmount;
    position.initialMarginInUsd -= reduceMargin;
    return reduceMarginAmount;
}
```

After updating leverage, Alice's SOL short position will be:
- Margin: $20
- fromBalance: $20

and 80 USDC will be unused:

```solidity
function updatePositionLeverage(uint256 requestId, UpdateLeverage.Request memory request) external {
    //.
    position.leverage = request.leverage;
    uint256 reduceMargin = position.initialMarginInUsd - CalUtils.divRate(position.qty, position.leverage);
    -> uint256 reduceMarginAmount = _executeReduceMargin(position, symbolProps, reduceMargin, false);
    if (position.isCrossMargin) {
        -> accountProps.unUseToken(
            position.marginToken,
            reduceMarginAmount,
            Account.UpdateSource.UPDATE_LEVERAGE
        );
    } else {
        VaultProcess.transferOut(
            symbolProps.stakeToken,
            request.marginToken,
            request.account,
            reduceMarginAmount
        );
    }
}
```

Alice has "unused" $80 USDC but only $20 was reduced from the "fromBalance" of the SOL short position. The remaining $60 is not added to her BTC short position's "fromBalance". If Alice had done this as a withdrawal, it would have updated all "fromBalances" by looping through the positions until the unused amount was fully exhausted.
## Impact
Users will have less "fromBalance" than they should, increasing their cross available value. Hence, high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/AssetsProcess.sol#L122C5-L155C6

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L274-L338

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L134-L229

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L370-L406
## Tool used

Manual Review

## Recommendation
If there is an excess amount that can't be decreased from the current positions fromBalance then loop and add to other positions until its fully used just like its done in deposit/withdraw flows.