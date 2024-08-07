Active Punch Jellyfish

High

# Users profit in short cross will leave the fees in UsdPool instead of LpPool

## Summary
In order to account the fees properly, all fees should be collected in the corresponding LpPool. However, in one case the fees are left in the UsdPool.
## Vulnerability Detail
The correct flow of the fees tells that the fees must need to be in the LpPool after a position is closed/settled. 

User profit in cross LONG
User loss in cross LONG
User profit in isolated LONG
User loss in isolated LONG
User loss in cross SHORT
User profit in isolated SHORT
User loss in isolated SHORT

In all of these scenarios the fees are settled in the LpPool. However, 
**User profit in cross SHORT the fees are in the UsdPool instead of the LpPool!** 

**Textual PoC:**
**Create a cross order with the asset
SHORT position 100$ margin 5x lev on token TAPIR which the price is 1$:**

orderMargin = 100 USDC
orderMarginFromBalance = 100 USDC

FOR USDC:
balance.amount = 0
balance.usedAmount = 100

fee = 2 USDC
balance.usedAmount = 98
balance.amount = 98

increaseMargin = 98 USDC
increaseMarginFromBalance = 98 USDC
increaseQty = 490$

initialMargin = 98 USDC
initialMarginInUsd = 98$
initialMarginInUsdFromBalance = 98$
closeFeeInUsd = 2$
realizedPnl = -2$
holdPoolAmount = 392 USDC
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
**Price is 0.9$ close entire pos**

totalPnlInUsd = +49$

settledBorrowingFee = 4 tokens
settledFundingFee = 4 tokens
closeFee = 2 tokens
settledFee = 10 tokens

settledMargin = 153.3 USDC
recordPnlToken = 153.3 - 98 = 55.3 USDC
poolPnlToken = -65.3 USDC

FOR USDC:
balance.amount -= 10 = 88
balance.usedAmount -= 98 = 0
balance.amount += 65.3 = 153.3

**From UsdPool to portfolio vault 55.3 USDC sent**

**From UsdPool to stake token 2 USDC sent (closeFee)**

pool.lossAmount += 65.3 (USDC)
usdpool.amount -= 65.3
usdpool.unsettledAmount += 65.3

So basically, the 55.3 USDC sent from UsdPool to Portfolio for users profit. Then, 2 USDC which is only the close fee sent from UsdPool to LpPool. The remaining 8 USDC fees are still standing in the UsdPool but they should be also sent to the LpPool! 

## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L60-L204

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L338-L414
## Tool used

Manual Review

## Recommendation
Send the entire fees to LpPool not just the close fees in the case of user profit in short cross.