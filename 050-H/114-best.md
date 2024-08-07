Active Punch Jellyfish

High

# Deleveraging can result in a zero borrowed amount while maintaining the leveraged position

## Summary
Positions borrow the leverage amount from the pool and pay a borrowing fee for doing so. If the token price changes between the leverage update requests, an account can end up having leverage without any borrowed amount.
## Vulnerability Detail
Assume Alice has:

$100 10x Isolated LONG on tokenA, traded at $1. Alice's position is worth $1000, meaning she borrowed 900 tokenA, with a `holdAmount` of 900.

Some time passes, and tokenA drops to $0.40. Alice decides to deleverage to 2x. Since her position is isolated and the quantity won't change, the new margin required is $500. Alice previously had $100, so she needs to add $400 worth of tokenA, which is 1000 tokenA. 

1000 tokenA should be unheld from the pool. However, due to these lines in `PositionMarginProcess::_executeAddMargin`, it will only unhold the maximum amount, which is 900:

```solidity
function _executeAddMargin(Position.Props storage position, AddPositionMarginCache memory cache) internal {
    //..
    uint256 subHoldAmount = cache.addMarginAmount.min(position.holdPoolAmount);
    position.holdPoolAmount -= subHoldAmount;
    LpPoolProcess.updatePnlAndUnHoldPoolAmount(cache.stakeToken, position.marginToken, subHoldAmount, 0, 0);
}
```

Since 1000 tokenA is over the borrowed amount, only 900 tokens will be unheld from the pool, leaving the position with no `holdAmount`. Consequently, Alice's position will no longer incur borrowing fees. Overall, she provided a total of $500 worth of tokens, but her position is now worth $1000 in unchanged quantity.
## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L91-L132

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L340-L368
## Tool used

Manual Review

## Recommendation
