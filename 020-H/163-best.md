Dazzling Leather Sidewinder

High

# Incorrect implementation of the `PositionMarginProcess.updatePositionFromBalanceMargin()` function.

## Summary

The `updatePositionFromBalanceMargin()` function does nothing when `position.initialMarginInUsd == position.initialMarginInUsdFromBalance && amount < 0`. However, in this case, the function should actually reduce the `initialMarginInUsdFromBalance` of the position.

## Vulnerability Detail

In the `updatePositionFromBalanceMargin()` function, when `amount < 0`, it should reduce the value of `initialMarginInUsdFromBalance` for the position. However, as shown at `L309`, the function does nothing when `position.initialMarginInUsd == position.initialMarginInUsdFromBalance && amount < 0`. Consequently, if users withdraw their assets, the margin amounts of the positions are not reduced accordingly. This results in users being able to utilize more tokens than they have deposited.

```solidity
    function updatePositionFromBalanceMargin(
        Position.Props storage position,
        bool needSendEvent,
        uint256 requestId,
        int256 amount
    ) public returns (uint256 changeAmount) {
309     if (position.initialMarginInUsd == position.initialMarginInUsdFromBalance || amount == 0) {
            changeAmount = 0;
            return 0;
        }
        [...]
    }
```

## Impact

As a result, users may be able to utilize more tokens than they have deposited.

## Code Snippet

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L303-L338

## Tool used

Manual Review

## Recommendation

The `PositionMarginProcess.updatePositionFromBalanceMargin()` function should be fixed as follows.

```diff
-       if (position.initialMarginInUsd == position.initialMarginInUsdFromBalance || amount == 0) {
+       if ((position.initialMarginInUsd == position.initialMarginInUsdFromBalance && amount > 0) || amount == 0) {
            changeAmount = 0;
            return 0;
        }
```