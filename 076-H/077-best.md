Glorious Onyx Condor

High

# Submitting mint request using user's trading balance and cancelling it will not refund tokens back to trading account

## Summary
The Elfi protocol provides a feature that allows users to transfer a portion of their tokens after depositing them into a trading account, helping them manage their finances more effectively. 
According to the dev team:
> When the keeper fails to execute the execute method, it will call the corresponding cancel method to revoke this request

 The problem arises if the mint request uses the trading balance to get elfi tokens.

## Vulnerability Detail
This is the logic where users can use trading account balances to obtain elfi tokens.
```solidity
    function _mintStakeToken(Mint.Request memory mintRequest) internal returns (uint256 stakeAmount) {
    -- SNIP --
        if (mintRequest.requestTokenAmount > mintRequest.walletRequestTokenAmount) {
            _transferFromAccount( 
                mintRequest.account,
                mintRequest.requestToken,
                mintRequest.requestTokenAmount - mintRequest.walletRequestTokenAmount
            );
        }
    -- SNIP --
```
The `MintProcess::cancelMintStakeToken` refunds the tokens sent from user EOA to the vault. However, the trading balance account isn't refunded if mint requests are executed and funded by the trading account.

```solidity
    function cancelMintStakeToken(uint256 requestId, Mint.Request memory mintRequest, bytes32 reasonCode) external {
        if (mintRequest.walletRequestTokenAmount > 0) { //@audit missing implementation for trading account
            VaultProcess.transferOut(
                mintRequest.isCollateral
                    ? IVault(address(this)).getPortfolioVaultAddress() // ok
                    : IVault(address(this)).getLpVaultAddress(),// ok
                mintRequest.requestToken,
                mintRequest.account, //transfer to user
                mintRequest.walletRequestTokenAmount
            );
        }
        Mint.remove(requestId);
        emit CancelMintEvent(requestId, mintRequest, reasonCode);
    }
``` 


## Impact
Users who staked with trading balance will lose tokens if mint requests are cancelled.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/MintProcess.sol#L94
## Tool used

Manual Review

## Recommendation
The development team can choose to implement either refund to the EOA (Externally Owned Account) or to the trading account.

The following solution, updates the trading balance account:
```diff
    function cancelMintStakeToken(uint256 requestId, Mint.Request memory mintRequest, bytes32 reasonCode) external {
+       if (mintRequest.requestTokenAmount > mintRequest.walletRequestTokenAmount) {
+       Account.Props storage accountProps = Account.load(mintRequest.account);
+       accountProps.checkExists();
+       accountProps.addToken(mintRequest.requstToken, mintrequest.requestTokenAmount);
+       }
+       else (mintRequest.walletRequestTokenAmount > 0) { //@audit missing implementation for trading account
            VaultProcess.transferOut(
                mintRequest.isCollateral
                    ? IVault(address(this)).getPortfolioVaultAddress() // ok
                    : IVault(address(this)).getLpVaultAddress(),// ok
                mintRequest.requestToken,
                mintRequest.account, //transfer to user
                mintRequest.walletRequestTokenAmount
            );
        }
        Mint.remove(requestId);
        emit CancelMintEvent(requestId, mintRequest, reasonCode);
    }

``` 