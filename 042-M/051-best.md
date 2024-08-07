Active Punch Jellyfish

Medium

# `autoReducePositions` uses the wrong price

## Summary
Positions can be auto closed by keeper calling `autoReducePositions`. However, the price used in this function is not inline with the typical decrease position flow.
## Vulnerability Detail
When a position is decreased the min/max price calculated as follows in `OrderProcess::_getExecutionPrice()` internal function:
```solidity
if (Order.PositionSide.INCREASE == order.posSide) {
            isMinPrice = Order.Side.SHORT == order.orderSide;
        } else {
            isMinPrice = Order.Side.LONG == order.orderSide;
        }
```

If a LONG order is closed, the order side has to be SHORT and vice versa. Assuming a LONG position is about to be closed, the order side is SHORT. This means `isMinPrice` will be `DECREASE/SHORT = false`, which indicates we will use the max price of the oracle. The actual price is fetched in the following lines of the function as follows:

```solidity
uint256 indexPrice = OracleProcess.getLatestUsdUintPrice(indexToken, isMinPrice);
```

Since we will use the maximum price, this will be in favor of the user.

However, when the `PositionFacet::autoReducePositions` function closes the position, it uses the minimum price for closing LONG orders and vice versa for the SHORT orders.
```solidity
position.decreasePosition(
                DecreasePositionProcess.DecreasePositionParams(
                    requestId,
                    position.symbol,
                    false,
                    position.isCrossMargin,
                    position.marginToken,
                    position.qty,
                    -> OracleProcess.getLatestUsdUintPrice(position.indexToken, position.isLong) // isLong is true hence, we get the min price
                )
            );
```

In the end users position will be closed in a worse pricing than a typical decrease position flow
## Impact
Auto reducing will always close the positions worse than a normal decrease position flow. It is not inconsistent and not favorable of user in any case. Hence, I'll label this as medium.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/PositionFacet.sol#L196-L216

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L236-L271
## Tool used

Manual Review

## Recommendation
Change to this:
`OracleProcess.getLatestUsdUintPrice(position.indexToken, !position.isLong)`