Active Punch Jellyfish

Medium

# If stable tokens depeg, short funding fees will not be accounted properly

## Summary
Funding fees are calculated using a Masterchef-like per token approach. In long orders, the per token calculation uses the token denomination. However, in short orders, it uses the USD value instead of the token value. If the stable token depegs, either temporarily or indefinitely, the funding fees for short positions will not be accounted for correctly.
## Vulnerability Detail
Short funding fee per qty is denominated in USD terms as we can observe in `MarketQueryProcess::getUpdateMarketFundingFeeRate` function as follows:
```solidity
if (cache.totalLongOpenInterest > 0) {
            cache.currentLongFundingFeePerQty = cache.longPayShort
                ? cache.totalFundingFee.div(cache.totalLongOpenInterest)
                : _boundFundingFeePerQty(
                    cache.totalFundingFee.div(cache.totalLongOpenInterest),
                    cache.fundingFeeDurationInSecond
                );
                // USD to token conversion
            -> cache.longFundingFeePerQtyDelta = CalUtils
                .usdToToken(
                    cache.currentLongFundingFeePerQty,
                    TokenUtils.decimals(symbolProps.baseToken),
                    OracleProcess.getLatestUsdUintPrice(symbolProps.baseToken, true)
                )
                .toInt256();
            cache.longFundingFeePerQtyDelta = cache.longPayShort
                ? cache.longFundingFeePerQtyDelta
                : -cache.longFundingFeePerQtyDelta;
        }
        if (cache.totalShortOpenInterest > 0) {
            // does not converts to USD amount to any stable token 
            cache.shortFundingFeePerQtyDelta = cache.longPayShort
                ? -_boundFundingFeePerQty(
                    cache.totalFundingFee.div(cache.totalShortOpenInterest),
                    cache.fundingFeeDurationInSecond
                ).toInt256()
                : (cache.totalFundingFee.div(cache.totalShortOpenInterest)).toInt256();
        }
```

Whenever user interacts with the protocol and update its position, the funding fees will be realized according to the latest per token and the users per token value until his latest interaction as we can observe in `FeeProcess::updateFundingFee` function:
```solidity
function updateFundingFee(Position.Props storage position) public {
        .
        -> int256 realizedFundingFeeDelta = CalUtils.mulIntSmallRate(
            position.qty.toInt256(),
            (fundingFeePerQty - position.positionFee.openFundingFeePerQty)
        );
        int256 realizedFundingFee;
        if (position.isLong) {
            realizedFundingFee = realizedFundingFeeDelta;
            position.positionFee.realizedFundingFee += realizedFundingFeeDelta;
            position.positionFee.realizedFundingFeeInUsd += CalUtils.tokenToUsdInt(
                realizedFundingFeeDelta,
                TokenUtils.decimals(position.marginToken),
                OracleProcess.getLatestUsdPrice(position.marginToken, position.isLong)
            );
        } else {
             // funding fee in form of USD so we convert it to token here for SHORT
            -> realizedFundingFee = CalUtils.usdToTokenInt(
                realizedFundingFeeDelta,
                TokenUtils.decimals(position.marginToken),
                OracleProcess.getLatestUsdPrice(position.marginToken, position.isLong)
            );
            -> position.positionFee.realizedFundingFee += realizedFundingFee;
            -> position.positionFee.realizedFundingFeeInUsd += realizedFundingFeeDelta;
        }
        .
    }
```

Now, considering the above, let's do a scenario where things can go wrong. 
Assume Bob opens a short position and by the time he opened the position DAI value was 1$ and his per token value is "X".

After 2 months, DAI depegs to 0.9$ for a month. In this time period since the short funding fee rates are USD based it will only update the per qty delta in USD terms. If Bob would've close or update his position he would get an unfair advantage since for 2 months DAI was 1$ and now it is 0.8$ and when position is updated it will use the latest price which is 0.8$.

So if the value for this above is 100$
```solidity
 int256 realizedFundingFeeDelta = CalUtils.mulIntSmallRate(
            position.qty.toInt256(),
            (fundingFeePerQty - position.positionFee.openFundingFeePerQty)
        );
```
His `realizedFundingFee` will be calculated as 100 / 0.8 = 125 DAI which would not be perfectly accurate because for 2 months it was 1:1 and now its 1:0.8.
```solidity
realizedFundingFee = CalUtils.usdToTokenInt(
                realizedFundingFeeDelta,
                TokenUtils.decimals(position.marginToken),
                OracleProcess.getLatestUsdPrice(position.marginToken, position.isLong)
            );
```

Assume Bob didn't close the position and let it live for another month, during which the DAI peg was restored and it's back to $1. Now, assume Bob closes the position and `realizedFundingFeeDelta` is 105, which means `realizedFundingFee` is also 105. This wouldn't be correct either because, for a month, DAI was depegged and Bob kept his position. He should receive more DAI in settlement from funding fees for that time interval due to the depegged period. 

Overall, if the stable tokens are not always 1$, funding fees will not be calculated correctly. 
## Impact
If stable tokens depegs funding fees will not accrue fairly. Also it seriously encourages shorters to close their position if they're funding fees are in profits and encourages shorters funding fees are in profit to stay. I'd say this is a mislogic in core function so labeling medium.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MarketQueryProcess.sol#L110-L161

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/FeeProcess.sol#L102-L137
## Tool used

Manual Review

## Recommendation
Acknowledge this or get the average token price for stable tokens and use it as the denomination for the per token value.