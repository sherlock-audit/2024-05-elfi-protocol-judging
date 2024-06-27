Active Punch Jellyfish

High

# Pool value calculation skips accounting for stable token losses and short uPnL

## Summary
When isolated short positions are closed, the pool's value will not account for the loss in USDC when calculating the pool's total value.
## Vulnerability Detail
Pool's total value is crucial for Elfi as it determines the minting and burning of shares in their ERC4626-like system. The following code snippet shows how the value is calculated when there are stable token `unsettledAmount` or `amount` values:

```solidity
for (uint256 i; i < stableTokens.length; i++) {
    LpPool.TokenBalance storage tokenBalance = pool.stableTokenBalances[stableTokens[i]];
    if (tokenBalance.amount > 0 || tokenBalance.unsettledAmount > 0) {
        int256 unPnl = getMarketUnPnl(pool.symbol, oracles, false, stableTokens[i], true);
        value = value.add(
            CalUtils.tokenToUsdInt(
                (tokenBalance.amount.toInt256() +
                    tokenBalance.unsettledAmount -
                    tokenBalance.lossAmount.toInt256() +
                    unPnl),
                TokenUtils.decimals(stableTokens[i]),
                OracleProcess.getIntOraclePrices(oracles, stableTokens[i], true)
            )
        );
    }
}
```

The initial check:

```solidity
if (tokenBalance.amount > 0 || tokenBalance.unsettledAmount > 0)
```

can be false, but `tokenBalance.lossAmount.toInt256()` and `unPnl` can still be non-zero and need to be added/subtracted from the value. This issue arises when isolated positions close in profit, erasing the unsettled stable token and leaving a stable token loss for the pool. Since `tokenBalance.amount` and `tokenBalance.unsettledAmount` will be zero after closing the position, the stable token loss will not be accounted for in the pool's total value.

**Coded PoC:**
```typescript
it("Pool value skips the stable token loss and pnl", async function () {
		// dev: RUN THIS SAME TEST WITH CROSS MARGIN AND YOU WILL SEE THAT THE POOL VALUE WILL BE LESSER
		// BECAUASE IN CROSS MARGIN THERE WILL BE UNSETTLED FEE AND THE STABLE TOKENS WILL BE ADDED TO CALCULATION
		// HOWEVER IN ISOLATED THIS WILL NOT BE THE CASE AND WE WILL SKIP THE STABLE TOKEN LOSS AND PNL WHEN CALCULATING THE PV
		const wbtcAm = precision.token(10, 18);
		await deposit(fixture, {
			account: user0,
			token: wbtc,
			amount: wbtcAm,
		});
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		// Actual price is 25k
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
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		const orderMargin = precision.token(1000, 6); // 1000 USDC
		usdc.connect(user0).approve(diamondAddr, orderMargin);
		const executionFee = precision.token(2, 15);
		const tx = await orderFacet.connect(user0).createOrderRequest(
			{
				symbol: btcUsd,
				orderSide: OrderSide.SHORT,
				posSide: PositionSide.INCREASE,
				orderType: OrderType.MARKET,
				stopType: StopType.NONE,
				isCrossMargin: false,
				marginToken: usdcAddr,
				qty: 0,
				leverage: precision.rate(5),
				triggerPrice: 0,
				acceptablePrice: 0,
				executionFee: executionFee,
				placeTime: 0,
				orderMargin: orderMargin,
				isNativeToken: false,
			},
			{
				value: executionFee,
			}
		);

		await tx.wait();

		const requestId = await marketFacet.getLastUuid(ORDER_ID_KEY);

		await orderFacet.connect(user3).executeOrder(requestId, oracleBeginning);

		let accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("acc info", accountInfo);

		let poolInfo = await poolFacet.getPoolWithOracle(xBtc, oracleBeginning);
		console.log("Pools value after executing the order", poolInfo.poolValue);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		const oracleNext = [
			{
				token: wbtcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: precision.price(22_000),
				maxPrice: precision.price(22_000),
			},
			{
				token: usdcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: precision.price(1),
				maxPrice: precision.price(1),
			},
		];

		// mimick fees
		await mine(100, { interval: 15 });

		let positionInfo = await positionFacet.getSinglePosition(
			user0.address,
			btcUsd,
			usdcAddr,
			false
		);

		// close the position in losses, pool profit
		const tx2 = await orderFacet.connect(user0).createOrderRequest(
			{
				symbol: btcUsd,
				orderSide: OrderSide.LONG,
				posSide: PositionSide.DECREASE,
				orderType: OrderType.MARKET,
				stopType: StopType.NONE,
				isCrossMargin: false,
				marginToken: usdcAddr,
				qty: positionInfo.qty,
				leverage: precision.rate(5),
				triggerPrice: 0,
				acceptablePrice: 0,
				executionFee: executionFee,
				placeTime: 0,
				orderMargin: orderMargin,
				isNativeToken: false,
			},
			{
				value: executionFee,
			}
		);

		await tx2.wait();

		const requestId2 = await marketFacet.getLastUuid(ORDER_ID_KEY);

		await orderFacet.connect(user3).executeOrder(requestId2, oracleNext);

		accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("acc info", accountInfo);

		poolInfo = await poolFacet.getPoolWithOracle(xBtc, oracleNext);
		console.log("Pools value final", poolInfo.poolValue);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
	});
```
## Impact
Share minting and redeeming will be unfair. Some users can incur losses while some incur profits unusually.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L110-L144
## Tool used

Manual Review

## Recommendation
Regardless of the amount and unsettledAmount add uPnl and stable token loss