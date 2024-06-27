Rough Emerald Gazelle

Medium

# MarketProcess.updateMarketOI() fails to validate minOpenInterest < symbolConfig.longShortOiBottomLimit, as a result, minOpenInterest could be as zero.

## Summary
MarketProcess.updateMarketOI() fails to validate minOpenInterest < symbolConfig.longShortOiBottomLimit, as a result, minOpenInterest could be as zero.

## Vulnerability Detail

In order to ensure the health of the market, there is a check that ``minOpenInterest < symbolConfig.longShortOiBottomLimit``. 
during increasing/decreasing an order. 

Consider the flow: OrderFacet.executeOrder -> OrderProcess.executeOrder -> OrderProcess._executeDecreaseOrder -> DecreaseOrderProcess.decreasePosition -> MarketProcess.updateMarketOI.

Inside MarketProcess.updateMarket() there is a check for  ``minOpenInterest < symbolConfig.longShortOiBottomLimit`` at L:

[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/MarketProcess.sol#L129-L161](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/MarketProcess.sol#L129-L161)

However, the check does not work since after the check, no errror is raised, the code simply returns. 

```javascript
 uint256 minOpenInterest = longOpenInterest.min(shortOpenInterest);
            if (minOpenInterest < symbolConfig.longShortOiBottomLimit) {
                return;
            }
```
This means, violation will be ignored. 

## Impact
MarketProcess.updateMarketOI() fails to validate minOpenInterest < symbolConfig.longShortOiBottomLimit, as a result, minOpenInterest could be as zero.

## Code Snippet

## Tool used
Foundry 

Manual Review

## Recommendation
Raise an error when the violation condition is met. 
