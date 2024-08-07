Soaring Tawny Fly

High

# The `lossFee` is simply added to the `commonData` and not reimbursed to the keeper, leading to potential losses for the keeper.

## Summary
The `lossFee` is simply added to the `commonData` and not reimbursed to the keeper, leading to potential losses for the keeper.
## Vulnerability Detail
The processExecutionFee function is designed to calculate and handle the execution fee required by the keeper and ensure that this fee is appropriately managed between the user and the keeper. The function also addresses scenarios where the actual gas cost exceeds or falls below the user's provided execution fee. Below is the implementation of the function:
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/GasProcess.sol#L17-L41

1. Execution Fee Calculation:

     * The function correctly calculates the gas used and the corresponding execution fee.
     * It accounts for both scenarios where the actual execution fee exceeds or is less than the user's provided fee.
     * https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/GasProcess.sol#L18-L19
     
2. Fee Adjustments:

    * If the actual execution fee exceeds the user's provided fee, the executionFee is capped at the userExecutionFee, and the difference is considered a lossFee(Which is also calculated wrong).
   * If the actual execution fee is less than the user's provided fee, the difference is treated as a refundFee.
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/GasProcess.sol#L22-L27

3. Transfer and Withdrawal Mechanisms:

    * The user's execution fee is transferred from the vault  using VaultProcess.transferOut.
    * The execution fee is withdrawn for the keeper using VaultProcess.withdrawEther.
    * Any refund fee is returned to the user's account via VaultProcess.withdrawEther.
    https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/GasProcess.sol#L28-L34

4. Handling Loss Fees:

     * The lossFee is added to a common data pool via CommonData.addLossExecutionFee.
     * There is no mechanism in the current implementation to return the lossFee back to the keeper, which might be a potential issue 
       as it could lead to unrecovered costs for the keeper.
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/GasProcess.sol#L38-L40

### Issue: 
The lossFee is simply added to the common data pool and not reimbursed to the keeper, leading to potential losses for the keeper.
## Impact

*  This could disincentivize keepers from participating, as they may incur losses without compensation.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/GasProcess.sol#L17-L41
## Tool used

Manual Review

## Recommendation
Implement a function to incentivize the keepers for there loss in execution fee.