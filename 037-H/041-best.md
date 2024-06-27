Active Punch Jellyfish

High

# Position net value is using outdated fees

## Summary
When a positions net value is calculated it factors the fees as well. However, these fees are outdated calculations. Such delay can lead to late liquidations and other unwanted occasions.
## Vulnerability Detail
This is how the cross net value is calculated:
```solidity
function _getCrossNetValue(
        PositionQueryProcess.PositionStaticsCache memory cache,
        uint256 portfolioNetValue,
        uint256 totalUsedValue,
        uint256 orderHoldUsd
    ) internal pure returns (int256) {
        return
            (portfolioNetValue + cache.totalIMUsd + orderHoldUsd).toInt256() +
            cache.totalPnl -
            totalUsedValue.toInt256() -
            cache.totalPosFee;
    }
```

As we can observe in above snippet it substracts the total fees from the net value. Let's see how this value is calculated in PositionQueryProcess.sol::getPositionFee():

```solidity
cache.fundingFeePerQty = MarketQueryProcess.getFundingFeePerQty(position.symbol, position.isLong);
        cache.unRealizedFundingFeeDelta = CalUtils.mulIntSmallRate(
            position.qty.toInt256(),
            (cache.fundingFeePerQty - position.positionFee.openFundingFeePerQty)
        );
```

```solidity
cache.cumulativeBorrowingFeePerToken = MarketQueryProcess.getCumulativeBorrowingFeePerToken(
            symbolProps.stakeToken,
            position.isLong,
            position.marginToken
        );
        cache.unRealizedBorrowingFeeDelta = CalUtils.mulSmallRate(
            CalUtils.mulRate(position.initialMargin, position.leverage - CalUtils.RATE_PRECISION),
            cache.cumulativeBorrowingFeePerToken - position.positionFee.openBorrowingFeePerToken
        );
```
 
For funding fees we get the latest perToken value from MarketQueryProcess.getFundingFeePerQty and for borrowing fees we get the latest perToken value from MarketQueryProcess.getCumulativeBorrowingFeePerToken. Which both of these values are the latest perToken values that somebody interacted with the market not the current per token values. This means that the cross net value is only up to date up to the latest interaction someone had with the market! 

**Coded PoC:**
```typescript
it("Positions can be liquidated because of outdated fee calculation", async function () {
		const usdcAm = precision.token(1_000_000, 6);
		// User has 1M USDC
		await deposit(fixture, {
			account: user0,
			token: usdc,
			amount: usdcAm,
		});
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
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
		const orderMargin = precision.token(500_000); // 500k
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

		const requestId = await marketFacet.getLastUuid(ORDER_ID_KEY);

		await orderFacet.connect(user3).executeOrder(requestId, oracleBeginning);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		let [crossNetValue, mm] = await accountFacet.getCrossMMRTapir(
			user0.address,
			oracleBeginning
		);
		console.log("Cross net value after the position", crossNetValue);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		// Accrue some fees
		await mine(1000, { interval: 150 });

		[crossNetValue, mm] = await accountFacet.getCrossMMRTapir(
			user0.address,
			oracleBeginning
		);
		console.log("Cross net value after some time passed", crossNetValue);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		// User1 creates an order, this will update the fees
		// doens't matter how big the position is
		await deposit(fixture, {
			account: user1,
			token: usdc,
			amount: usdcAm,
		});

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
				leverage: precision.rate(5),
				triggerPrice: 0,
				acceptablePrice: 0,
				executionFee: executionFee,
				placeTime: 0,
				orderMargin: precision.token(1000),
				isNativeToken: false,
			},
			{
				value: executionFee,
			}
		);

		await tx2.wait();

		const requestId2 = await marketFacet.getLastUuid(ORDER_ID_KEY);

		await orderFacet.connect(user3).executeOrder(requestId2, oracleBeginning);

		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		[crossNetValue, mm] = await accountFacet.getCrossMMRTapir(
			user0.address,
			oracleBeginning
		);
		console.log("Cross net value after user1s position", crossNetValue);
```

**Test Logs:**
Cross net value after the position 957031875000000000000000n
Cross net value after some time passed 957031875000000000000000n
Cross net value after user1s position 948117391066265975050000n

As seen when the other user interacted the protocol user0's net value dropped significantly! 
## Impact
Net cross value is used in liquidations and it's a crucial value for that. If it's delayed then the liquidations can be stale which protocol can go insolvent in extreme cases. Hence, high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/AccountProcess.sol#L20-L41

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionQueryProcess.sol#L206-L251

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MarketQueryProcess.sol#L163-L166

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MarketQueryProcess.sol#L68-L80
## Tool used

Manual Review

## Recommendation
Calculate the latest per token via these functions which will give the actual latest per token value. 
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionQueryProcess.sol#L161-L199