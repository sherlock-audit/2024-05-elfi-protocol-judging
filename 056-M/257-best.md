Magnificent Syrup Tardigrade

Medium

# `_executeDecreaseOrder` pass `isCrossMargin=false` is not correct crossMargin  value

## Summary
While processing the decreasef position order we passed `isCrossMargin=false` with out considering that the position `isCrossMargin=true`. 

## Vulnerability Detail
While in case of `autoReducePositions` we passed the correct value from `position.isCrossMargin`. 
```solidity
function _executeDecreaseOrder(
        uint256 orderId,
        Order.OrderInfo memory order,
        Symbol.Props memory symbolProps
    ) internal {
        address account = order.account;
        bool isLong = Order.Side.LONG == order.orderSide;

        Position.Props storage position = Position.load(account, order.symbol, order.marginToken, order.isCrossMargin);
        position.checkExists();

        if (position.isLong == isLong) {
            revert Errors.DecreaseOrderSideInvalid();
        }

        if (position.qty < order.qty) {
            order.qty = position.qty;
        }

        MarketProcess.updateMarketFundingFeeRate(symbolProps.code);
        MarketProcess.updatePoolBorrowingFeeRate(symbolProps.stakeToken, position.isLong, order.marginToken);

        uint256 executionPrice = _getExecutionPrice(order, symbolProps.indexToken);

        position.decreasePosition(
            DecreasePositionProcess.DecreasePositionParams(
                orderId,
                order.symbol,
                false,
@>              false,
                order.marginToken,
                order.qty,
                executionPrice
            )
        );
        Account.load(order.account).delOrder(orderId);
        emit OrderFilledEvent(orderId, order, block.timestamp, executionPrice);
    }
```
## Impact
The Protocol will consider crossMargin position as isolate Position.  

## Code Snippet
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L414-L450](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L414-L450)
## Tool used

Manual Review

## Recommendation
pass correct value of `isCrossMargin` in `decreasePosition`
