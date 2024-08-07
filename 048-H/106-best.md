Active Punch Jellyfish

High

# If the stake token is minted from portfolio vault, positions from balances are not decreased

## Summary
When the mint stakes is minted with the portfolio vault tokens (users cross balance tokens) the entire from balances must need to change since the tokens are no longer part of users cross portfolio. 
## Vulnerability Detail
As we can see in the normal withdrawal flow, if any token balance is withdrawn, it affects the current positions, updating all positions accordingly, as shown in the `withdraw` function:

```solidity
function withdraw(uint256 requestId, WithdrawParams memory params) public {
    // ...
    -> PositionMarginProcess.updateAllPositionFromBalanceMargin(
        requestId,
        params.account,
        params.token,
        -(params.amount.toInt256()),
        ""
    );
}
```

In the `MintProcess::executeMintStakeToken` function flow, token balances change if the stake token is minted from portfolio vault balances:

```solidity
function _transferFromAccount(address account, address token, uint256 needAmount) internal {
    Account.Props storage tradeAccount = Account.load(account);
    if (tradeAccount.getTokenAmount(token) < needAmount) {
        revert Errors.MintFailedWithBalanceNotEnough(account, token);
    }
    -> tradeAccount.subTokenIgnoreUsedAmount(token, needAmount, Account.UpdateSource.TRANSFER_TO_MINT);
    int256 availableValue = tradeAccount.getCrossAvailableValue();
    if (availableValue < 0) {
        revert Errors.MintFailedWithBalanceNotEnough(account, token);
    }
}
```

However, this change does not update the positions from balances. This results in incorrect cross available values for the users' positions.
## Impact
Since maintaining accurate balances is crucial to ensure a fair cross available value, and the above issue indicates that this balance will be disrupted, I will label it as high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MintProcess.sol#L68-L91

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MintProcess.sol#L264-L274

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MintProcess.sol#L130-L171
## Tool used

Manual Review

## Recommendation
Just like withdraw function loop over the positions and update the from balances