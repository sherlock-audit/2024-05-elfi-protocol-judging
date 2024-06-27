Energetic Lemonade Woodpecker

Medium

# Attacker Checkmates Victim With Inflation Attack or DoS

## Summary
The implementation of the vault logic is creates window of oppurtunity where an attacker can exploit a victim for his `requestToken` deposited into the pool.
Attacker can either attack a victim with inflation attack or DoS. 
Attacker plots their attack by putting the right tools in place --> Victim attempts to deposit --> Attacker attempts Inflation attack on victim --> If Victim doesn't sets slippage protection (Exploit completed)--> If Victim sets slippage protection (Attacker switch to DoS to keep the attack opportunity open for another trail/victim)  --> Victim ends up not being able to execute transaction because his output will not meet his desired output --> Victim is stuck with two lose-lose choices: remove protection and get exploited, or forfeit the transaction opportunity.

## Vulnerability Detail
An attacker is able to exploit other stakers for their deposit at the point of staking. Inflation attack in a nutshell is an exploit that is a result of how division and decimal precision works in Solidity.

### Inflation Attack

**Key formulas in the contract:**
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MintProcess.sol#L253
mintStakeTokenAmount = (totalSupply * baseMintAmountInUsd) / poolValue
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L146
redeemTokenAmount= (unStakeAmount * poolValue) / totalSupply;

`poolValue` is the value of assets held in a pool at the point of staking/redeeming, which includes the  pool's baseTokenBalance.amount and baseTokenBalance.unsettledAmount. These two variables can be updated by various actions in the system, creating a loop for inflating `poolValue` without increasing totalSupply, which is a corner of this attack.

*See the details section below for some of the many ways these variable can be increased.*

>> Please read the Proof of Concept section before continuing

**Extra effort to increase attackers odds**
To increase the chances of carrying out this attack successfully, the attackers needs to have all his transactions already loaded in the in the mempool waiting to be executed at the right time. This way he isn't reliant on the keeper's order of request execution. With the current protocol design, the attacker can easily do this.

