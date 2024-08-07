Glorious Onyx Condor

High

# Keeper cannot call execute function due to invalid access control implementation

## Summary
As per README:
>Admin Role (TRUSTED): Used for managing other Roles
Keeper Role (RESTRICTED): Only used for executing two-phase commits and scheduled tasks.

> For permissioned functions, please list all checks and requirements that will be made before calling the function.
All two-phase commit functions (functions starting with 'execute' in Facet) can only be called by the Keeper Role, such as executeOrder().
All first-phase cancellation functions can only be called by the Keeper Role (except for cancelOrder, which users can also use to cancel orders).

We know that Keeper Role != Admin Role. Problem arise when keeper tries to execute request.
## Vulnerability Detail
In `StakeFacet::executeMintStakeToken` which calls `MintProcess::executeMintStakeToken`. Within that function, it will transfer tokens from the LP vault to a token pool which is only callable by an account with the `ADMIN_ROLE`. Whenever keeper role call the two step execution function it reverts.
```solidity
    function executeMintStakeToken( 
        uint256 requestId,
        Mint.Request memory mintRequest
    ) external returns (uint256 stakeAmount) {
    -- SNIP --
        if (!mintRequest.isCollateral && mintRequest.walletRequestTokenAmount > 0) {
            IVault(address(this)).getLpVault().transferOut( //@audit-issue Require Admin Role
                mintRequest.requestToken,
                mintRequest.stakeToken,
                mintRequest.walletRequestTokenAmount
            );
        }
```
```solidity
    function transferOut(address token, address receiver, uint256 amount) external onlyRole(ADMIN_ROLE) {
        if (receiver == address(this)) {
            revert AddressSelfNotSupported(receiver);
        }
        TransferUtils.transfer(token, receiver, amount);
    }
```
## Impact
Denial of service when keeper bot call execution function. All requests will not be executed.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/vault/Vault.sol#L16
## Tool used

Manual Review

## Recommendation
Create another access control modifier that allows keeper bot to execute the function.