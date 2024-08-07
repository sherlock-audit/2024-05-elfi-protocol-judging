Active Punch Jellyfish

High

# Closing partial positions miscounts the settled fees

## Summary
When positions are partially closed calculating settleMargin recordPnlToken will be wrong because of the additional division.
## Vulnerability Detail
Bob opens a BTC LONG 5x 1000$ margin position where initially 1 BTC is 25k$. 

When the price hits 27.5k$ Bob decides to close the half of his position.

Since Bob is closing half of the position the following else statement will be executed in the DecreasePositionProcess::_updateDecreasePosition internal function:
```solidity
else {
            cache.decreaseMargin = cache.position.initialMargin.mul(decreaseQty).div(cache.position.qty);
            cache.unHoldPoolAmount = cache.position.holdPoolAmount.mul(decreaseQty).div(cache.position.qty);
            cache.closeFeeInUsd = CalUtils.mulRate(decreaseQty, closeFeeRate);
            (cache.settledBorrowingFee, cache.settledBorrowingFeeInUsd) = FeeQueryProcess.calcBorrowingFee(
                decreaseQty,
                position
            );
            cache.decreaseIntQty = decreaseQty.toInt256();
            cache.positionIntQty = cache.position.qty.toInt256();
            cache.settledFundingFee = cache.position.positionFee.realizedFundingFee.mul(cache.decreaseIntQty).div(
                cache.positionIntQty
            );
            cache.settledFundingFeeInUsd = cache
                .position
                .positionFee
                .realizedFundingFeeInUsd
                .mul(cache.decreaseIntQty)
                .div(cache.positionIntQty);

            if (cache.closeFeeInUsd > cache.position.positionFee.closeFeeInUsd) {
                cache.closeFeeInUsd = cache.position.positionFee.closeFeeInUsd;
            }
            cache.closeFee = FeeQueryProcess.calcCloseFee(tokenDecimals, cache.closeFeeInUsd, tokenPrice.toUint256());
            cache.settledFee =
                cache.settledBorrowingFee.toInt256() +
                cache.settledFundingFee +
                cache.closeFee.toInt256();
            cache.settledMargin = CalUtils.usdToTokenInt(
                (cache.position.initialMarginInUsd.toInt256() - _getPosFee(cache) + pnlInUsd)
                    .mul(cache.decreaseIntQty)
                    .div(cache.positionIntQty),
                TokenUtils.decimals(cache.position.marginToken),
                tokenPrice
            );
            cache.recordPnlToken = cache.settledMargin - cache.decreaseMargin.toInt256();
            cache.poolPnlToken =
                cache.decreaseMargin.toInt256() -
                CalUtils.usdToTokenInt(
                    (cache.position.initialMarginInUsd.toInt256() + pnlInUsd).mul(cache.decreaseIntQty).div(
                        cache.positionIntQty
                    ),
                    TokenUtils.decimals(cache.position.marginToken),
                    tokenPrice
                );
            cache.decreaseMarginInUsd = cache.position.initialMarginInUsd.mul(decreaseQty).div(position.qty);
            cache.realizedPnl = CalUtils.tokenToUsdInt(
                cache.recordPnlToken,
                TokenUtils.decimals(cache.position.marginToken),
                tokenPrice
            );
        }
```

As we can observe in above code snippet, the settled fees are converted to usd individuals and assigned to variable such as `cache.settledFundingFeeInUsd`, `cache.settledBorrowingFeeInUsd`. These values are already divided by "2" since we are decreasing the half of the position. 

When we calculate the `cache.settledMargin` we will do the following math operation:
```solidity
(cache.position.initialMarginInUsd.toInt256() - _getPosFee(cache) + pnlInUsd)
                    .mul(cache.decreaseIntQty)
                    .div(cache.positionIntQty)
```
cache.position.initialMarginInUsd.toInt256() -> is not divided by the decrease amount yet
**_getPosFee(cache) -> is the sum of all settled fees in usd which all divided by the decrease amount already!** 
pnlInUsd -> is the total pnl of the total position not divided by the decrease amount yet

as we can see `_getPosFee` already divided by the decrease amount and when we calculate the `settledMargin` we do divide it one more time to decrease amount which lowers down the fee amounts and give us a wrong value `settleMargin`, `poolPnlToken` and `recordPnlToken` values which are crucial for the system. 

