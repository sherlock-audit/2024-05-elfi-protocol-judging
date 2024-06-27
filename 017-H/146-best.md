Proper Tartan Llama

High

# Attacker can inflate stake rewards as he wants.

## Summary
`FeeRewardsProcess.sol#updateAccountFeeRewards` function uses balance of account as amount of stake tokens.
Since it is possible to transfer stake tokens to any accounts, attacker can flash loan other's stake tokens to inflate stake rewards.

## Vulnerability Detail
`FeeRewardsProcess.sol#updateAccountFeeRewards` function is the following.
```solidity
    function updateAccountFeeRewards(address account, address stakeToken) public {
        StakingAccount.Props storage stakingAccount = StakingAccount.load(account);
        StakingAccount.FeeRewards storage accountFeeRewards = stakingAccount.getFeeRewards(stakeToken);
        FeeRewards.MarketRewards storage feeProps = FeeRewards.loadPoolRewards(stakeToken);
        if (accountFeeRewards.openRewardsPerStakeToken == feeProps.getCumulativeRewardsPerStakeToken()) {
            return;
        }
63:     uint256 stakeTokens = IERC20(stakeToken).balanceOf(account);
        if (
            stakeTokens > 0 &&
            feeProps.getCumulativeRewardsPerStakeToken() - accountFeeRewards.openRewardsPerStakeToken >
            feeProps.getPoolRewardsPerStakeTokenDeltaLimit()
        ) {
            accountFeeRewards.realisedRewardsTokenAmount += (
                stakeToken == CommonData.getStakeUsdToken()
                    ? CalUtils.mul(
                        feeProps.getCumulativeRewardsPerStakeToken() - accountFeeRewards.openRewardsPerStakeToken,
                        stakeTokens
                    )
                    : CalUtils.mulSmallRate(
                        feeProps.getCumulativeRewardsPerStakeToken() - accountFeeRewards.openRewardsPerStakeToken,
                        stakeTokens
                    )
            );
        }
        accountFeeRewards.openRewardsPerStakeToken = feeProps.getCumulativeRewardsPerStakeToken();
        stakingAccount.emitFeeRewardsUpdateEvent(stakeToken);
    }
```
Balance of account is used as amount of stake tokens in `L63`.
But since the stake tokens can be transferred to any other account, attacker can inflate stake token rewards by flash loan.

Example:
1. User has two account: `account1`, `account2`.
2. User has staked 1000 ETH in `account1` and 1000 ETH in `account2`.
3. After a period of time, user transfer 1000 xETH from `account2` to `account1` and claim rewards for `account1`.
4. Now, attacker can claim rewards twice for `account1`.
5. In the same way, attacker can claim rewards twice for `account2` too.

## Impact
Attacker can inflate stake rewards as he wants using this vulnerability.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/FeeRewardsProcess.sol#L63

## Tool used

Manual Review

## Recommendation
Use `stakingAccount.stakeTokenBalances[stakeToken].stakeAmount` instead of stake token balance as follows.
```solidity
    function updateAccountFeeRewards(address account, address stakeToken) public {
        StakingAccount.Props storage stakingAccount = StakingAccount.load(account);
        StakingAccount.FeeRewards storage accountFeeRewards = stakingAccount.getFeeRewards(stakeToken);
        FeeRewards.MarketRewards storage feeProps = FeeRewards.loadPoolRewards(stakeToken);
        if (accountFeeRewards.openRewardsPerStakeToken == feeProps.getCumulativeRewardsPerStakeToken()) {
            return;
        }
--      uint256 stakeTokens = IERC20(stakeToken).balanceOf(account);
++      uint256 stakeTokens = stakingAccount.stakeTokenBalances[stakeToken].stakeAmount;
        if (
            stakeTokens > 0 &&
            feeProps.getCumulativeRewardsPerStakeToken() - accountFeeRewards.openRewardsPerStakeToken >
            feeProps.getPoolRewardsPerStakeTokenDeltaLimit()
        ) {
            accountFeeRewards.realisedRewardsTokenAmount += (
                stakeToken == CommonData.getStakeUsdToken()
                    ? CalUtils.mul(
                        feeProps.getCumulativeRewardsPerStakeToken() - accountFeeRewards.openRewardsPerStakeToken,
                        stakeTokens
                    )
                    : CalUtils.mulSmallRate(
                        feeProps.getCumulativeRewardsPerStakeToken() - accountFeeRewards.openRewardsPerStakeToken,
                        stakeTokens
                    )
            );
        }
        accountFeeRewards.openRewardsPerStakeToken = feeProps.getCumulativeRewardsPerStakeToken();
        stakingAccount.emitFeeRewardsUpdateEvent(stakeToken);
    }
```