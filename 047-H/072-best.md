Active Punch Jellyfish

High

# LpPool's can become insolvent if shorters are in huge profits

## Summary
Pools available liquidity is crucial for a pool to be solvent in all times for traders. However, if there are too many shorters in profits, pools available liquidity will not catch that and protocol can go insolvent when they start to realize their profits.
## Vulnerability Detail
When users open LONG positions, the pool used is the LpPool of the asset. For example, if users open BTC LONG, they borrow the BTC from the LpPool, ensuring that you can't open an infinite number of LONG positions when the pool has no BTC to give. 

However, when shorting, stable tokens are used. If users short BTC and there are 10 BTC in the pool where 1 BTC is worth $25k, the total value would be $250k. Users can short BTC up to any amount as long as the UsdPool has enough stable tokens to borrow from. This means there can be more than $250k worth of profits for shorters at any time in the pool. When the pool realizes stable losses, those losses will be later converted to stable tokens and sent back to the UsdPool, meaning that the LpPool always needs to remain solvent.

```solidity
if (stableTokens.length > 0) {
    uint8 baseTokenDecimals = TokenUtils.decimals(pool.baseToken);
    int256 baseTokenPrice = OracleProcess.getIntOraclePrices(oracles, pool.baseToken, true);
    for (uint256 i; i < stableTokens.length; i++) {
        LpPool.TokenBalance storage tokenBalance = pool.stableTokenBalances[stableTokens[i]];
        if (
            tokenBalance.lossAmount > 0 &&
            tokenBalance.amount.toInt256() + tokenBalance.unsettledAmount < tokenBalance.lossAmount.toInt256()
        ) {
            int256 tokenUsd = CalUtils.tokenToUsdInt(
                tokenBalance.lossAmount.toInt256() -
                    tokenBalance.amount.toInt256() -
                    tokenBalance.unsettledAmount,
                TokenUtils.decimals(stableTokens[i]),
                OracleProcess.getIntOraclePrices(oracles, stableTokens[i], true)
            );
            int256 stableToBaseToken = CalUtils.usdToTokenInt(tokenUsd, baseTokenDecimals, baseTokenPrice);
            if (baseTokenAmount > stableToBaseToken) {
                baseTokenAmount -= stableToBaseToken;
            } else {
                baseTokenAmount = 0;
            }
        }
    }
}
```

In the above code snippet, it correctly adds the realized losses and subtracts them from the total available value. However, it doesn't consider the unrealized losses, which can be significant enough to make the pool insolvent if people keep opening positions. 

For example, say there are no stable losses and the total available liquidity is 10 BTC at the time. However, there is a short position that is in significant profit. When this profit is realized, there will be around 2 BTC worth of stable token loss. Other users open long positions, and now the pool has 1 BTC of available liquidity. However, the pool actually had 10-2=8 BTC worth of available liquidity, considering the unrealized loss. Now, the pool has 1 BTC, meaning it gave out 1 BTC knowing that it wasn't available before.
## Impact
I think this is a serious threat because it involves insolvency. Total short open interests can be capped at the market level, but this will not be enough to fix the issue because you can only cap the notional amount, not the profit and loss. Even with an open interest lower than the maximum cap, the protocol can still go insolvent if a position has a huge profit, which equates to a huge loss for the pool.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L151-L191

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/IncreasePositionProcess.sol#L34-L130
## Tool used

Manual Review

## Recommendation
The best solution would be to add unrealized losses and profits to this value to ensure 100% solvency and accuracy at all times. However, this can't be entirely precise because tracking the total short and long P&Ls isn't 100% accurate, given that the market uses the average entry price. 
I think the best approach would be to introduce a function that liquidates a short position if it is in significant profit to the extent that the pool can't afford it.