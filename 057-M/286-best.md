Breezy Pearl Beetle

High

# Order Creation with Zero Margin Due to Incorrect Execution Fee Validation

## Summary
The vulnerability lies in the _validateGasFeeLimitAndInitialMargin function. If a user submits an order with an orderMargin equal to the executionFee, the function allows the creation of an order with zero margin. This bypasses checks that should prevent orders with zero margin from being submitted, potentially leading to a situation where invalid orders are processed.

## Vulnerability Detail
The condition inside _validateGasFeeLimitAndInitialMargin allows for a situation where the margin is reduced by the execution fee, potentially to zero:
```solidity
if (
    params.isNativeToken &&
    params.posSide == Order.PositionSide.INCREASE &&
    !params.isCrossMargin &&
    params.orderMargin >= params.executionFee
) {
    return (params.orderMargin - params.executionFee, true);
}
```
If `params.orderMargin` is equal to `params.executionFee`, the function returns zero for the order margin:
```solidity
return (params.orderMargin - params.executionFee, true);
```


## Impact
Orders with zero margin can be created breaking an invariant of the system.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L453-L481

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L47-L100

## Tool used

Manual Review

## Recommendation
Make sure that orderMargin is not equal to executionFee
