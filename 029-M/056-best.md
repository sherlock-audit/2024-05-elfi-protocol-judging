Active Punch Jellyfish

High

# Late liquidations of cross positions mistakenly update the insurance fund

## Summary
If a cross position is liquidated late, the insurance fund will update its balance incorrectly.
## Vulnerability Detail
Assume Bob has a 5x LONG 100$, 100 tokenA position whereas tokenA is 1$ at the time of opening the position.
Cross balances of Bob for tokenA:
balance.amount = 110
balance.used = 100
Assume after sometime tokenA drops to 0.9$ the loss for the Bob will be (500(qty) * 1-0.9) = -50$
Also assume that there are 40 tokenA settledFee (borrowing+funding+close).

A liquidator liquidates the account quickly and DecreasePositionProcess:decreasePosition will be called.
Following the execution of the function the internal `_updateDecreasePosition` functions following lines will be executed:
```solidity
if (isLiquidation) {
                cache.settledMargin = isCrossMargin
                    ? CalUtils.usdToTokenInt(
                        cache.position.initialMarginInUsd.toInt256() -
                            _getPosFee(cache) +
                            pnlInUsd -
                            PositionQueryProcess.getPositionMM(cache.position).toInt256(),
                        TokenUtils.decimals(cache.position.marginToken),
                        tokenPrice
                    )
                    : int256(0);
                cache.recordPnlToken = cache.settledMargin - cache.decreaseMargin.toInt256();
                cache.poolPnlToken =
                    cache.decreaseMargin.toInt256() -
                    CalUtils.usdToTokenInt(
                        cache.position.initialMarginInUsd.toInt256() + pnlInUsd,
                        TokenUtils.decimals(cache.position.marginToken),
                        tokenPrice
                    );
            }
```

Let's fill the values above according to our example. 
cache.decreaseMargin.toInt256 = 100 tokenA
cache.position.initialMarginInUsd.toInt256 = 100$
_getPosFee() = 40 tokens * 1.1 (price) = 44$
pnlInUsd = -50$
crossMM = 500(qty) / 40(maxLeverage * 2) = 12.5$

cache.settledMargin = toToken(100 - 44 - 50 - 12.5) = -5.9 tokenA
cache.recordPnlToken = -5.9 - 100 = -105.9 tokenA
cache.poolPnlToken = 100 - ((100 - 50) / 1.1) = 54.54 tokenA

Following the execution in `decreasePosition` the `_settleCrossAccount` function will be called. 
Since there are 40 fees these will be subtracted and 100 tokens will be unused from Bob's cross account:
balance.amount -= 40 = 70
balance.used = 0

Then, since the `recordPnlToken` is negative Bobs cross account will subtract the amount:
balance.amount -= 105.9 = -35.9
balance.used += 35.9 = 35.9
balance.liability += 35.9 = 35.9

Then the `-recordPnlToken - addLiability` amount of tokenA will be transferred from PortfolioVault to stakeToken:
105.9 - 35.9 = 70 tokens sent.

Then following the execution, `LpPoolProcess:updatePnlAndUnHoldPoolAmount` function will be called:
70 tokens will be added to pools base amount and 
35.9 tokens will be added to pools unsettled base amount.

Then back to our execution flow, `_addFunds` internal function will be called:
```solidity
function _addFunds(DecreasePositionCache memory cache) internal {
        if (cache.position.isCrossMargin) {
            -> InsuranceFund.addFunds(
                cache.stakeToken,
                cache.position.marginToken,
                CalUtils.usdToToken(
                    PositionQueryProcess.getPositionMM(cache.position),
                    TokenUtils.decimals(cache.position.marginToken),
                    cache.marginTokenPrice
                )
            );
            return;
        }
      .
}
```

If we would've calculated the settledMargin without the crossMM, the settledMargin would be:
(100 - 44 - 50) / 1.1 = 5.45 tokenA which means only 5.45 tokenA is available for user however, insurance fund is expecting all the 12.5$ worth of tokenA is available in users cross account and adds it accordingly! 

In the end 12.5$ worth of tokenA will be added to InsuranceFund. However, this is not correct because the liquidation were late and 12.5$ worth of tokenA shouldn't be added to insurance fund but instead the difference between the initial cross MM and what's left. 
## Impact
Although InsuranceFund is not in scope it is clear that the funds are added to InsuranceFunds which we don't really need the exact implementation. The key factor here is that all liquidations are on point than the above scenario described in vulnerability details will not happen. However, this wouldn't be a liquidator fault in a very volatile market and can happen naturally. Hence, high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L60-L204

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L206-L336

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionQueryProcess.sol#L138-L142

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L338-L414

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/LpPoolProcess.sol#L35-L69

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L446-L458
## Tool used

Manual Review

## Recommendation
calculate the settled margin without the crossMM. If the value is higher than the crossMM use crossMM value. If its lesser than it then add to Insurance fund only the diff, crossMM - whatsLeft