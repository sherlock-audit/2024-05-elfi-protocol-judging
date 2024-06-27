Active Punch Jellyfish

High

# Keepers can open positions that are already liquidatable

## Summary
Users positions can be opened already liquidated because there are no checks when the position is opened whether the position is liquidatable or not. 
## Vulnerability Detail
When users submit their order requests, the requests are never validated to determine if they will be liquidatable immediately. That being said, in very volatile markets or with users' greedy positions, positions(isolated) or accounts(cross) can be liquidated immediately upon opening. 

**Coded PoC:**
```typescript
it("Keeper opens a position that is liqable already", async function () {
		const usdcAmount = precision.token(1000, 6);
		await deposit(fixture, {
			account: user0,
			token: usdc,
			amount: usdcAmount,
		});

		const orderMargin = precision.token(5000); // 5000$
		const executionFee = precision.token(2, 15);
		const tx = await orderFacet.connect(user0).createOrderRequest(
			{
				symbol: btcUsd,
				orderSide: OrderSide.LONG,
				posSide: PositionSide.INCREASE,
				orderType: OrderType.MARKET,
				stopType: StopType.NONE,
				isCrossMargin: true,
				marginToken: wbtcAddr,
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

		const tokenPrice = precision.price(25000);
		const usdcPrice = precision.price(100, 6); // 0.99$
		const oracle = [
			{
				token: wbtcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: tokenPrice,
				maxPrice: tokenPrice,
			},
			{
				token: usdcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: usdcPrice,
				maxPrice: usdcPrice,
			},
		];

		const requestId = await marketFacet.getLastUuid(ORDER_ID_KEY);

		await orderFacet.connect(user3).executeOrder(requestId, oracle);

		const nextWbtcPrice = precision.price(18000);
		const nextOracle = [
			{
				token: wbtcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: nextWbtcPrice,
				maxPrice: nextWbtcPrice,
			},
			{
				token: usdcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: usdcPrice,
				maxPrice: usdcPrice,
			},
		];

		// @dev: added this function to facet to see if the account is liquidatable in this test
		const isLiqable = await accountFacet.getIsLiqableTapir(
			user0.address,
			nextOracle
		);

		console.log("is liqable", isLiqable);

		expect(isLiqable).to.be.equal(true);

		const positionInfo = await positionFacet.getSinglePosition(
			user0.address,
			btcUsd,
			wbtcAddr,
			true
		);

		const tx2 = await orderFacet.connect(user0).createOrderRequest(
			{
				symbol: btcUsd,
				orderSide: OrderSide.SHORT,
				posSide: PositionSide.DECREASE,
				orderType: OrderType.MARKET,
				stopType: StopType.NONE,
				isCrossMargin: true,
				marginToken: wbtcAddr,
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
		await orderFacet.connect(user3).executeOrder(requestId2, nextOracle);
	});
```
## Impact
Protocol can be insolvent if the liquidation is too deep such that the accounts collateral is not enough to cover the potential losses. 
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/OrderFacet.sol#L66-L87

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L102-L234
## Tool used

Manual Review

## Recommendation
check if the opened position will make the account liquidatable at the end of the execute order. 