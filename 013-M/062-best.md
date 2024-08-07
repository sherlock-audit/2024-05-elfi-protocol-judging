Overt Fiery Starfish

Medium

# Lack of execution fee mechanism in AccountFacet

## Summary
When the keeper execute `executeWithdraw` or `cancelWithdraw`, no execution fee is payed for the keeper.

## Vulnerability Detail
In OrderFacet and StakeFaucet, when the keepers execute increase order/ stake tokens, there will be some execution fee for the keepers. However, in AccountFacet, we lack of execution fee mechanism. Considering if gas price increases or there is not enough motivation to trigger `executeWithdraw` or `cancelWithdraw`.
This will cause traders' redeem may be blocked.

```javascript
    function executeWithdraw(uint256 requestId, OracleProcess.OracleParam[] calldata oracles) external override {
        RoleAccessControl.checkRole(RoleAccessControl.ROLE_KEEPER);
        Withdraw.Request memory request = Withdraw.get(requestId);
        if (request.account == address(0)) {
            revert Errors.WithdrawRequestNotExists();
        }
        OracleProcess.setOraclePrice(oracles);
        AssetsProcess.executeWithdraw(requestId, request);
        OracleProcess.clearOraclePrice();
    }

```

## Impact
The keepers has less motivation to trigger `executeWithdraw` or `cancelWithdraw` compared with other operations. This will block the traders' collateral withdraw.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/AccountFacet.sol#L48-L57

## Tool used

Manual Review

## Recommendation
Add execution fee mechanism for AccountFacet.