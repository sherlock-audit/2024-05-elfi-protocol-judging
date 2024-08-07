Orbiting Raisin Osprey

Medium

# Users can open cross position less than minOrderMarginUSD

## Summary
user with role config can set minOrderMarginUSD its mean any user cannot open cross position with orderMargin less than minOrderMarginUSD but its possible in some way when keeper want to execute an order [_executeIncreaseOrderMargin](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L273) function is responsible for [checking position is cross or not](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L280) and if position is cross after that check getCrossAvailableValue is negative or not if getCrossAvailableValue is negative after that [add order.orderMargin to getCrossAvailableValue](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L282) and store result in fixOrderMarginInUsd and if fixOrderMarginInUsd is greater than zero after that [replaces orderMargin with fixOrderMarginInUsd](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L287) but fixOrderMarginInUsd can be less minOrderMarginUSD

## Vulnerability Detail
For example:
1- Alice deposits 1 WEI of  WETH as collateral
2- Alice open a cross position with orderMargin 100 USD when ETH price is 381300000000(base on chainlink)
*** getCrossAvailableValue function returns -99999999999999996187 
totalNetValue = 3813(1 WEI) 
orderHoldInUsd = 100e18
totalUsedValue=100e18
totalBorrowingValue=100e18
totalIMUsd=0
totalIMUsdFromBalance = 0
crossAvailableValue = (totalNetValue + totalIMUsd + orderHoldInUsd) - totalUsedValue + totalPnl - (totalIMUsdFromBalance + totalBorrowingValue)
crossAvailableValue = (3813 + 0 + 100e18) - 100e18 + 0 - (0 + 100e18) = -99999999999999996187
fixOrderMarginInUsd = orderMargin + crossAvailableValue
fixOrderMarginInUsd = 100e18 + (-99999999999999996187) = 3813
orderMargin = fixOrderMarginInUsd
orderMargin = 3813
in result orderMargin is less than minOrderMarginUSD[10e18]
## Impact
Users can open cross position less than minOrderMarginUSD
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L287
## Tool used

Manual Review

## Recommendation
```diff
        if (order.isCrossMargin) {
            if (accountProps.getCrossAvailableValue() < 0) {
                
                console2.log('r22_accountProps.getCrossAvailableValue():', accountProps.getCrossAvailableValue());
                
                int256 fixOrderMarginInUsd = order.orderMargin.toInt256() + accountProps.getCrossAvailableValue();
                if (fixOrderMarginInUsd <= 0) {
                    revert Errors.BalanceNotEnough(account, marginToken);
                }
                accountProps.subOrderHoldInUsd(order.orderMargin);
+             if(fixOrderMarginInUsd < AppTradeConfig.getTradeConfig().minOrderMarginUSD)         
+                         revert Errors.PlaceOrderWithParamsError();   
                order.orderMargin = fixOrderMarginInUsd.toUint256();
```
