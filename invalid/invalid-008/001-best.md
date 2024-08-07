Glorious Onyx Condor

High

# Implementation did not account for Fee On Transfer tokens

## Summary
Elfi supports all ERC20 tokens including Fee-On-Transfer tokens such as USDT. The implementation did not account for Fee-On-Transfer.
## Vulnerability Detail
If the token is a fee-on-transfer token (e.g., USDT), the amount deposited into the vault is reduced by the fee, resulting in fewer tokens received and discrepancies in the user account balance. Additionally, for rebasing tokens, changes in value (increases or decreases) are not properly accounted for.
Example Scenario:
Let's imagine USDT in arbitrum started to charge fees 1% per transfer.

1) Alice wants to deposit 100 USDT.
2) Vault will receive only 99 USDT.
3) Alice has the ability to margin trade with 100 USDT and gained profit of 900 USDT
4) Alice tries to withdraw 1000 USDT, but not able to do so due to the 1% FOT charge.

Vice Versa on Withdrawal, when user submit withdrawal request, it cannot be executed since the actual collateral in vault is lesser than accounted for.

This is also affected in the StakingFaucet, where amount receives will be lesser.
## Impact
Wrong accounting causes vault to reflect incorrect amount of collateral deposited, this allow user to perform unfair leverage against the actual collateral value.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/AssetsProcess.sol#L99

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/AssetsProcess.sol#L104
```solidity
    function deposit(DepositParams calldata params) external {
        -- SNIP --
            commonData.addTradeTokenCollateral(token, params.amount);  // Wrong accounting
        -- SNIP --
            
        }
        if (accountProps.owner == address(0)) { // sets to your account if address 0, else choose another address to deposit to
            accountProps.owner = params.account;
        }
        // wrong accounting
        accountProps.addToken(token, params.amount, Account.UpdateSource.DEPOSIT); // add token amount based on token address
        -- SNIP --
```
## Tool used

Manual Review

## Recommendation
For fee-on-transfer tokens, ensure the balance before and after the transfer matches the specified amount. For rebasing tokens that decrease in value, implement a method to update cached reserves based on the current balance. This approach is complex. For rebasing tokens that increase in value, include a method to transfer the excess tokens out of the protocol, potentially directly to users.