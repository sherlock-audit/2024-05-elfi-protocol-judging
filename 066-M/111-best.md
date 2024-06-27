Soaring Tawny Fly

Medium

# Opening Long Positions with Index Tokens Not Supported.

## Summary
Users can open a long position in ELFI by providing margin tokens. However, upon examining the execution of the `_executeIncreaseOrder` function, it appears that the intention is to allow users to open a long position by providing the index token of the market as margin. Despite this intention, there exists a check within the function that prohibits such behavior

## Vulnerability Detail

`OrderFacet:executeOrder()` is responsible for executing orders submitted by users. For example, when a user submits an order for a long position, the execution flow is as follows: `OrderFacet:executeOrder() -> OrderProcess:executeOrder() -> OrderProcess:_executeOrder()`. 

Within `OrderProcess:_executeOrder()`, there is a check:
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L139-L142

This check ensures that if the `marginToken` is the same as the `indexToken`, then the marginTokenPrice will be set to the same value as the indexTokenPrice. This allows users to provide margin using the index token. And this clears the intention of protocol that user can provide `indexToken` as a `marginToken`

However, the `_validExecuteOrder()` function, which is called inside `executeOrder()`, contains another check:
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L102-L112

Specifically, within` _validExecuteOrder()`, there is a conditional statement that checks if the order is an increase and is a long position:
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L326-L333

This check, `isLong && order.marginToken != symbolProps.baseToken`, contradicts the previous condition that allows a user to provide margin using the index token. As a result, users will not be able to open a long position with the margin provided as the index token, which adversely affects the intended behavior of the protocol.
## Impact
1. Users will not be able to open long position by providing margin in `indexToken` affecting the intended behavior of the protocol.
## Code Snippet

## Tool used
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L326-L333
Manual Review

## Recommendation
Modify this check `isLong && order.marginToken != symbolProps.baseToken` so that user can provide margin in also `indexToken`