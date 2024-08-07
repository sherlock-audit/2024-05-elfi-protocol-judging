Magnificent Syrup Tardigrade

High

# `cache.isLiquidation` will always be false , will create DoS to Liquidate Position

## Summary
When decreasing or liquidating a position, we check for an edge case where if the position is being `liquidated=true` and `cross-margin=false`,  `settleMargin<0` than we don;t revert. However, the function will revert because cache.isLiquidation is always false. Negating it will make it true, causing the function to throw an error.

## Vulnerability Detail
`decreasePosition` function can be used to decrease position or liquidate position. in case of liquidation we always wanted to liquidate position regardless of its position atate. 
that's why in some case if `settleMargin<0` we don't care about it because the position must be liquidated.
In liquidation flow we pass `isLiquidation=true` which will help to prevent the following error to occur
```solidity
    if (cache.settledMargin < 0 && !cache.isLiquidation && !position.isCrossMargin) {
           revert Errors.PositionShouldBeLiquidation();
       }
``` 
But when we check inside `_updateDecreasePosition` function the value for `isLiquidation` is not set due to which its default value (false) will be used in above if statement.
1. Bob wants to liquidate Alice position which is not cross Margin.
2. Bob will submit a transaction for liquidation.
3. After execution of `_updateDecreasePosition` if we have `settleMargin=-10;isCrossMargin=false`. `cache.isLiquidation=false` Because its values was not set.
4. The `PositionShouldBeLiquidation` Error will be fired.
5. Hence Alice position can not be liquidated.

## Impact
The position can not be liquidated because of wrong `cache.isLiquidation` value being used.

## Code Snippet
[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L75](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L75)
## Tool used

Manual Review

## Recommendation
User the value from function arguments or set `cache.isLiquidation` value before above check.
