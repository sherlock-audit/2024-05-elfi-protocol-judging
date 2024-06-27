Active Punch Jellyfish

High

# Mismatching funding fees can result in the protocol incurring a deficit or insolvency risk

## Summary
When funding fees are calculated if it's high enough it can be capped for one side but the other side would not get the new adjusted funding fee. Result of it can end up for misplayed funding fees and even bad debt in some cases.  
## Vulnerability Detail
Assume there are:
$150 totalLongOpenInterest
$50 totalShortOpenInterest
10 seconds passed since the last interaction

Following the execution in `MarketQueryProcess::getUpdateMarketFundingFeeRate()` the following calculations will be done:

fundingRatePerSecond = 10^^-8

totalFundingFee = 150 * 10 * 10^^-8 =
= 1.5 * 10^^-5

currentLongFundingFeePerQty = 
1.5 * 10^^-5 / 150
= -10^^-7

shortFundingFeePerQtyDelta =  1.5 * 10^^-5 / 50 
3 * 10 ^^-7
Assume the max cap is 2 * 10 ^^-7:
We pick 2*10^-7
= 2*10^^-7

longFundingFeeRate = 
(-10^^-7 * 3600 / 10 )/ 10^^-5
= -3.6

shortFundingFeeRate =
(2*10^^-7 * 3600 / 10) / 10^^-5
= 7.2

For short position that holds 50:
realizedFundingFeeDelta =
50 * 2*10^^-7 =
= 10^^-5

For long position that holds 150:
realizedFundingFeeDelta = 
-150 * 10^^-7 =  
= -1.5 * 10^^-5

As we can observe, longs pay 1.5 and shorts receive 1. There is a discrepancy of 0.5 in funding fees that are not paid to short users.

This discrepancy can lead to insolvency because of how the pool accounts for its total holdings. The pool's total value is calculated as the amount plus `unsettledAmount`, where `unsettledAmount` is essentially the accrued funding fees. If the long's 1.5 fee is accounted for as unsettled, the contract assumes this 1.5 will be paid back to shorts, so the protocol is always counting correctly in the long run. However, this assumption is incorrect because shorts will not receive the 1.5 funding fee; they will only receive 1 in our case. Therefore, the excess "0.5" accounted in the pool's total value is incorrect because it will never be returned by the other party.

**Textual Proof of Concept:**
Assume the pool has 100 baseAmount and 10 unsettledFee, totaling 110 assets. Someone can open positions based on a value of 110, expecting that at the end of the day, when the unsettledFees are settled, 10 assets will be returned to the system. However, if the funding fee for the short party is capped, they will only receive 8 fees instead of 10. Consequently, the pool will incorrectly account for "2" assets.
## Impact
Miscounting of pools total value. Positions that are opened will think there is enough funds but actually these fees will never returned by the other party, resulting a position opened without proper collateral. Hence, high. 
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MarketProcess.sol#L29-L52

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MarketQueryProcess.sol#L110-L161

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L110-L145
## Tool used

Manual Review

## Recommendation
If the maximum is picked then adjust the counter party's funding fee accordingly. Always give out the same funding fees for both parties. If longs pays 10 then shorts should receive the 10 and vice versa
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MarketQueryProcess.sol#L110-L161