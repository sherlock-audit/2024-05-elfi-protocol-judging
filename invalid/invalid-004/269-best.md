Dapper Tiger Flamingo

Medium

# Frontrunning executeWithdraw call by keeper

## Summary

## Vulnerability Detail
`executeWithdraw` at the line https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/AccountFacet.sol#L48 can be frontrun by anyone who calls directly to `AssetsProcess.executeWithdraw(requestId, request);`

## Impact
Prevents Keeprs from calling `executeWithdraw` with a special price set to oracle.

## Code Snippet
```
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

## Tool used

Manual Review

## Recommendation
