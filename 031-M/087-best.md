Active Punch Jellyfish

Medium

# Users can't withdraw their full stake token amount if they are the last withdrawer

## Summary
When users deposit to stake token liquidity, they receive shares. However, redeeming these shares for the underlying tokens will not be fully possible, and the user will always lose money if they are the last to withdraw from the system.
## Vulnerability Detail
When the pool calculates if a user's shares are redeemable relative to the pool's current liquidity, it calls the `pool.getPoolAvailableLiquidity()` function in the `RedeemProcess::_executeRedeemStakeToken` internal function.

The pool's available liquidity is the sum of all tokens and stable tokens it has, multiplied by a discount factor. This is calculated in the `LpPoolQueryProcess::getAvailableLiquidity()` internal function as follows:
```solidity
CalUtils.mulRate(baseTokenAmount, pool.getPoolLiquidityLimit().toInt256());
```

This ensures that the available liquidity will always be lower than the actual amount. For example, if there are 100 tokens, the total available liquidity will be 90 tokens. This means that even if there are no positions open, users can't fully withdraw their tokens without incurring a loss.

**Coded PoC:**
```typescript
it("Can't redeem the full amount without reverts due to pool liq limit", async function () {
		const wbtcAm = precision.token(10, 18);
		await deposit(fixture, {
			account: user0,
			token: wbtc,
			amount: wbtcAm,
		});

		const oracleBeginning = [
			{
				token: wbtcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: precision.price(25_000),
				maxPrice: precision.price(25_000),
			},
			{
				token: usdcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: precision.price(1),
				maxPrice: precision.price(1),
			},
		];

		let pi = await poolFacet.getPoolWithOracle(xBtc, oracleBeginning);
		console.log("pi", pi.availableLiquidity);

		// // user0 has exactly 100 shares so he should be able to claim it back
		// // but because of the pool available liquidity all he can redeem back is
		// // 79.94 shares so the rest is lost for the user.

		// reverts with "RedeemWithAmountNotEnough"
		await handleRedeem(fixture, {
			stakeToken: xBtc,
			redeemToken: wbtc,
			unStakeAmount: precision.token(100),
			oracle: oracleBeginning,
		});
```
## Impact
Admin can set the discount to "0" to fix this. However, setting it to "0" might cause more problems. Hence, labelling as medium.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/RedeemProcess.sol#L133-L226

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L151-L191
## Tool used

Manual Review

## Recommendation
