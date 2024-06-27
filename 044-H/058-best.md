Active Punch Jellyfish

High

# Closing positions does not decrease the pool's entry price, leading to misleading pool value calculations

## Summary
When positions are increased, the pool's entry price is weighted and increased accordingly. However, this adjustment does not occur when positions are decreased, leading to an invalid pool entry price and inaccurate overall pool value calculations.
## Vulnerability Detail
The pool's value is used in various places in Elfi, such as calculating the total value for ERC4626-like minting and redeeming of LP stake tokens. The pool's value is calculated as follows, referring to `LpPoolQueryProcess::getPoolIntValue`:
(pool.baseTokenBalance.amount.toInt256() + pool.baseTokenBalance.unsettledAmount + unPnl) + stableTokens

unPnl is calculated using the average entry price and open interest of the market. 

When positions are increased both open interest and entry price increases. Entry price is increased by calculating the average price. 
For example, if there is a 1000$ QTY LONG position with entry price of 1$ and 1000$ QTY LONG position with entry price of 2$ is about to be opened; then the entry price for the pool will be 1500$. 
```solidity
function _addOI(Market.MarketPosition storage position, UpdateOIParams memory params, uint256 tickSize) internal {
        if (position.openInterest == 0) {
            .
        } else {
           -> position.entryPrice = CalUtils.computeAvgEntryPrice(
                position.openInterest,
                position.entryPrice,
                params.qty,
                params.entryPrice,
                tickSize,
                params.isLong
            );
            position.openInterest += params.qty;
        }
        .
    }
```

However, when a position is decreased the entry price is not decreased as its done in increasing the position:
```solidity
function _subOI(Market.MarketPosition storage position, UpdateOIParams memory params) internal {
        if (position.openInterest <= params.qty) {
           .
        } else {
            position.openInterest -= params.qty;
        }
        .
    }
```

This will lead to incorrect pool value calculations. 

**Textual PoC:**
Assume two LONG positions with same QTY where the position1 opened when the price was 1$ and position2 opened when the price was 2$ hence, the entry price of the market is 1.5$. 

Current price is 2$. Position1 is closed with profits. However, entry price is still 1.5$. 

Current price is 1.6$. Pool is clearly in profits because the user LONG order opened in 2$ and current price is 1.6$. However, since the pools entry price is 1.5$ pool still thinks its in losses respect to entire market. 

Position2 closed when the price was 1.6$. Pool value was prior to closing position because of the entry price was lower than the current price. However, when the second position is closed the entry price is resetted to "0" and all the profits realized for the pool which lead to pools value spike up suddenly creating an unfair advantage of users who are minting/redeeming in this period. 

**Coded PoC:**
```typescript
it("Decrease position is not updating pool entry price, misleading pool value", async function () {
		const usdcAm = precision.token(500_000, 6);
		await deposit(fixture, {
			account: user0,
			token: usdc,
			amount: usdcAm,
		});
		await deposit(fixture, {
			account: user1,
			token: usdc,
			amount: usdcAm,
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
		const orderMargin = precision.token(25_000); // 1 BTC
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
				leverage: precision.rate(10),
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
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		const oracleNext = [
			{
				token: wbtcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: precision.price(50_000),
				maxPrice: precision.price(50_000),
			},
			{
				token: usdcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: precision.price(1),
				maxPrice: precision.price(1),
			},
		];

		const tx2 = await orderFacet.connect(user1).createOrderRequest(
			{
				symbol: btcUsd,
				orderSide: OrderSide.LONG,
				posSide: PositionSide.INCREASE,
				orderType: OrderType.MARKET,
				stopType: StopType.NONE,
				isCrossMargin: true,
				marginToken: wbtcAddr,
				qty: 0,
				leverage: precision.rate(10),
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
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////

		let positionInfo = await positionFacet.getSinglePosition(
			user0.address,
			btcUsd,
			wbtcAddr,
			true
		);

		// close the first position in profits!
		const tx3 = await orderFacet.connect(user0).createOrderRequest(
			{
				symbol: btcUsd,
				orderSide: OrderSide.SHORT,
				posSide: PositionSide.DECREASE,
				orderType: OrderType.MARKET,
				stopType: StopType.NONE,
				isCrossMargin: true,
				marginToken: wbtcAddr,
				qty: positionInfo.qty,
				leverage: precision.rate(10),
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

		await tx3.wait();

		const requestId3 = await marketFacet.getLastUuid(ORDER_ID_KEY);

		await orderFacet.connect(user3).executeOrder(requestId3, oracleNext);

		let poolInfo = await poolFacet.getPoolWithOracle(xBtc, oracleNext);
		console.log(
			"Pool value after the first user closes the order",
			poolInfo.poolValue
		);

		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		positionInfo = await positionFacet.getSinglePosition(
			user1.address,
			btcUsd,
			wbtcAddr,
			true
		);

		// close the second position in losses!
		const tx4 = await orderFacet.connect(user1).createOrderRequest(
			{
				symbol: btcUsd,
				orderSide: OrderSide.SHORT,
				posSide: PositionSide.DECREASE,
				orderType: OrderType.MARKET,
				stopType: StopType.NONE,
				isCrossMargin: true,
				marginToken: wbtcAddr,
				qty: positionInfo.qty,
				leverage: precision.rate(10),
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

		await tx4.wait();

		const requestId4 = await marketFacet.getLastUuid(ORDER_ID_KEY);

		await orderFacet.connect(user3).executeOrder(requestId4, oracleNext);

		poolInfo = await poolFacet.getPoolWithOracle(xBtc, oracleNext);
		console.log(
			"Pool value after the second user closes the order",
			poolInfo.poolValue
		);
```

**Test Logs:**
Pool value after the first user closes the order 4690291686366666666700000n
Pool value after the second user closes the order 4772375039400000000000000n

Right after the position closing as you see the pool value is higher than before!
## Impact
Pool value will be miscounted leading to unfair minting and redeeming of shares. Users can mint/redeem more/less shares leading to losses or unfair profits. Hence, high. 
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L110-L144

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L241-L279

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/IncreasePositionProcess.sol#L22-L130

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L60-L204

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MarketProcess.sol#L129-L206
## Tool used

Manual Review

## Recommendation
Decrease the entry price in average just like its done in increasing 