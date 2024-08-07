Active Punch Jellyfish

High

# The redeem process updates the rewards in the wrong order

## Summary
When users redeem their stakeToken's their balance decreases hence right before the redeem request the rewards should be synced. However, the code does the opposite which leads to wrong accrual of rewards.
## Vulnerability Detail
As we can see in `RedeemProcess::executeRedeemStakeToken()`, after the burning of the tokens, the rewards are updated:
```solidity
function executeRedeemStakeToken(uint256 requestId, Redeem.Request memory redeemRequest) external {
        uint256 redeemAmount;
        if (CommonData.getStakeUsdToken() == redeemRequest.stakeToken) {
            redeemAmount = _redeemStakeUsd(redeemRequest);
        } else if (CommonData.isStakeTokenSupport(redeemRequest.stakeToken)) {
            redeemAmount = _redeemStakeToken(redeemRequest);
        } else {
            revert Errors.StakeTokenInvalid(redeemRequest.stakeToken);
        }

        -> FeeRewardsProcess.updateAccountFeeRewards(redeemRequest.account, redeemRequest.stakeToken);
        .
    }
```

However, this approach is incorrect. The actual flow should be to update the account's fee rewards right before the burn to sync the account with its latest accrued rewards. Once this is done and the burn is completed, there is no need to update the account's fee rewards again since the next time the user interacts, the fee rewards will be synced, similar to the typical Masterchef contract approach.
## Impact
Unfair accrual of rewards, high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L68C5-L83C6
## Tool used

Manual Review

## Recommendation
Accrue the rewards in the beginning function 