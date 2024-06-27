Magnificent Syrup Tardigrade

High

# `LpPool:unHoldStableToken` will always revert due to wrong require statement

## Summary
LpPool support hold and unHold stable token amount, However when `unHoldStableToken` is called the function will always revert. 

## Vulnerability Detail
The stable token in LpPool can also be Hold if the execution of funds is pending. so In this case we fist call `holdStableToken` and , for releasing this amount we call `unHoldStableToken` the Issue Here is in `unHoldStableToken` function lets have a look:
```solidity
function unHoldStableToken(Props storage self, address stableToken, uint256 amount) external {
        // @audit : it will revert on holdAmount is less and also revert in case holdAmount>=amount
@>       require(self.stableTokenBalances[stableToken].holdAmount < amount, "sub hold bigger than hold");
        PoolTokenUpdateEventCache memory cache = _convertBalanceToCache(
            self.stakeToken,
            stableToken,
            self.stableTokenBalances[stableToken]
        );
        self.stableTokenBalances[stableToken].holdAmount -= amount;
        cache.holdAmount = self.stableTokenBalances[stableToken].holdAmount;
        _emitPoolUpdateEvent(cache);
    }
```
In above code we requires that the holdAmount must be less than amount we are going to subtract, it will fail on ` underflow or overflow`. 
```solidity
// to pass require statement we assume that holdAmount= 10e18 and amount 12e18 
uint256 holdAmount = 10e18;
uint256 amount = 12e18;

holdAmount = holdAmount - amount
Traces:
  [345] 0xBd770416a3345F91E4B34576cb804a576fa48EB1::run()
    └─ ← panic: arithmetic underflow or overflow (0x11)

⚒️ Chisel Error: Failed to execute REPL contract!
```
paste above code in `chisel` it will revert.

## Impact
The `holdAmount` for stable token will never be subtracted.

## Code Snippet
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/LpPool.sol#L307](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/LpPool.sol#L307)
## Tool used

Manual Review

## Recommendation
change the require statement
```diff
diff --git a/elfi-perp-contracts/contracts/storage/LpPool.sol b/elfi-perp-contracts/contracts/storage/LpPool.sol
index fd4e87a0..42564874 100644
--- a/elfi-perp-contracts/contracts/storage/LpPool.sol
+++ b/elfi-perp-contracts/contracts/storage/LpPool.sol
@@ -304,7 +304,7 @@ library LpPool {
     }
 
     function unHoldStableToken(Props storage self, address stableToken, uint256 amount) external {
-        require(self.stableTokenBalances[stableToken].holdAmount < amount, "sub hold bigger than hold");
+        require(self.stableTokenBalances[stableToken].holdAmount >= amount, "sub hold bigger than hold");
         PoolTokenUpdateEventCache memory cache = _convertBalanceToCache(
             self.stakeToken,
             stableToken,

```

