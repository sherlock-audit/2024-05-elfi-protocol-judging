Active Punch Jellyfish

High

# If cross positions use the same margin token as collateral and close without liability, then fee accounting will be completely wrong

## Summary
Cross positions can use different assets or the same asset as the position's margin asset. If the assets are the same, the entire loss, including the fees, will be sent from the portfolio vault to the stakeToken. However, for cross margin trades, the fees are always recorded as unsettled, because in the future they will be taken from the portfolio vault and settled.
## Vulnerability Detail
Starting from the `DecreasePositionProcess::decreasePosition()` function, the first thing will be to update the fees and unsettled fee/base token amounts:
```solidity
function decreasePosition(Position.Props storage position, DecreasePositionParams calldata params) external {
        int256 totalPnlInUsd = PositionQueryProcess.getPositionUnPnl(position, params.executePrice.toInt256(), false);
        Symbol.Props memory symbolProps = Symbol.load(params.symbol);
        AppConfig.SymbolConfig memory symbolConfig = AppConfig.getSymbolConfig(params.symbol);
        -> FeeProcess.updateBorrowingFee(position, symbolProps.stakeToken);
        -> FeeProcess.updateFundingFee(position);
}
```

```solidity
function updateFundingFee(Position.Props storage position) public {
        .
        .
        MarketProcess.updateMarketFundingFee(
            position.symbol,
            realizedFundingFee,
            position.isLong,
            -> true, // increase the unsettled amount!
            position.marginToken
        );
```

```solidity
function updateMarketFundingFee(
        bytes32 symbol,
        int256 realizedFundingFeeDelta,
        bool isLong,
        -> bool needUpdateUnsettle,
        address marginToken
    ) external {
        .
        -> if (needUpdateUnsettle) {
            Symbol.Props storage symbolProps = Symbol.load(symbol);
            LpPool.Props storage pool = LpPool.load(symbolProps.stakeToken);
            -> if (isLong) {
                pool.addUnsettleBaseToken(realizedFundingFeeDelta);
            } else {
                pool.addUnsettleStableToken(marginToken, realizedFundingFeeDelta);
            }
        }
    }
```

Then, If the position is cross, the fees are increasing the "unsettled" amounts as follows in the decrease position flow:
```solidity
FeeProcess.chargeTradingFee(
            cache.closeFee,
            symbolProps.code,
            cache.isLiquidation ? FeeProcess.FEE_LIQUIDATION : FeeProcess.FEE_CLOSE_POSITION,
            cache.position.marginToken,
            cache.position
        );

        FeeProcess.chargeBorrowingFee(
            position.isCrossMargin,
            cache.settledBorrowingFee,
            symbolProps.stakeToken,
            cache.position.marginToken,
            position.account,
            cache.isLiquidation ? FeeProcess.FEE_LIQUIDATION : FeeProcess.FEE_BORROWING
        );
```
```solidity
function chargeTradingFee(
        uint256 fee,
        bytes32 symbol,
        bytes32 feeType,
        address feeToken,
        Position.Props memory position
    ) internal {
        .
        .
        .
        .
        -> if (position.isCrossMargin) {
            marketTradingRewardsProps.addUnsettleFeeAmount(feeToken, cache.feeToMarketRewards);
            stakingRewardsProps.addUnsettleFeeAmount(cache.stakeToken, feeToken, cache.feeToStakingRewards);
            daoRewardsProps.addUnsettleFeeAmount(cache.stakeToken, feeToken, cache.feeToDaoRewards);
        }
        emit ChargeTradingFeeEvent(symbol, position.account, position.key, feeType, feeToken, fee);
    }
```

Then, when a cross account is closed with losses the following lines will be executed in decrease position flow:
```solidity
else if (cache.recordPnlToken < 0) {
            addLiability = accountProps.subTokenWithLiability(
                cache.position.marginToken,
                (-cache.recordPnlToken).toUint256()
            );
            VaultProcess.transferOut(
                portfolioVault,
                cache.position.marginToken,
                cache.stakeToken,
                (-cache.recordPnlToken).toUint256() - addLiability,
                true
            );
        }
```

First, the token will be subtracted from the cross balance. Since the account has a greater same token balance in his cross account this will create no liabilities, hence, the `addLiability` will be "0". Then, the entire `recordPnlToken` will be sent from portfolio vault to stakeToken. Note that the `recordPnlToken` is the sum of "fees + pnl".

Moving on with the execution flow the following call will be made to update the funding fee:
```solidity
MarketProcess.updateMarketFundingFee(
            symbolProps.code,
            -cache.settledFundingFee,
            cache.position.isLong,
            -> !position.isCrossMargin, // if cross this will be FALSE!
            cache.position.marginToken
        );
```

As stated in the above code comment the 4th argument will be `false` which will not decrease the funding fees that are previously added when the position was in `DecreasePosition::decreasePosition()` level (check the very first above code snippets, 4th argument was "true" there).

This will create discrepancy in the fees for cross accounts. Cross positions fees should be taken from portfolio vault however, in this case, all the funds are already sent to stakeToken. When fees are settled they will be again taken from portfolio vault and there will be double counting.

