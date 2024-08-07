Energetic Lemonade Woodpecker

High

# Loss of Funds Due Caused by _settleCrossAccount incorrectly Accounting PositionMarginFromBalance

## Summary
When a cross position is being decreased (`DecreasePositionProcess::decreasePosition`), and `_settleCrossAccount` is called to balance the books (balance and liability) of the account, the margin from balance is updated with an incorrect `changeToken` value.  

## Vulnerability Detail
When `DecreasePositionProcess::_settleCrossAccount` is called, it makes a call to `PositionMarginProcess::updateAllPositionFromBalanceMargin` to update the margin of all positions of an account. 
**Point I**
Firstly, let us look at what the  `PositionMarginProcess::updateAllPositionFromBalanceMargin` does when called.
The `updateAllPositionFromBalanceMargin` function updates the balance margin for all positions of a given account by iterating through each position, checking if it matches the specified token and if it is a cross-margin position, and adjusting the balance margin accordingly. The function takes in an int256 `amount` param which is added to `position.initialMarginInUsdFromBalance`. 
When the value is >0 it increases the  `position.initialMarginInUsdFromBalance` as seen when the called [during deposit](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/AssetsProcess.sol#L109-L115).

*Note the `repayAmount` for liabilities is deducted from the `param.amount` when called from `AssetsProcess::deposit`, indicating the payment of account debt, and difference should go to margin from balance*

When the value is <0 it decreases the  `position.initialMarginInUsdFromBalance` as seen when the called [during withdraw](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/AssetsProcess.sol#L148-L154).
*Note the negation of `params.amount` when called from  `AssetsProcess::withdraw`, indicating the amount should be withdrawn from the account balance. Also, note that this is a similar situation to when positions are decreased*

**Point II**
Now lets examine `changeToken`, which is the value passed into the `PositionMarginProcess.updateAllPositionFromBalanceMargin` function when called from `DecreasePositionProcess::_settleCrossAccount`. 
`changeToken` should be the difference between the old initial margin and settled margin(updated margin) `PositionMarginProcess.updateAllPositionFromBalanceMargin` is called, i.e. the borrowed margin. However, changeToken is sum of settled margin and the borrowed margin.
> Please check Code Snippet before continuing


**The Issue:**
The problem is that instead of adding just the difference of the initial margin and settled margin, it includes part of the initialMargin, the part included in settledMargin. As pointed out in Point I, when `PositionMarginProcess.updateAllPositionFromBalanceMargin` is called, it adds the `amount`  param to the `initialMarginFromBalance` of the position. 

## Impact
This leads to the over or under count of the `initialMarginFromBalance` of account positions, which effectively steals assets from the protocol to the user or the user to the protocol, depending on the bias direction of changeToken.

## Code Snippet
Here is the simple run down of the calculation for `changeToken` according to the formula [here](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L399-L405):
```js
	changeToken = usdToTokens(decreaseMarginFromBalance) + settledMargin - decreaseMargin

// Given the following constraints and definitions extracted from the code:
	decreaseMargin >= usdToToken(decreaseMarginInUsdFromBalance)
	decreaseMarginFromBalance == marginFromBalance
	settledMargin == aMarginFromBalance + aBorrowedMargin (factoring adjustment i.e. fees and PnL)
	decreaseMargin <= marginFromBalance + borrowedMargin

// changeToken calculation:
	changeToken = decreaseMarginFromBalance + settledMargin - decreaseMargin
	changeToken = marginFromBalance + aMarginFromBalance + aBorrowedMargin - marginFromBalance + borrowedMargin (assuming the whole margin being decreased)
	changeToken = aMarginFromBalance + aBorrowedMargin + borrowedMargin
	changeToken = settledMargin + borrowedMargin
```

changeToken is the sum of the updated margin and borrowed margin.

## Tool used
Manual Review

## Recommendation
`changeToken` should be the difference between the old initial margin and settled margin, instead of the settledMargin + borrowedMargin.