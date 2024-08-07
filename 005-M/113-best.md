Brief Chambray Swan

High

# Contract will reach a point where users will not be able to call `deposit`

## Summary
With the contract working as intended, after a long enough period of time the perceived amount of collateral for a specific point will cross `collateralTotalCap` resulting in the fact that users will not be able to deposit more tokens of that type.

## Vulnerability Detail
When `deposit` is called, after the necessary checks the method `commonData.addTradeTokenCollateral` is called which increases the total amount of collateral for a specific token in order to track how much collateral of this type of token exists and so that it does not cross the `collateralTotalCap`. 

On the other hand, the function `subTradeTokenCollateral` which is used to reduce the amount of total collateral per token is never called anywhere in the project resulting in an inaccurate value for `self.tradeCollateralTokenDatas[token].totalCollateral` as it tracks the amount of tokens that have entered the contract and not how many tokens are currently present inside the vault of the contract. And because there is a check weather the calling `deposit` would pass this cap it will eventually make it so that calling `deposit` with specific tokens would revert every time.

## Impact
Eventual denial of service for `deposit`

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/AssetsProcess.sol#L81-L120

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/storage/CommonData.sol#L74-L84

## Tool used
Manual Review

## Recommendation
Call `subTradeTokenCollateral` when the amount of collateral is being reduced (e.g. calling `withdraw`)