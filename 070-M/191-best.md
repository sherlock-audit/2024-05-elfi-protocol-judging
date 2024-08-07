Energetic Lemonade Woodpecker

Medium

# The protocol double charges settledFundingFee to users

## Summary
`settledFundingFee` which constitute part of the `cache.settledFee` is double charged to users.

## Vulnerability Detail
When `DecreasePositionProcess::decreasePosition` is called to decrease a cross position, `DecreasePositionProcess::_settleCrossAccount` is called to settle the balance and liabilities of the account. Part of this settlement includes repaying fees owed to the protocol by the account. One of such fees is the `settledFee`. 
 `settledFee` is the total fees to be paid by the account, and it's the sum of the `settledBorrowingFee`, `settledFundingFee`, and `closeFee` as seen [here](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L233-L236) and [here](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L301-L304). These fees are estimated by `DecreasePositionProcess::_updateDecreasePosition` and its payment recorded in `DecreasePositionProcess::decreasePosition` and `DecreasePositionProcess::_settleCrossAccount`. 
 The problem is with the payment of `settledFundingFee`, which double charge by the protocol. First [here](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L96) in the `DecreasePositionProcess::decreasePosition` when the cross position is being updated, and secondly [here](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L344-L350) in `DecreasePositionProcess::_settleCrossAccount` as `settledFee`. Note that the `settledFundingFee` is a constituent of `settledFee`.
 

## Impact
Loss for funds for users as the pay twices for a fee.


## Code Snippet
 `settledFee` is the sum of the `settledBorrowingFee`, `settledFundingFee`, and `closeFee` 
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L233-L236
```js
	cache.settledFee =
		cache.settledBorrowingFee.toInt256() +
		cache.settledFundingFee +
		cache.closeFee.toInt256();
```

SettledFundingFee is first charged in the `DecreasePositionProcess::decreasePosition` when the cross position is being updated.
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L96
```js
	{
		position.qty -= params.decreaseQty;
		position.initialMargin -= cache.decreaseMargin;
		position.initialMarginInUsd -= cache.decreaseMarginInUsd;
		position.initialMarginInUsdFromBalance -= cache.decreaseMarginInUsdFromBalance;
		position.holdPoolAmount -= cache.unHoldPoolAmount;
		position.realizedPnl += cache.realizedPnl;
		position.positionFee.realizedBorrowingFee -= cache.settledBorrowingFee;
		position.positionFee.realizedBorrowingFeeInUsd -= cache.settledBorrowingFeeInUsd;
		position.positionFee.realizedFundingFee -= cache.settledFundingFee;
		position.positionFee.realizedFundingFeeInUsd -= cache.settledFundingFeeInUsd;
		position.positionFee.closeFeeInUsd -= cache.closeFeeInUsd;
		position.lastUpdateTime = ChainUtils.currentTimestamp();
	}
```

Secondly, in `DecreasePositionProcess::_settleCrossAccount` as `settledFee`.
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L344-L350
```js
	if (cache.settledFee > 0) {
		accountProps.subTokenWithLiability(
			cache.position.marginToken,
			cache.settledFee.toUint256(),
			Account.UpdateSource.SETTLE_FEE
		);
	}
```


## Tool used
Manual Review


## Recommendation
Subtract `settledFundingFee` from `settledFee` before settling it in `DecreasePositionProcess::_settleCrossAccount`
```diff
if (cache.position.isCrossMargin) {
+ 	cache.settledFee -= cache.settledFundingFee;
	uint256 addLiability = _settleCrossAccount(params.requestId, accountProps, position, cache);
	accountProps.repayLiability(cache.position.marginToken);
```
