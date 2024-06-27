Narrow Olive Kookaburra

Medium

# The keeper will suffer continuing losses due to miss compensation for L1 rollup fees

## Summary
While keepers submits transactions to L2 EVM chains, they need to pay both L2 execution fee and L1 rollup fee. The current implementation only compensates the keeper based on L2 gas consumption, the keeper will suffer continuing losses due to miss compensation for L1 rollup fees.

## Vulnerability Detail
As shown of the ````Arbitrum```` and ````Base````(op-stack) docs:
https://docs.arbitrum.io/arbos/l1-pricing
https://docs.base.org/docs/fees/
https://docs.optimism.io/stack/transactions/fees#l1-data-fee
Each L2 transaction costs both L2 execution fee and L1 rollup/data fee (for submitting L2 transaction to L1)

But current implementation only compensates the keeper the L2 gas consumption (L19).
```solidity
File: contracts\process\GasProcess.sol
17:     function processExecutionFee(PayExecutionFeeParams memory cache) external {
18:         uint256 usedGas = cache.startGas - gasleft();
19:         uint256 executionFee = usedGas * tx.gasprice;
20:         uint256 refundFee;
21:         uint256 lossFee;
22:         if (executionFee > cache.userExecutionFee) {
23:             executionFee = cache.userExecutionFee;
24:             lossFee = executionFee - cache.userExecutionFee;
25:         } else {
26:             refundFee = cache.userExecutionFee - executionFee;
27:         }
28:         VaultProcess.transferOut(
29:             cache.from,
30:             AppConfig.getChainConfig().wrapperToken,
31:             address(this),
32:             cache.userExecutionFee
33:         );
34:         VaultProcess.withdrawEther(cache.keeper, executionFee);
35:         if (refundFee > 0) {
36:             VaultProcess.withdrawEther(cache.account, refundFee);
37:         }
38:         if (lossFee > 0) {
39:             CommonData.addLossExecutionFee(lossFee);
40:         }
41:     }

```


## Impact
The keeper will suffer continuing losses on each transaction

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/GasProcess.sol#L19

## Tool used

Manual Review

## Recommendation
Compensating  L1 rollup fee as references of the above ````Arbitrum```` and ````Optimism```` docs:
