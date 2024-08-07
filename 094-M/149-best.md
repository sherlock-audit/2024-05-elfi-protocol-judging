Odd Misty Hare

Medium

# Incorrect Handling of Execution Fee in `_validateUpdateLeverageExecutionFee` Function

## Summary

The `_validateUpdateLeverageExecutionFee` function in the `PositionFacet` contract has an issue that can lead to double deposits of execution fee under certain conditions.

## Vulnerability Detail
The issue lies in the condition check inside the [`_validateUpdateLeverageExecutionFee` function:](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/PositionFacet.sol#L271)

```solidity
if (params.isNativeToken && params.addMarginAmount >= params.executionFee && !params.isCrossMargin) {
    return (params.addMarginAmount - params.executionFee, true);
}
```

If `params.isNativeToken` is `true`, `params.addMarginAmount` is greater than or equal to `params.executionFee`, but `params.isCrossMargin` is also `true`, the function will not subtract the execution fee from the `addMarginAmount`. Instead, it will proceed to deposit the execution fee separately using `AssetsProcess.depositToVault`.

This means that the execution fee will be deposited twice:
1. Once as part of the `addMarginAmount`.
2. Once separately as the `executionFee`.

Let's look at this scenario:

1. **Initial Conditions:**
   - `params.isNativeToken` is `true`.
   - `params.addMarginAmount` is `5 ETH`.
   - `params.executionFee` is `1 ETH`.
   - `params.isCrossMargin` is `true`.

2. The function `_validateUpdateLeverageExecutionFee` is called with the above parameters.

3. **Execution:**
   - The condition `params.isNativeToken && params.addMarginAmount >= params.executionFee && !params.isCrossMargin` evaluates to `false` because `params.isCrossMargin` is `true`.
   - The function proceeds to deposit the execution fee of `1 ETH` separately using `AssetsProcess.depositToVault`.
   - The function returns `(params.addMarginAmount, false)`, which is `(5 ETH, false)`.

4. **Result:**
   - The execution fee of `1 ETH` is deposited twice:
     - Once as part of the `addMarginAmount` (5 ETH).
     - Once separately as the `executionFee` (1 ETH).
   - This leads to an unintended double deposit of the execution fee.

## Impact

Breaks accounting. This can lead to the execution fee being deposited twice, resulting in incorrect accounting and potential loss of funds for the contract or the user.


## Code Snippet

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/PositionFacet.sol#L271

## Tool used

Manual Review

## Recommendation

The condition in the `_validateUpdateLeverageExecutionFee` function should be tweaked to ensure that the execution fee is only deposited once. 

```diff

-        if (params.isNativeToken && params.addMarginAmount >= params.executionFee && !params.isCrossMargin) {
+        if (params.isNativeToken && params.addMarginAmount >= params.executionFee && (!params.isCrossMargin || params.addMarginAmount == params.executionFee)) {
             return (params.addMarginAmount - params.executionFee, true);
         }
```

This checks if `params.isCrossMargin` is `true`, the `addMarginAmount` should be equal to the `executionFee` to avoid depositing the fee twice. If `params.isCrossMargin` is `false`, the original condition still applies.

