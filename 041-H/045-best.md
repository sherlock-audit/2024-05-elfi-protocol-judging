Active Punch Jellyfish

High

# Long orders always pays lesser in fees while short orders always pays higher due to oracle pricing

## Summary
Long orders pay lower fees due to the inconsistent margin token price, while short orders incur higher fees. This discrepancy naturally makes long orders more incentivized and short orders more disincentivized.
## Vulnerability Detail
When a position is opened there are 3 fees to be charged:
1. closeFee
2. borrowingFee
3. fundingFee

When a LONG position closed fees are calculated with the oracle `min` price:
```solidity
function updateBorrowingFee(Position.Props storage position, address stakeToken) public {
        .
        position.positionFee.realizedBorrowingFee += realizedBorrowingFeeDelta;
        position.positionFee.realizedBorrowingFeeInUsd += CalUtils.tokenToUsd(
            realizedBorrowingFeeDelta,
            TokenUtils.decimals(position.marginToken),
            -> OracleProcess.getLatestUsdUintPrice(position.marginToken, position.isLong)
        );
        .
    }

function updateFundingFee(Position.Props storage position) public {
       .
       .
        int256 realizedFundingFee;
        if (position.isLong) {
            .
            position.positionFee.realizedFundingFeeInUsd += CalUtils.tokenToUsdInt(
                realizedFundingFeeDelta,
                TokenUtils.decimals(position.marginToken),
                -> OracleProcess.getLatestUsdPrice(position.marginToken, position.isLong)
            );
        } else {
            realizedFundingFee = CalUtils.usdToTokenInt(
                realizedFundingFeeDelta,
                TokenUtils.decimals(position.marginToken),
                -> OracleProcess.getLatestUsdPrice(position.marginToken, position.isLong)
            );
            .
            position.positionFee.realizedFundingFeeInUsd += realizedFundingFeeDelta;
        }
        .
    }
```

When the positions `position.positionFee.realizedFundingFeeInUsd` and `position.positionFee.realizedBorrowingFeeInUsd` updated then these values are used to calculate the total fees in USD and finally used to calculate users and pools profit/loss.

Assume that the position is fully closed then these lines will be used to calculate the `settledMargin` and `recordPnlToken`:
```solidity
                cache.settledMargin = CalUtils.usdToTokenInt(
                    cache.position.initialMarginInUsd.toInt256() - _getPosFee(cache) + pnlInUsd,
                    TokenUtils.decimals(cache.position.marginToken),
                    tokenPrice
                );
                cache.recordPnlToken = cache.settledMargin - cache.decreaseMargin.toInt256();
```
 
`_getPosFee(cache)` is the total sum of fees in USD and since for a LONG order this is calculated via oracles `min` price this will be the minimum value in USD. Hence, the `settledMargin` will be a higher value and `recordPnlToken` will be a higher value too. In the end LONG orders pays lesser borrowing fees to pool and lesser funding fees to shorts. Short orders are the opposite, they pay more fees to longs and long fees to pool in borrowing fees.


**Textual PoC**:

Assume tokenA LONG position is being closed, tokenA max price is 1$ and min price is 0.9$ in oracle.

5 tokens in borrowing fee and 5 tokens in funding fees are accrued. Since it's a long position, the fees will be calculated in USD as 5 * 0.9 = 4$ instead of 5*1 = 5$. Total fees to be paid excluding the closeFee will be 8$. 

Assume position `initialMarginInUsdis` 100$ and `pnlInUsd` is 50$. 

Settled margin will be: toToken(100 - 8 + 50) = 142 token (LONG positions uses the max price as execution price hence, `tokenPrice` is the max value)

However, if the same position would be a SHORT position then the fees would be 10$ instead of 8$. 
## Impact
Long orders pay less fees to shorts in funding fees and pays less borrowing fee and closing fee to pool where as short orders are the opposite, they pay more fees to all parties. This discrepancy creates a greater advantage for long orders since they pay lesser funding fees and receive higher funding fees. In a scaled system, this advantage will be a greater problem hence, I'll label it as high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/FeeProcess.sol#L76-L137

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L60-L65

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L206-L336
## Tool used

Manual Review

## Recommendation
When calculating the fees always use the higher price to accrue more fees to parties. Or use the lesser price for both. The key point is to use the same pricing for both long and shorts to keep them in same incentive.