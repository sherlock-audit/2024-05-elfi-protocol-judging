Active Punch Jellyfish

High

# Cross available value is not accounting the position fees

## Summary
Cross available value is the maximum margin that an account can open a position. This value currently subtracts if the account has any losses but not assumes the fees which can be negative as well. This makes the account have a greater maximum margin than it should be.
## Vulnerability Detail
Cross available value is calculated in AccountProcess::getCrossAvailableValue() function as follows:
```solidity
            (totalNetValue + cache.totalIMUsd + accountProps.orderHoldInUsd).toInt256() -
            totalUsedValue.toInt256() +
            (cache.totalPnl >= 0 ? int256(0) : cache.totalPnl) -
            (cache.totalIMUsdFromBalance + totalBorrowingValue).toInt256();
```

As we can observe in above code snippet if there is a negative PnL it is subtracted from the positions available cross value. The reason for this is that if the account has a negative PnL that means when the position is realized accounts net value will drop hence, it is critical to account anything that can/will drop the accounts net value such as sum PnL of the positions account has. However, this calculation missing a key factor that also can drop the accounts net value which is the fees; closeFee, borrowingFee and fundingFee. When the position is settled these fees will added on top of the PnL so it can be assumed that it will affect the users latest settled margin. 

**Textual PoC:**
Assume an account has
totalNetValue = 200
cache.totalIMUsd = 100
totalUsedValue = 100
totalBorrowingValue = 100
totalIMUsdFromBalance = 0
totalPnl = 0
totalFees = 20

the cross available value for this account would be:
(100 + 100 + 0) -
100 +
0 -
(0 + 100) =
= 100$

This means that account can open an another position with a margin of 100$. However, there are 20 fees to pay which if the account would've closed the position account would had 80$ cross available value! In an extreme case if the fees are very high like say 80 account can open a position while it's actually eligible to liquidations. 

Another case would be account withdrawing the cross available value which is 100$ worth of collateral although the position is already in -20$ which will make the position not fully collateralized.  
## Impact
Users can open positions with a greater margin than their actual total balance. Hence, high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/AccountProcess.sol#L127-L147
## Tool used

Manual Review

## Recommendation
Add the fees just like the PnL. If it's negative (funding fees) then don't add it, if it's positive subtract it from the total value.