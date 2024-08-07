Energetic Lemonade Woodpecker

High

# Transactions can be executed with stale prices using Blockstuffing

## Summary
An attacker can use blockstuffing to cause transactions to execute with stale prices in favor of their orders/positions.

## Vulnerability Detail
Because the protocol relies on the keeper to provide token prices whenever a request is to be executed, any delay in the processing of these transactions implies the utilization of stale prices when the transaction finally executes. This poses a risk to the protocol as an attacker can take this as an opportunity for risk-free trades. A malicious actor can utilize attack techniques such as blockstuffing to delay the processing of their order to take advantage of an old price. On chains like Arbitrum and Base, it is conceivable to buy multiple consecutive blocks by stuffing them with transactions using a high gas fee. This can be used to manipulate the protocols as executed transaction timing is sensitive and critical.


## Impact
Attack can exploit the protocol the extra risk-free profits.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/AccountFacet.sol#L48
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/OrderFacet.sol#L66
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/PositionFacet.sol#L61
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/PositionFacet.sol#L149
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/StakeFacet.sol#L72

## Tool used
Manual Review

## Recommendation
Use active oracles such as Chainlink for pricing and always check for the staleness of your price feed.