**Textual PoC:**
Following the increasePosition and decreasePosition flows here is the scenario:

LONG position 100$ margin 5x lev on token TAPIR which the price is 1$:

orderMargin = 100 TAPIR
orderMarginFromBalance = 100 TAPIR

FOR TAPIR:
balance.amount = 0
balance.usedAmount = 100

fee = 2 TAPIR
balance.usedAmount = 98
balance.amount = 98

increaseMargin = 98 TAPIR
increaseMarginFromBalance = 98 TAPIR
increaseQty = 490$

initialMargin = 98 TAPIR
initialMarginInUsd = 98$
initialMarginInUsdFromBalance = 98$
closeFeeInUsd = 2$
realizedPnl = -2$
holdPoolAmount = 392 TAPIR
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
////////////////////////////////////////////////////////////////
Price is 0.9$ close entire pos

totalPnlInUsd = -49$

settledBorrowingFee = 4 tokens
settledFundingFee = 4 tokens
closeFee = 2 tokens

settledFee = 10 tokens
settledMargin = toToken(98 - (10*0.9) - 49) =
= 44.44 tokens
recordPnlToken = 44.44 - 98 =
= -53.56 tokens
poolPnlToken = 98 - toToken(98 - 49) =
= 43.56 tokens

FOR TAPIR:
balance.amount -= 10 = 88
balance.usedAmount -= 98 = 0
balance.amount -= 53.56 = 34.44

**From portfolio vault to stakeToken 53.56 tokens sent**

pool.baseAmount += 43.56 = 1043.56

**So basically fee is in the stakeToken but not added to baseAmount**

**Coded PoC:**
```typescript
it("Use different token as collateral", async function () {
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

		let poolInfo = await poolFacet.getPoolWithOracle(xBtc, oracleBeginning);
		console.log("Pools balances base amount", poolInfo.baseTokenBalance.amount);
		console.log(
			"Pools balances unsettled base amount",
			poolInfo.baseTokenBalance.unsettledAmount
		);

		let balanceBeforeClosePos = await wbtc.balanceOf(xBtc);
		console.log(
			"Balance after closing the position in loss",
			balanceBeforeClosePos
		);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
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

		let positionInfo = await positionFacet.getSinglePosition(
			user0.address,
			btcUsd,
			wbtcAddr,
			true
		);

		// mimick fees
		await mine(1000, { interval: 300 });

		// close the position in losses, pool profit
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

		poolInfo = await poolFacet.getPoolWithOracle(xBtc, oracleBeginning);
		console.log(
			"Pools balances base amount after",
			poolInfo.baseTokenBalance.amount
		);
		console.log(
			"Pools balances unsettled base amount after",
			poolInfo.baseTokenBalance.unsettledAmount
		);

		let accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("acc info", accountInfo);

		let balanceAfterClosePos = await wbtc.balanceOf(xBtc);
		console.log(
			"Balance after closing the position in loss",
			balanceAfterClosePos
		);

		// no token transfer happent. All the fees and pnl will be in portfolio vault
		expect(balanceBeforeClosePos).eq(balanceAfterClosePos);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
	});

	it("Use the same token as collateral", async function () {
		const wbtcAm = precision.token(20, 18);
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

		let poolInfo = await poolFacet.getPoolWithOracle(xBtc, oracleBeginning);
		console.log("Pools balances base amount", poolInfo.baseTokenBalance.amount);
		console.log(
			"Pools balances unsettled base amount",
			poolInfo.baseTokenBalance.unsettledAmount
		);

		let balanceBeforeClosePos = await wbtc.balanceOf(xBtc);
		console.log(
			"Balance after closing the position in loss",
			balanceBeforeClosePos
		);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
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

		let positionInfo = await positionFacet.getSinglePosition(
			user0.address,
			btcUsd,
			wbtcAddr,
			true
		);

		// mimick fees
		await mine(1000, { interval: 300 });

		// close the position in losses, pool profit
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

		poolInfo = await poolFacet.getPoolWithOracle(xBtc, oracleBeginning);
		console.log(
			"Pools balances base amount after",
			poolInfo.baseTokenBalance.amount
		);
		console.log(
			"Pools balances unsettled base amount after",
			poolInfo.baseTokenBalance.unsettledAmount
		);

		let accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log("acc info", accountInfo);

		let balanceAfterClosePos = await wbtc.balanceOf(xBtc);
		console.log(
			"Balance after closing the position in loss",
			balanceAfterClosePos
		);

		// token transfer happent. All the fees and the pnl is in the pool!
		expect(balanceBeforeClosePos).lessThan(balanceAfterClosePos);
		///////////////////////////////////////////////////////////////////////////
		///////////////////////////////////////////////////////////////////////////
	});
```

## Impact
Double counting of fees for cross positions. Portfolio vault will be insolvent because of the double paid fees. Hence, high.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L60-L204

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L338-L414

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/storage/Account.sol#L131-L162

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/FeeProcess.sol#L139-L240

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/FeeProcess.sol#L76-L137

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MarketProcess.sol#L104-L127
## Tool used

Manual Review

## Recommendation
