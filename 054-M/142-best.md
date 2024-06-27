Narrow Olive Kookaburra

Medium

# Missing compensation for the ````21,000```` intrinsic gas cost

## Summary
Every EVM transaction (on both L1 and L2) has an immediate ````21,000```` intrinsic gas cost, it's charged before any execution of smart contract code. The current impletmentation is missing to compensate this portion of gas cost, the keeper would suffer lost on each transaction.

Reference: https://stackoverflow.com/questions/50827894/why-does-my-ethereum-transaction-cost-21000-more-gas-than-i-expect

## Vulnerability Detail
The current ````startGas````(L67) can't account for the ````21,000```` intrinsic gas cost.
```solidity
File: contracts\facets\OrderFacet.sol
66:     function executeOrder(uint256 orderId, OracleProcess.OracleParam[] calldata oracles) external override { 
            // @audit at least 21,000+ gas is consumed before this line
67:         uint256 startGas = gasleft(); 
68:         RoleAccessControl.checkRole(RoleAccessControl.ROLE_KEEPER);
69:         Order.OrderInfo memory order = Order.get(orderId);
70:         if (order.account == address(0)) {
71:             revert Errors.OrderNotExists(orderId);
72:         }
73:         OracleProcess.setOraclePrice(oracles);
74:         OrderProcess.executeOrder(orderId, order);
75:         OracleProcess.clearOraclePrice();
76:         GasProcess.processExecutionFee(
77:             GasProcess.PayExecutionFeeParams(
78:                 order.isExecutionFeeFromTradeVault
79:                     ? IVault(address(this)).getTradeVaultAddress()
80:                     : IVault(address(this)).getPortfolioVaultAddress(),
81:                 order.executionFee,
82:                 startGas,
83:                 msg.sender,
84:                 order.account
85:             )
86:         );
87:     }


```


## Impact
The keeper will suffer continuing 21,000 intrinsic gas losses on each transaction

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/OrderFacet.sol#L67

## Tool used

Manual Review

## Recommendation
```diff
File: contracts\facets\OrderFacet.sol
66:     function executeOrder(uint256 orderId, OracleProcess.OracleParam[] calldata oracles) external override { 

-67:         uint256 startGas = gasleft(); 
+67:         uint256 startGas = gasleft() + 30000; // @audit 21000 intrinsic gas plus 9000 extra gas for calldata and facet lookup in diamond fallback() function
```

