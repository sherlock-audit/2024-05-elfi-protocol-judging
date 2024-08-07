Active Punch Jellyfish

High

# Keepers loss gas is never accounted

## Summary
When keepers send excess gas, the excess gas is accounted in Diamonds storage so that keeper can compensate itself later. However, losses are never accounted due to math error in calculating it.  
## Vulnerability Detail
Almost every facet uses the same pattern, which eventually calls the `GasProcess::processExecutionFee` function:

```solidity
function processExecutionFee(PayExecutionFeeParams memory cache) external {
    uint256 usedGas = cache.startGas - gasleft();
    uint256 executionFee = usedGas * tx.gasprice;
    uint256 refundFee;
    uint256 lossFee;
    if (executionFee > cache.userExecutionFee) {
        executionFee = cache.userExecutionFee;
        // @review always 0
        lossFee = executionFee - cache.userExecutionFee;
    } else {
        refundFee = cache.userExecutionFee - executionFee;
    }
    // ...
    if (lossFee > 0) {
        CommonData.addLossExecutionFee(lossFee);
    }
}
```

As we can see in the snippet above, if the execution fee is higher than the user's provided execution fee, then the execution fee is set to `cache.userExecutionFee`, and the loss fee is calculated as the difference between these two, which are now the same value. This means the `lossFee` variable will always be "0", and the loss fees for keepers will never be accounted for.
## Impact
In a scaled system, these fees will accumulate significantly, resulting in substantial losses for the keeper. Hence, labelling it as high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/GasProcess.sol#L17-L40
## Tool used

Manual Review

## Recommendation