Stalling their transaction in the memPool until they are willing to strike:
- Create a request with an large `executionFee`, such that the pool will have to [reimburse](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/GasProcess.sol#L36) you with `refundFee` (which is in native token) at the end of the execution call.
- Create this transaction from an address that can't receive native token to prevent the transaction from executing successfully
- When the time is write toggle your address to be able to receive native token for the Tx to execute. This is achievable with metamorphic and upgradable contracts.


*Note that the Inflation attacker is only works when the victim doesn't set their slippage protection, thus the attacker only strikes in said condition.*

Furthermore, when slippage protection is set another attack vector opens up: DoS. 

### DoS Attack
Attacker can prevent a victim from staking, by going ahead with the inflation attacker when the victim sets a slippage, as the victims transaction will revert due to the return stakeToken amount being less than the minimum amount out set by the victim. Since the cost of the attack is very cheap especially on a layer2 network, the risk to reward for this DoS is very attractive. Although they isn't directly profitable, it is a strategy to keep the attack opportunity opened for a long time. The DoS on it own can be used to prevent others from sharing in the pools reward or search for a scenario such as this:
- Victim sets slippage protection
- Attacker prevents them from staking, in hopes of them trying again without slippage protection, or some other less diligent/technical or unknowing victim comes along with their order which has no slippage protection
- Attacker finds the prey they have been waiting for, and exploit them. 
The above scenario is very feasible considering the potential risk to reward of the attack.

## Proof of Concept
Assuming the requestToken(USDC) decimals is 6 for simplicity

Consider the following scenario:
- Attacker sees the victims stake transaction in the mempool and frontrun them with 1wei of USDC (0.000001) in the pool and receive 0.000001 stakeToken. Note that first stake is done on a 1:1 exchange rate.
> poolValue: 0.000001 USDC
> totalSupply: 0.000001

- Secondly, before the victims transaction executes, the attacker inflates poolValue by a value greater than the victims `baseMintAmountInUsd`, causing a precision loss that leads to the pool minting zero tokens for the victim, but adding their baseMintAmountInUsd to the poolValue.
- Assuming victims baseMintAmountInUsd is 100e6 (100 USD)
*Note: mintStakeTokenAmount = (totalSupply * baseMintAmountInUsd) / poolValue*
> mintStakeTokenAmount = (0.000001 * 100e6) / 101e6 == 0.0000009 == 0
> PoolValue after inflation: 101e6

- Attacker can then redeems his stakeTokens to the poolValue. see here.


<details>

**Except for the StakeFacet, here are some of the routes to update a pool's baseTokenBalance.amount without minting a stakeToken:**

StakeFacet::executeMintStakeToken() --> executeMintStakeToken() --> _mintStakeToken() --> addBaseToken() --> self.baseTokenBalance.amount += amount

PositionFacet::autoReducePositions() --> decreasePosition() --> updatePnlAndUnHoldPoolAmount() --> addBaseToken() --> self.baseTokenBalance.amount += amount

FeeFacet::distributeFeeRewards() --> distributeFeeRewards().\_updateBaseTokenRewardsToLpPool() --> addBaseToken() --> self.baseTokenBalance.amount += amount

PositionFacet::executeUpdatePositionMarginRequest() --> updatePositionMargin().\_executeAddMargin() --> updatePnlAndUnHoldPoolAmount() --> addBaseToken() --> self.baseTokenBalance.amount += amount

PositionFacet::executeUpdateLeverageRequest() --> updatePositionLeverage().\_executeAddMargin() --> updatePnlAndUnHoldPoolAmount() --> addBaseToken() --> self.baseTokenBalance.amount += amount

LiquidationFacet::liquidationAccount() --> liquidationCrossPositions() --> decreasePosition() --> updatePnlAndUnHoldPoolAmount() --> addBaseToken() --> self.baseTokenBalance.amount += amount

LiquidationFacet::liquidationPosition() --> liquidationIsolatePosition() --> decreasePosition() --> updatePnlAndUnHoldPoolAmount() --> addBaseToken() --> self.baseTokenBalance.amount += amount

OrderFacet::executeOrder() --> executeOrder() --> \_executeDecreaseOrder() --> decreasePosition() --> updatePnlAndUnHoldPoolAmount() --> addBaseToken() --> self.baseTokenBalance.amount += amount

RebalanceFacet::autoRebalance() --> autoRebalance().\_rebalanceStableTokens() --> addBaseToken() --> self.baseTokenBalance.amount += amount


**Except for the StakeFacet, here are some of the routes to update a pool's baseTokenBalance.unsettledAmount without minting a stakeToken:**

PositionFacet::executeUpdatePositionMarginRequest() --> updatePositionMargin() --> \_executeAddMargin() --> updatePnlAndUnHoldPoolAmount() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

PositionFacet::executeUpdateLeverageRequest() -->updatePositionLeverage() --> \_executeAddMargin() --> updatePnlAndUnHoldPoolAmount() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

LiquidationFacet::liquidationAccount() --> liquidationCrossPositions() --> decreasePosition() --> updatePnlAndUnHoldPoolAmount() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

LiquidationFacet::liquidationPosition() --> liquidationIsolatePosition() --> decreasePosition() --> updatePnlAndUnHoldPoolAmount() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

OrderFacet::executeOrder() --> executeOrder() --> \_executeDecreaseOrder() --> decreasePosition() --> updatePnlAndUnHoldPoolAmount() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

LiquidationFacet::liquidationAccount() --> liquidationCrossPositions() --> decreasePosition() --> updateFundingFee() --> updateMarketFundingFee() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

LiquidationFacet::liquidationPosition() --> liquidationIsolatePosition() --> decreasePosition() --> updateFundingFee() --> updateMarketFundingFee() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

OrderFacet::executeOrder() --> executeOrder() --> \_executeDecreaseOrder() --> decreasePosition() --> updateFundingFee() --> updateMarketFundingFee() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

LiquidationFacet::liquidationAccount() --> liquidationCrossPositions() --> decreasePosition() --> updateMarketFundingFee() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

LiquidationFacet::liquidationPosition() --> liquidationIsolatePosition() --> decreasePosition() --> updateMarketFundingFee() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

OrderFacet::executeOrder() --> executeOrder() --> \_executeDecreaseOrder() --> decreasePosition() --> updateMarketFundingFee() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

FeeFacet::distributeFeeRewards() --> distributeFeeRewards() --> \_updateBaseTokenRewardsToLpPool() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

OrderFacet::executeOrder --> executeOrder() --> \_executeIncreaseOrder() --> increasePosition() --> updateFundingFee() --> updateMarketFundingFee() --> addUnsettleBaseToken() --> self.baseTokenBalance.unsettledAmount += amount

</details>

## Code Snippet
**Key formulas in the contract:**
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MintProcess.sol#L253
mintStakeTokenAmount = (totalSupply * baseMintAmountInUsd) / poolValue
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L146
redeemTokenAmount= (unStakeAmount * poolValue) / totalSupply;

## Impact
Loss of funds for users from users attempting to stake.


## Tool used
Manual Review

## Recommendation
Consider reviewing Openzeppelin strategy for preventing inflation attacks on their vault system.
