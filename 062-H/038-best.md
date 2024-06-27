Active Punch Jellyfish

High

# Profit/loss are not correctly calculated for users and pools

## Summary
When positions are closed in profits or losses the users and pools portion are calculated wrong. 
## Vulnerability Detail
Assume the following position:
LONG, 100$ margin, 5x leverage on token TAPIR which the price of 1 TAPIR is 1$:

orderMargin = 100 TAPIR
orderMarginFromBalance = 100 TAPIR

BEGINNING CROSS VALUES:
balance.amount = 100
balance.usedAmount = 100

openFee = 2 TAPIR
balance.usedAmount = 98
balance.amount = 98

increaseMargin = 98 TAPIR
increaseMarginFromBalance = 98 TAPIR
increaseQty = 490$ (98*5)

initialMargin = 98 TAPIR
initialMarginInUsd = 98$
initialMarginInUsdFromBalance = 98$
closeFeeInUsd = 2$
realizedPnl = -2$
holdPoolAmount = 392 TAPIR

Now, assume we are closing the position when the price hits 1.1$:
Price is 1.1$ close entire pos

totalPnlInUsd = +49$ (490 * (1.1-1)) 

Assume the following fees accrued to our position:
settledBorrowingFee = 4 TAPIR
settledFundingFee = 4 TAPIR
closeFee = 2 TAPIR (normally this is in USD but I converted it to TAPIR value for clarity) 
settledFee = 10 TAPIR (4 + 4 + 2) 

settledMargin = toUsd(98 - 10 * 1.1 + 49) = 123.63 TAPIR 
recordPnlToken = 123.63 - 98 = 25.63 TAPIR
poolPnlToken = 98 - toToken(98 + 49) = -35.63 TAPIR

FOR TAPIR:
balance.amount -= 10 = 88
balance.usedAmount -= 98 = 0
balance.amount += 35.63 = 123.63

From stakeToken to portfolio vault 25.63 tokens sent

We can observe that such position yielded us 25.63 TAPIR tokens which is 25.63 * 1.1 = 28.193$. 

Simply, the settledFees were 10 TAPIR and our pnl was +49$. Which means our net profit should be the `pnl - all the fees`
49$ in TAPIR would be 49/1.1 = 44.54 TAPIR
Settled fees are 10 TAPIR. 
Net profit = 44.54 - 10 = 34.54 TAPIR!

However, we only received 25.63 TAPIR instead of 34.54 TAPIR
## Impact
Profits and losses are cutted they are not the actual values which makes the Elfi competing with other perpetuals harder. 
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L102-L112
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/IncreasePositionProcess.sol#L34-L131
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L206-L336
## Tool used

Manual Review

## Recommendation
