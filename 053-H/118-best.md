Active Punch Jellyfish

High

# Updating leverage changes the cross net and cross available value

## Summary
When a position's leverage is updated, the cross net value changes. This can make an account's holdings appear greater than they are or, worse, cause them to decrease, leading to liquidation just by changing the leverage.
## Vulnerability Detail
Accounts' cross net value is important in determining the liquidations and overall health of the user's cross positions.

Cross net value is calculated as follows:
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

Where portfolio net value is the net amount held * discount, and the total used value is the net token used * liquidation factor. 

Let's assume Alice has the following balances:
1- USDC (discount of 99%, liquidation factor of 110%):
   - Amount: 1000
   - Used: 0
   - Liability: 0

2- tokenA (discount of 99%, liquidation factor of 110%):
   - Amount: 0
   - Used: 100
   - Liability: 0

Additionally, Alice has a LONG 10x tokenA position with an initial margin of $100 and a quantity of 10*100 = $1000, with tokenA initially trading at $1.

Alice's total net value would be:
((1000 * 99/100) + 100) + 0 - (100 * 110/100) - 0 =
= 980$ 

Now, Alice thinks her position is too highly leveraged and decides to reduce her leverage to save herself from potential downside risks, bringing it down to 2x. This will increase her initial margin and net token used value. The increased margin will be $400, since the quantity has to remain the same.

Alice now has a $500 margin and 500 used tokens. Let's re-calculate Alice's cross value:
((1000 * 99/100) + 500) + 0 - (500 * 110/100) - 0 =
= 940$ 

As we can observe, Alice's cross net value dropped significantly. This could lead Alice to be liquidated if there were actual losses from the position!

This happens because when a position is de-leveraged, both the margin and the used amount increase. However, since the used amount is always multiplied by the liquidation factor, it grows more rapidly than the initial margin, which is a fixed amount.

**Coded PoC:**
```typescript
it("Delever decreases the cross net value", async function () {
		const usdcAm = precision.token(200_000, 6);
		await deposit(fixture, {
			account: user0,
			token: usdc,
			amount: usdcAm,
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

		const orderMargin = precision.token(50_000); // 100$
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

		let crossNetValue = await accountFacet.getCrossMMRTapir(
			user0.address,
			oracleBeginning
		);
		console.log("CNV first", crossNetValue.crossValue);

		let accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("Account info before", accountInfo.tokenBalances);

		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
		let ptx = await positionFacet.connect(user0).createUpdateLeverageRequest(
			{
				symbol: btcUsd,
				isLong: true,
				isNativeToken: false,
				isCrossMargin: true,
				leverage: precision.rate(2),
				marginToken: wbtcAddr,
				addMarginAmount: 0,
				executionFee: executionFee,
			},
			{
				value: executionFee,
			}
		);

		await ptx.wait();

		await positionFacet
			.connect(user3)
			.executeUpdateLeverageRequest(BigInt(1112), oracleBeginning);

		crossNetValue = await accountFacet.getCrossMMRTapir(
			user0.address,
			oracleBeginning
		);
		console.log("CNV finale", crossNetValue.crossValue);

		accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("Account info final", accountInfo.tokenBalances);
});
```

**Test Logs:**
CNV first 194703187500000000000000n
Account info before Result(2) [
  Result(4) [ 200000000000n, 0n, 0n, 0n ],
  Result(4) [ 0n, 2000000000000000000n, 0n, 15000000000000000n ]
]
CNV finale 190981312500000000000000n
Account info final Result(2) [
  Result(4) [ 200000000000n, 0n, 0n, 0n ],
  Result(4) [ 0n, 4977500000000000000n, 0n, 15000000000000000n ]
]
## Impact
Accounts can have more or less cross net value after updating leverage. Updating the leverage does not change the total quantity and it should not be changing the cross net value of users. Users have more cross net value by increasing their leverage and have lesser cross net value by decreasing their leverage which is conflicting and can lead to over borrowed positions or unfair liquidations. 
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/AccountProcess.sol#L149-L160

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/AccountProcess.sol#L162-L198

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L134-L229

https://github.com/sherlock-audit/2024-05-elfi-protocol/blame/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L340-L368
## Tool used

Manual Review

## Recommendation
