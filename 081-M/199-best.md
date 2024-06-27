Energetic Lemonade Woodpecker

High

# Attacker can extract more funding fees by blockstuffing

## Summary
Due to one of the multiplying factors for how much funding fee traders receive being reliant on `block.timestamp`, an attacker can manipulate the system in their favor by stretching out the `fundingFeeDurationInSecond`.

## Vulnerability Detail
**Relevant design details**
- longPayShort determine if shorts or long earns funding fee that has accrued for fundingFeeDurationInSecond.
- longPayShort is determined by comparing totalLongOpenInterest to totalShortOpenInterest; whichever is greater, pays the other.
- funding fee is updated when getUpdateMarketFundingFeeRate is called, which is only called during order execution or position liquidation.
- fundingFeeDurationInSecond is the time since the last funding fee was paid and it is a multiplying factor for totalFundingFee received by traders

Attacker can prevent the processing of orderExecution or liquidation Txs through blockstuffing, in order to extend the `fundingRatePerSecond` used as a multiply factor to calculate their funding fee, effectively stealing from their counter traders. It is conceivable for such an attack to be coordinated by a group of traders on the same side of a trade. By default, everyone on their side profits fairly from it, and can share the risk evenly. 
To take the attack a step further, attackers seize the blockspace until the set of txs in the mempool favors them, leaving their counter traders with little to no chance of accruing funding fees. The higher the `fundingRatePerSecond` the more attractive the risk to reward ratio for this attack becomes.

This is feasible because no matter the amount of open interest against their positions, it doesn't impact price movement as it is a perpetual market. Therefore the only cost of this attacker is the transaction fee for holding the blockspace, which is significantly cheap on layer2s. Additionally the more the open interest for the opposing side of their trade builds up, the more funding fee they receive, further incentiving them to continue the attack indefinitely.


## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MarketQueryProcess.sol#L124

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MarketQueryProcess.sol#L130-L133


## Impact
Loss of reward for traders on the opposing end of the attacker's trader and unfair reward distribution


## Tool used
Manual Review


## Recommendation
Implement the FundingFee such that the longer the `longPayShort` bias continues, the lesser the `fundingRatePerSecond`. Alternatively, work around having to rely critically on `block.timestamp` for calculating 