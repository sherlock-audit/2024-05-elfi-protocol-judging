Energetic Lemonade Woodpecker

High

# Accounting error caused by Protocol incorrect assumption about order margin tokens

## Summary
The  `OrderProcess::_validatePlaceOrder(params)` assumes all created cross order are in stableToken, causing errors in its accounting.


## Vulnerability Detail

When `OrderProcess::createOrderRequest` is called to create a cross margin order, `OrderProcess::_validatePlaceOrder(params)` is first called to validate the user provided params. The only check the function does for cross margin positions is ensuring the `params.orderMargin` is greater or equal to the `minOrderMarginUSD`. 
The issue here is the caused by the function not checking to ensure that `marginToken` is a stableToken, before compare it to the minimum order margin in USD `minOrderMarginUSD`.


## Impact
Direct comparison of assets of different units can lead to over estimation or underestimation of accounts in the network.


## Code Snippet
The only check the function does for cross margin positions is ensuring the `params.orderMargin` is greater or equal to the `minOrderMarginUSD`. 
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L408-L410
```js
_validatePlaceOrder(IOrder.PlaceOrderParams calldata params) internal view {
	if (params.isCrossMargin && params.orderMargin < AppTradeConfig.getTradeConfig().minOrderMarginUSD) {
		revert Errors.PlaceOrderWithParamsError();
	}
}
```


## Tool used
Manual Review


## Recommendation
The function should validate what token type the marginToken is and convert to USD if needed before comparing it to `minOrderMarginUSD`
