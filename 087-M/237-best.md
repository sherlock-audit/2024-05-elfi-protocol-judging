Great Maroon Wasp

Medium

# The calculations in `isSubAmountAllowed` overcounted balance.unsettledAmount in LpPool.sol



## Summary
The calculations in `isSubAmountAllowed` overcounted balance.unsettledAmount in LpPool.sol
## Vulnerability Detail
```javascript
 function isSubAmountAllowed(Props storage self, address token, uint256 amount) internal view returns (bool) {
        TokenBalance storage balance = token == self.baseToken
            ? self.baseTokenBalance
            : self.stableTokenBalances[token];
        if (balance.amount < amount) {
            return false;
        }
        uint256 poolLiquidityLimit = getPoolLiquidityLimit(self);
        if (poolLiquidityLimit == 0) {
            return
 @>>               balance.amount.toInt256() + balance.unsettledAmount - balance.holdAmount.toInt256() >=
                amount.toInt256();
        } else {
            return
                CalUtils.mulRate(
 @>>                   balance.amount.toInt256() - amount.toInt256() + balance.unsettledAmount,
                    poolLiquidityLimit.toInt256()
                ) >= balance.holdAmount.toInt256();
        }
    }

```
We can see that the calculations in `isSubAmountAllowed` overcounted balance.unsettledAmount. The balance.unsettledAmount represents the fees earned by the pool, but the assets have not yet been transferred. 
The same issues are in `getPoolAvailableLiquidity` and `getUsdPoolAvailableLiquidity` in LpPoolQueryProcess.sol.
## Impact
The higher-level function calls to getMaxWithdraw and isSubAmountAllowed should return false, but they return true instead, preventing the function from continuing to execute correctly.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/LpPool.sol#L401
## Tool used

Manual Review

## Recommendation
```diff
 function isSubAmountAllowed(Props storage self, address token, uint256 amount) internal view returns (bool) {
        TokenBalance storage balance = token == self.baseToken
            ? self.baseTokenBalance
            : self.stableTokenBalances[token];
        if (balance.amount < amount) {
            return false;
        }
        uint256 poolLiquidityLimit = getPoolLiquidityLimit(self);
        if (poolLiquidityLimit == 0) {
            return
-               balance.amount.toInt256() + balance.unsettledAmount - balance.holdAmount.toInt256() >=
+               balance.amount.toInt256() - balance.holdAmount.toInt256() >=
                amount.toInt256();
        } else {
            return
                CalUtils.mulRate(
-                   balance.amount.toInt256() - amount.toInt256() + balance.unsettledAmount,
+                   balance.amount.toInt256() - amount.toInt256() ,
                    poolLiquidityLimit.toInt256()
                ) >= balance.holdAmount.toInt256();
        }
    }

```

