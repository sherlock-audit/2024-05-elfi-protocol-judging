Active Punch Jellyfish

Medium

# Decreasing positions with market orders will not be executable if `acceptablePrice` is provided by user

## Summary
Positions can be decreased via MARKET or STOP orders. If users choose to decrease via MARKET, the decrease order might revert or act unexpectedly because of the conflict between the market price and the acceptable price. 
## Vulnerability Detail
Assume user Bob has a LONG ETH position that he opened when the price of ETH was 3000$. 

After sometime ETH price goes up to 3100$. Bob wants to close the position entirely and realize the profits. 

Bob wants to do it with MARKET and does not want the position to be closed any price lower than 3090$. Hence, Bob crafts the order as:
Type: DECREASE
Side: SHORT(has to be opposite of the position)
acceptablePrice: 3090$.

The transaction will revert because of the following checks in the `_getExecutionPrice()` internal function:
```solidity
if (Order.Type.MARKET == order.orderType) {
            uint256 indexPrice = OracleProcess.getLatestUsdUintPrice(indexToken, isMinPrice);
            -> if (
                (isMinPrice && order.acceptablePrice > 0 && indexPrice < order.acceptablePrice) ||
                (!isMinPrice && order.acceptablePrice > 0 && indexPrice > order.acceptablePrice)
            ) {
                revert Errors.ExecutionPriceInvalid();
            }
            return indexPrice;
        }
```
Since the order requested by Bob is DECREASE/SHORT the `isMinPrice` variable will be `false`
`indexPrice` will be 3100$ (current ETH price)
`order.acceptablePrice` will be 3090$. 

Hence the if check will revert. 

## Impact
Orders can be successfully requested by users. Checks are allowing such orders to happen. However, when keeper try to execute them the transaction will revert and users needs to cancel their order. Since this type of orders are not forbidden when creating the request and fails while executing. It can be interpreted that this is a core contracts mislogic hence, I am labelling this as medium.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/49c4ba456d54810bfe36e44be9476acb9f0b23a1/contracts/process/OrderProcess.sol#L293-L351
## Tool used

Manual Review

## Recommendation
Don't allow such orders to be requested in first place. Let users do the same operation via STOP type.