In the end, if the total settled fees are "positive" then closing partial positions will be always more profitable for the user. If the total settled fees are "negative" then closing full positions will be always more profitable for the user. 

**Coded PoC:**
```typescript
it("Close in two parts", async function () {
		const usdcAmount = precision.token(2000, 6);
		await deposit(fixture, {
			account: user0,
			token: usdc,
			amount: usdcAmount,
		});
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////

		const orderMargin = precision.token(1000); // 1000$
		usdc.connect(user0).approve(diamondAddr, orderMargin);
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

		const tokenPrice = precision.price(25000);
		const usdcPrice = precision.price(99, 6); // 0.99$
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

		await orderFacet.connect(user3).executeOrder(requestId, oracle);

		// mimick fees
		await mine(1000, { interval: 30 });
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////

		let positionInfo = await positionFacet.getSinglePosition(
			user0.address,
			btcUsd,
			wbtcAddr,
			true
		);

		// close only half of the position in profits
		const tx2 = await orderFacet.connect(user0).createOrderRequest(
			{
				symbol: btcUsd,
				orderSide: OrderSide.SHORT,
				posSide: PositionSide.DECREASE,
				orderType: OrderType.MARKET,
				stopType: StopType.NONE,
				isCrossMargin: true,
				marginToken: wbtcAddr,
				qty: BigInt(positionInfo.qty) / BigInt(2),
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

		const oracle2 = [
			{
				token: wbtcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: precision.price(27500),
				maxPrice: precision.price(27500),
			},
			{
				token: usdcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: usdcPrice,
				maxPrice: usdcPrice,
			},
		];

		const requestId2 = await marketFacet.getLastUuid(ORDER_ID_KEY);
		await orderFacet.connect(user3).executeOrder(requestId2, oracle2);

		let accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("Account after closing half of the position", accountInfo);

		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		positionInfo = await positionFacet.getSinglePosition(
			user0.address,
			btcUsd,
			wbtcAddr,
			true
		);

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

		await tx3.wait();

		const requestId3 = await marketFacet.getLastUuid(ORDER_ID_KEY);

		await orderFacet.connect(user3).executeOrder(requestId3, oracle2);

		accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("Account info final two parts", accountInfo);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
	});

	it("Close in one go", async function () {
		const usdcAmount = precision.token(2000, 6);
		await deposit(fixture, {
			account: user0,
			token: usdc,
			amount: usdcAmount,
		});
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////

		const orderMargin = precision.token(1000); // 1000$
		usdc.connect(user0).approve(diamondAddr, orderMargin);
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

		const tokenPrice = precision.price(25000);
		const usdcPrice = precision.price(99, 6); // 0.99$
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

		await orderFacet.connect(user3).executeOrder(requestId, oracle);

		// mimick fees
		await mine(1000, { interval: 30 });
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////

		let positionInfo = await positionFacet.getSinglePosition(
			user0.address,
			btcUsd,
			wbtcAddr,
			true
		);

		const oracle2 = [
			{
				token: wbtcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: precision.price(27500),
				maxPrice: precision.price(27500),
			},
			{
				token: usdcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: usdcPrice,
				maxPrice: usdcPrice,
			},
		];

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

		await tx3.wait();

		const requestId3 = await marketFacet.getLastUuid(ORDER_ID_KEY);

		await orderFacet.connect(user3).executeOrder(requestId3, oracle2);

		let accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("Account info final one go", accountInfo);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
	});
```

**Test Logs:**
Account info final two parts Result(11) [
  '0x70997970C51812dc3A010C7d01b50e0d17dc79C8',
  Result(2) [
    Result(4) [ 2000000000n, 0n, 0n, 0n ],
    Result(4) [ 13825113546993371n, 0n, 0n, 0n ]
  ],
Account info final one go Result(11) [
  '0x70997970C51812dc3A010C7d01b50e0d17dc79C8',
  Result(2) [
    Result(4) [ 2000000000n, 0n, 0n, 0n ],
    Result(4) [ 13721368330375054n, 0n, 0n, 0n ]
  ],
  
13825113546993371n > 13721368330375054n !
## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L206-L336
## Tool used

Manual Review

## Recommendation
Don't divide the `_getPosFees()` since its already divided by the decreased amount