Energetic Lemonade Woodpecker

High

# Loss of Users Funds When Increasing Leverage


## Summary
Users `addMarginAmount` is lost to the tradeVault when updatePositionLeverage() is called to increase leverage


## Vulnerability Detail
When users create an update position leverage request, they may specify additional margin (`addMarginAmount`), to manage their risk while updating their leverage. The problem is the protocol assumes no addMarginAmount was added when users are increasing their leverage.
As seen [here](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L197) and here, `addMarginAmount` is factored in isolated and cross positions initialMargin when decreasing leverage. However, it isn't factored in when increasing leverage for isolated positions, despite being transferred to the `tradeVault` when the request was created.

## Impact
This leads to permanent loss of funds for users as the request is deleted after execution, leaving no record of their deposit.


## Code Snippet
`addMarginAmount` is factored in isolated and cross positions initialMargin when decreasing leverage
```js
	else { // when increasing leverage
		position.leverage = request.leverage;
		uint256 reduceMargin = position.initialMarginInUsd - CalUtils.divRate(position.qty, position.leverage);
		uint256 reduceMarginAmount = _executeReduceMargin(position, symbolProps, reduceMargin, false);
	}
```


## Tool used

Manual Review


## Recommendation
Factor in the `addMarginAmount` for a request.