Wide Mocha Starfish

High

# Improper implementation of the `PositionMarginProcess.updatePositionFromBalanceMargin()` function.

## Summary

The updates to position values are not based on the current price of the `marginToken`.

## Vulnerability Detail

As shown in the code at `L314` and `L318`, all calculations are based on percentages relative to the maximum values. They do not factor in the current price of the `marginToken`. Consequently, even if the current `marginToken` price is significantly lower than when the position was last updated, users can still update their position using the higher price.

```solidity
    function updatePositionFromBalanceMargin(
        Position.Props storage position,
        bool needSendEvent,
        uint256 requestId,
        int256 amount
    ) public returns (uint256 changeAmount) {
        if (position.initialMarginInUsd == position.initialMarginInUsdFromBalance || amount == 0) {
            changeAmount = 0;
            return 0;
        }
        if (amount > 0) {
314         uint256 borrowMargin = (position.initialMarginInUsd - position.initialMarginInUsdFromBalance)
                .mul(position.initialMargin)
                .div(position.initialMarginInUsd);
            changeAmount = amount.toUint256().min(borrowMargin);
318         position.initialMarginInUsdFromBalance += changeAmount.mul(position.initialMarginInUsd).div(
                position.initialMargin
            );
        } else {
322         uint256 addBorrowMarginInUsd = (-amount).toUint256().mul(position.initialMarginInUsd).div(
                position.initialMargin
            );
            if (position.initialMarginInUsdFromBalance <= addBorrowMarginInUsd) {
                position.initialMarginInUsdFromBalance = 0;
                changeAmount = position.initialMarginInUsdFromBalance.mul(position.initialMargin).div(
                    position.initialMarginInUsd
                );
            } else {
                position.initialMarginInUsdFromBalance -= addBorrowMarginInUsd;
                changeAmount = (-amount).toUint256();
            }
        }
        if (needSendEvent && changeAmount > 0) {
            position.emitPositionUpdateEvent(requestId, Position.PositionUpdateFrom.DEPOSIT, 0);
        }
    }
```

## Impact

Users can update their positions' `initialMarginInUsdFromBalance` values using a price higher than the current price of the `marginToken`.

## Code Snippet

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L303-L338

## Tool used

Manual Review

## Recommendation

The `PositionMarginProcess.updatePositionFromBalanceMargin()` function should be based on the current price of the `marginToken`.