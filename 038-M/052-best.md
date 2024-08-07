Active Punch Jellyfish

High

# The `autoReducePositions` function is not updating the market funding and pool borrowing fees prior to closing the position

## Summary
When positions are decreased via `autoReducePositions` call, borrowing and funding fees are not updated which leads to inaccurate fee accounting for the users.
## Vulnerability Detail
The markets overall borrowing and funding rates are updated in the casual flow of decrease position via these function calls:
```solidity
function _executeDecreaseOrder(
        uint256 orderId,
        Order.OrderInfo memory order,
        Symbol.Props memory symbolProps
    ) internal {
        .
        .
        -> MarketProcess.updateMarketFundingFeeRate(symbolProps.code);
       ->  MarketProcess.updatePoolBorrowingFeeRate(symbolProps.stakeToken, position.isLong, order.marginToken);
}
```

This update is very crucial because it will accrue the interest up to the current time for both funding and borrowing fees, which are essential for determining users' and pools' profit/loss. However, as we can observe, the `autoReducePositions` function skips updating the rates and directly closes the position.
## Impact
Latest accrued interest will not be updated for the user and previous values will be used. User can get a higher profit or a lesser loss or completely opposites of these. This will also break the entire Masterchef alike flywheel and affect other users as well. Hence, high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/PositionFacet.sol#L196-L216
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L414-L451
## Tool used

Manual Review

## Recommendation
Add these two functions before calling the autoReducePositions:
```solidity
MarketProcess.updateMarketFundingFeeRate(symbolProps.code);
MarketProcess.updatePoolBorrowingFeeRate(symbolProps.stakeToken, position.isLong, order.marginToken);
```