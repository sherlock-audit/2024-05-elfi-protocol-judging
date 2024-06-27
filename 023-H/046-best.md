Soaring Tawny Fly

High

# Vulnerability in Execution of Stop and Take Profit Orders: Orders Executed Before Reaching Trigger Price

## Summary
 The `_getExecutionPrice` in `OrderProcess` Library intended to give the price at which the order should executed. The vulnerability allows a take profit order to be executed at a price below the specified trigger price for long positions, which violates the intended behavior of take profit conditions.

## Vulnerability Detail
Inside `_getExecutionPrice` initially it checks for `isMinPrice`
```solidity
bool isMinPrice;
        if (Order.PositionSide.INCREASE == order.posSide) {
            isMinPrice = Order.Side.SHORT == order.orderSide;
        } else {
            isMinPrice = Order.Side.LONG == order.orderSide;
        }
```

The vulnerability lies in the code snippet below. This if block checks whether the order is either a limit order or a stop order with a take profit condition. If one of these conditions is met, the code inside the block will execute; otherwise, it will revert.

* Let's assume a case where a user wants to place a trade in the ETH/USD pool.
* The user places a long trade with a `Stop` and `takeProfitOrder`, setting the take profit limit at `$3000`. The current price of ETH is $2800.
* The condition `(Order.Type.STOP == order.orderType && Order.StopType.TAKE_PROFIT == order.stopType)` will be true.
* The next condition `(isLong && order.triggerPrice >= currentPrice)` will also be true since the position is long, and the `order.triggerPrice (3000)` is greater than the `currentPrice (2800)`.
* As this condition returns true, the `currentPrice` will be returned, causing the user's trade to be executed below the `takeProfitPrice`, resulting in a loss for the user.

```solidity 
        if (
            Order.Type.LIMIT == order.orderType ||
            (Order.Type.STOP == order.orderType && Order.StopType.TAKE_PROFIT == order.stopType)
        ) {
         
         if ((isLong && order.triggerPrice >= currentPrice) || (!isLong && order.triggerPrice <= currentPrice)) {
                return currentPrice;
            }
            revert Errors.ExecutionPriceInvalid();
        }
```
## Impact
Premature execution of take profit orders at unfavorable prices, potentially causing financial losses for users.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L255-L262
## Tool used

Manual Review

## Recommendation
* Execute an additional if block specifically for the execution of stop profit trades to ensure accurate order execution.