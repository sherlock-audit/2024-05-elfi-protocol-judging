Active Punch Jellyfish

High

# When a position is closed, the execution fees for the canceled stop orders are lost for the user

## Summary
When a position is decreased fully or partially, all the stop orders for that particular position will be canceled. Normally, the cancel order flow returns the execution fee paid by the user. However, this type of cancellation does not do that. As a result, all the STOP_LOSS and TAKE_PROFIT order execution fees are lost for the user.
## Vulnerability Detail
When a position is decreased partially or in full, the DecreasePositionProcess::decreasePosition function will remove all the hanging close orders with the following line:
```solidity
CancelOrderProcess.cancelStopOrders(
            cache.position.account,
            symbolProps.code,
            cache.position.marginToken,
            cache.position.isCrossMargin,
            CancelOrderProcess.CANCEL_ORDER_POSITION_CLOSE,
            params.requestId
        );
```

As we can observe in CancelOrderProcess::cancelStopOrders function the order removed from the accounts storage and global storage:
```solidity
function cancelStopOrders(
        address account,
        bytes32 symbol,
        address marginToken,
        bool isCrossMargin,
        bytes32 reasonCode,
        uint256 excludeOrder
    ) external {
        .
        .
            if (
                orderInfo.symbol == symbol &&
                orderInfo.marginToken == marginToken &&
                Order.Type.STOP == orderInfo.orderType &&
                orderInfo.isCrossMargin == isCrossMargin
            ) {
                -> accountProps.delOrder(orderIds[i]);
                -> orderPros.remove(orderIds[i]);
               .
            }
        }
    }
```

When the order is removed from storage user can no longer cancel it and get back the execution fee. All the stop loss and take profit orders execution fees are lost for the user.
## Impact
When user decides to close positions they will lose the execution fees. It can interpreted as user mistake however, If the account is liquidated than it can't be users mistake and the execution fees are lost regardless.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L60-L204

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/CancelOrderProcess.sol#L64-L94
## Tool used

Manual Review

## Recommendation
Refund the execution fees as it's done in a normal cancel order