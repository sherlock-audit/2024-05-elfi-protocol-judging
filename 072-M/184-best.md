Energetic Lemonade Woodpecker

Medium

# Accounting error caused by getCrossAvailableValue undercounting the real value of CrossAvailableValue

## Summary
The protocol factors the `orderHoldInUsd` of an account which is an unfinalized value held by the account's active orders in the calculation of a cross available value, leading to the under appreaciation of a users acount value.


## Vulnerability Detail
When `AccountProcess::getCrossAvailableValue` is called to calculate the total available, it returns:
`totalNetValue` + `cache.totalIMUsd` + `accountProps.orderHoldInUsd).toInt256()` - `totalUsedValue.toInt256()` +`(cache.totalPnl >= 0 ? int256(0) : cache.totalPnl)` -  `(cache.totalIMUsdFromBalance + totalBorrowingValue).toInt256()`


**Key Definitions:**
- `totalNetValue`: is the available amount for all positions of an Account. It doesn't include doesn't include `usedAmount`. `totalNetValue` by default only contains cross positions because `tokenBalance.amount - tokenBalance.usedAmount == 0` for isolated positions. totalNetValue is in USD.
- `totalIMUsd`: is the total initial margin across all the cross positions. It is the sum of all `tokenUsedValue` and `accountProps.orderHoldInUsd`.
- `orderHoldInUsd`: orderHoldInUsd is the value held by the account's active orders, in USD,  for example, if there a active limit order,  the orderHoldInUsd will be > 0. `orderHoldInUsd` is added to `totalUsedValue` and `totalBorrowingValue` if `accountProps.orderHoldInUsd > 0`.
- `totalPnl`: is the total profit and loss (PnL) across all the cross positions of an account.
- `totalIMUsdFromBalance`: is the part of the total initial margin across all the cross positions that is covered by the users balance.
- `totalBorrowingValue`: is the part of the total initial margin across all the cross positions that is covered by the users balance. It is the sum of `tokenBalance.usedAmount - tokenBalance.liability - tokenBalance.amount` for each cross position.

**The issue:**
What the formula above intends to do is to sum the total value of a users account and subtract the portion that is in use. i.e. 
"Account Net balance - Portion of Account tied up in positiona and orders (i.e. Used Portion)". However, the issue lies in the following:
- `orderHoldInUsd` which is essentially addMargin for a new order yet to be executed is added to "Account Net balance" once, but added to the "Used Portion" twice via `totalUsedValue` and `totalBorrowingValue`, effectively under-counting the real cross available value. 


## Impact
The protocol constrain the purchase power of an account by undervalueing it.

## Code Snippet
`orderHoldInUsd` is added to the "Used Portion" twice via `totalUsedValue` and `totalBorrowingValue`.
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/AccountProcess.sol#L104-L105
```js
	function getCrossUsedValueAndBorrowingValue() public view returns (uint256, uint256) {
	// ...
        if (accountProps.orderHoldInUsd > 0) {
            totalUsedValue += accountProps.orderHoldInUsd;
            totalBorrowingValue += accountProps.orderHoldInUsd;
        }
        return (totalUsedValue, totalBorrowingValue);
	// ...
```


## Tool used

Manual Review


## Recommendation
The `accountProps.orderHoldInUsd` should not be factored in the evaluation of cross available value as its value is not finalized and can also be cancelled and revert, or the worse execution transaction meant to finalized the order (into a position) is reverted, leaving the `orderHoldInUsd` unchanged. To completely remove it's effect from the equation, it should be added twice to the "Account Net balance"
```diff
// Function snippet for buyToken
function buyToken() public {
-        emittedTokenWad += totalTokensForBuyers;
+	      uint256 initialEmittedTokenWad = emittedTokenWad;
	}
```
