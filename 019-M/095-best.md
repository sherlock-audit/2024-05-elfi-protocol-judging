Soaring Tawny Fly

High

# The implementation of `payExecutionFee()` didn't take `EIP-150` into consideration. Keepers can steal additional execution fee from users.

## Summary
The implementation of `processExecutionFee()` didn't take `EIP-150` into consideration. Keepers can steal additional execution fee from users

## Vulnerability Detail
The issue arises on `L18` of `GasProcess.sol:processExecutionFee()`, as it's an external function, calling`processExecutionFee()` is subject to EIP-150.
Only `63/64` gas is passed to the `GasProcess` sub-contract(`external library`), and the remaning `1/64` gas is reserved in the caller contract which will be refunded to `keeper` after the execution of the whole transaction. But calculation of `usedGas `  includes this portion of the cost as well.

A malicious keeper can exploit this issue to drain out all execution fee, regardless of the actual execution cost.
Let's take `executeMintStakeToken()` operation as an example to show how it works:
```solidity
executionFeeUserHasPaid = 200K Gwei
tx.gasprice = 1 Gwei
actualUsedGas = 100K
```
`actualUsedGas` is the gas cost since `startGas`(L76 of `StakeFacet .sol`) but before calling `processExecutionFee()`(L88 of `StakeFacet.sol`)

Let's say, the keeper sets tx.gaslimit to make
```solidity
startGas = 164K
```
Then the calculation of `usedGas` , `L18` of `GasProcess.sol`, would be
```solidity
uint256 usedGas= cache.startGas- gasleft() = 164K - (164K - 100K) * 63 / 64 = 101K
```
and
```solidity
executionFeeForKeeper = 101K * tx.gasprice = 101K * 1 Gwei = 101K Gwei
refundFeeForUser = 200K - 101K = 99K Gwei
```
As setting of `tx.gaslimit` doesn't affect the actual gas cost of the whole transaction, the excess gas will be refunded to `msg.sender`. Now, the keeper increases `tx.gaslimit` to make `startGas = 6500K`, the calculation of `usedGas` would be
```solidity
uint256 usedGas= cache.startGas- gasleft() = 6500K - (6500K - 100K) * 63 / 64 = 200K
```
and
```solidity
executionFeeForKeeper = 200K * tx.gasprice = 200K * 1 Gwei = 200K Gwei
refundFeeForUser = 200K - 200K = 0 Gwei
```
We can see the keeper successfully drain out all execution fee, the user gets nothing refunded.
## Impact
Keepers can steal additional execution fee from users.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/GasProcess.sol#L18C17-L18C25
## Tool used

Manual Review

## Recommendation
```diff
    function processExecutionFee(PayExecutionFeeParams memory cache) external {
-        uint256 usedGas = cache.startGas - gasleft();
+       uint256 usedGas = cache.startGas - gasleft() * 64 / 63;
        uint256 executionFee = usedGas * tx.gasprice;
        uint256 refundFee;
        uint256 lossFee;
```