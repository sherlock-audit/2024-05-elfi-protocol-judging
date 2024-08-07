Energetic Lemonade Woodpecker

Medium

# Accounting error when decreasing position leverage

## Summary
Accounting error when `PositionMarginProcess::_executeReduceMargin` is called during position leverage or margin update.


## Vulnerability Detail
When `PositionMarginProcess::_executeReduceMargin` is called when updating position leverage, part of its margin is decreased and sent to the account owner. But before that, `LpPoolProcess::holdPoolAmount` is called to check if the pool can support the withdrawal. The pool checks if its a long or short position and checks the stakeToken pool balance for balance sufficiency. 
The problem is that if it is a short position, the function only checks the UsdPool, assuming short positions can't be opened with crypto tokens. However, short positions can be open with crypto tokens, and in this scenario, the pool is deducted without updating its holdAmount.


## Impact
`LpPoolProcess::holdPoolAmount` causes an under appreciation of a users account.


## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/LpPoolProcess.sol#L17-L34
```js
    function holdPoolAmount() external {
        if (isLong) {} else {
            UsdPool.Props storage pool = UsdPool.load();
            if (
                !UsdPool.isHoldAmountAllowed(pool.stableTokenBalances[token], UsdPool.getPoolLiquidityLimit(), amount)
            ) {
                revert Errors.PoolAmountNotEnough(stakeToken, token);
            }
            pool.holdStableToken(token, amount);
        }
    }
```


## Tool used

Manual Review


## Recommendation
Factor in crypto token pool, when checking available liquidity.