Odd Misty Hare

High

# Incorrect Margin Token Check in `order.sol` allows users to place multiple short orders for the same symbol as long as they use different marginToken

## Summary
The `hasOtherShortOrders` function in the `Order.sol` allows users to bypass the restriction of having only one short position per symbol, leading to issues(check `Impact` section).

## Vulnerability Detail

The issue lies in the `hasOtherShortOrders` function. The function is meant to check if a user has any existing short orders for a given symbol, `isCrossMargin`, and `posSide`. However, the function includes an incorrect condition that checks if the `marginToken` of the order is different from the provided `marginToken` parameter:


[Look at this part of the code:](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/Order.sol#L110-L131)

```solidity
      function hasOtherShortOrders(
          uint256[] memory orders,
         bytes32 symbol,
         address marginToken,
          bool isCrossMargin
     ) external view returns (bool) {
         Order.Props storage self = load();
         for (uint256 i; i < orders.length; i++) {
              Order.OrderInfo storage orderInfo = self.orders[orders[i]];
              if (
                 orderInfo.symbol == symbol &&
------>    orderInfo.marginToken != marginToken &&
                 orderInfo.isCrossMargin == isCrossMargin &&
                 orderInfo.posSide == Order.PositionSide.INCREASE &&
                 orderInfo.orderSide == Order.Side.SHORT
             ) {
                  return true;
             }
         }
    return false;
}
```

This incorrect condition allows users to place multiple short orders for the same symbol as long as they use different `marginToken`s.

Then, the `hasOtherShortOrders` function is queried in the `createOrderRequest` function in the [`OrderProcess.sol` contract](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L53-L70):

```solidity
      if (
          params.posSide == Order.PositionSide.INCREASE &&
          params.orderSide == Order.Side.SHORT &&
--->   (PositionQueryProcess.hasOtherShortPosition(
              accountProps.owner,
              params.symbol,
              params.marginToken,
              params.isCrossMargin
          ) ||
              Order.hasOtherShortOrders(
                  accountProps.getAllOrders(),
                  params.symbol,
                  params.marginToken,
                  params.isCrossMargin
              ))
      ) {
          revert Errors.OnlyOneShortPositionSupport(params.symbol);
      }
```

The purpose of this check is to enforce the rule that a user can only have one short position for a given symbol. However, due to the issue in the `hasOtherShortOrders` function, the check becomes ineffective, allowing users to bypass the restriction.


## Impact
It allows users to bypass the restriction of having only one short position per symbol, potentially leading to unintended exposure and disruptions. A bad actor can easily take advantage of this loophole to create multiple short positions.


## Code Snippet

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/Order.sol#L110-L131

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L53-L70


## Tool used

Manual Review

## Recommendation

It's better to remove the ` orderInfo.marginToken != marginToken &&` condtion. When removed, the function will correctly check for the existence of any other short orders with the same symbol, regardless of the `marginToken`, and enforce the intended rule of allowing only one short position per symbol.

Like  the recommendation above, removing the line `orderInfo.marginToken != marginToken` entirely is the better solution to fix the issue in the `hasOtherShortOrders` function.

Also I thought if we change the condition to `orderInfo.marginToken == marginToken`, the function will only return `true` if there is another short order with the same `marginToken`.But this doesnt seem to be the intended behaviour of this function.

The purpose of the `hasOtherShortOrders` function is to check if there are any existing short orders for the given symbol, `isCrossMargin`, and `posSide`, regardless of the `marginToken`. By removing the line `orderInfo.marginToken != marginToken`, the function will correctly check for the existence of any other short orders based on the relevant criteria.

