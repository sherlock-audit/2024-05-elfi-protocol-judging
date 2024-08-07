Great Maroon Wasp

High

# In Cross Margin mode, the calculation for users borrowing from the pool is incorrect


## Summary
In Cross Margin mode, the calculation for users borrowing from the pool is incorrect.
## Vulnerability Detail
We know that isolated and cross margin are different. When a position is created, in isolated mode, the corresponding assets need to be transferred from the user’s wallet to the MarketVault, while in cross margin mode, the user only needs to have sufficient collateral in the PortfolioVault (any supported collateral will do).
For example, with 1x leverage going long on WETH-USDC, the position size is 1 WETH, and the price of WETH is 1000 USD.
- In isolated mode, when establishing the position, 1 WETH is transferred to the MarketVault, so the borrowing is 0.
- In cross margin mode, assuming the collateral in the PortfolioVault is 10,000 USDC, no funds are transferred when creating the position. Thus, 1000 USD is borrowed from the LpPool and used to purchase 1 WETH at the price of 1000 USD.

When the price of WETH rises to 2000 USD, closing the position makes it more evident.

- In isolated mode: The user profits 1000 USD (2000 USD - 1000 USD initial capital), and finally still gets their original 1 WETH (2000 USD), which is used for trading.

- In cross margin mode: The user profits 1000 USD (2000 USD - 1000 USD initial borrowed funds), and finally gets 0.5 WETH.
```javascript
function increasePosition(
        Position.Props storage position,
        Symbol.Props memory symbolProps,
        IncreasePositionParams calldata params
    ) external {
        uint256 fee = FeeQueryProcess.calcOpenFee(params.increaseMargin, params.leverage, symbolProps.code);
        FeeProcess.chargeTradingFee(fee, symbolProps.code, FeeProcess.FEE_OPEN_POSITION, params.marginToken, position);
        if (params.isCrossMargin) {
            Account.Props storage accountProps = Account.load(position.account);
            accountProps.unUseToken(params.marginToken, fee, Account.UpdateSource.CHARGE_OPEN_FEE);
            accountProps.subTokenWithLiability(params.marginToken, fee, Account.UpdateSource.CHARGE_OPEN_FEE);
        }

        uint256 increaseMargin = params.increaseMargin - fee;
        uint256 increaseMarginFromBalance = params.increaseMarginFromBalance > fee
            ? params.increaseMarginFromBalance - fee
            : 0;

        uint8 tokenDecimals = TokenUtils.decimals(params.marginToken);
        uint256 increaseQty = CalUtils.tokenToUsd(
            CalUtils.mulRate(increaseMargin, params.leverage),
            tokenDecimals,
            params.marginTokenPrice
        );

        AppConfig.SymbolConfig memory symbolConfig = AppConfig.getSymbolConfig(symbolProps.code);
        if (position.qty == 0) {
            position.marginToken = params.marginToken;
            position.entryPrice = params.indexTokenPrice;
            position.initialMargin = increaseMargin;
            position.initialMarginInUsd = CalUtils.tokenToUsd(increaseMargin, tokenDecimals, params.marginTokenPrice);
            position.initialMarginInUsdFromBalance = CalUtils.tokenToUsd(
                increaseMarginFromBalance,
                tokenDecimals,
                params.marginTokenPrice
            );
            position.positionFee.closeFeeInUsd = CalUtils.mulRate(increaseQty, symbolConfig.closeFeeRate);
            position.qty = increaseQty;
            position.leverage = params.leverage;
            position.realizedPnl = -(CalUtils.tokenToUsd(fee, tokenDecimals, params.marginTokenPrice).toInt256());
            position.positionFee.openBorrowingFeePerToken = MarketQueryProcess.getCumulativeBorrowingFeePerToken(
                symbolProps.stakeToken,
                params.isLong,
                params.marginToken
            );
            position.positionFee.openFundingFeePerQty = MarketQueryProcess.getFundingFeePerQty(
                symbolProps.code,
                params.isLong
            );
        } else {
            FeeProcess.updateBorrowingFee(position, symbolProps.stakeToken);
            FeeProcess.updateFundingFee(position);
            position.entryPrice = CalUtils.computeAvgEntryPrice(
                position.qty,
                position.entryPrice,
                increaseQty,
                params.indexTokenPrice,
                symbolConfig.tickSize,
                params.isLong
            );
            position.initialMargin += increaseMargin;
            position.initialMarginInUsd += CalUtils.tokenToUsd(increaseMargin, tokenDecimals, params.marginTokenPrice);
            position.initialMarginInUsdFromBalance += CalUtils.tokenToUsd(
                increaseMarginFromBalance,
                tokenDecimals,
                params.marginTokenPrice
            );
            position.positionFee.closeFeeInUsd += CalUtils.mulRate(increaseQty, symbolConfig.closeFeeRate);
            position.qty += increaseQty;
            position.realizedPnl += -(CalUtils.tokenToUsd(fee, tokenDecimals, params.marginTokenPrice).toInt256());
        }

        position.lastUpdateTime = ChainUtils.currentTimestamp();
@>>       uint256 increaseHoldAmount = CalUtils.mulRate(increaseMargin, (params.leverage - 1 * CalUtils.RATE_PRECISION));
@>>        position.holdPoolAmount += increaseHoldAmount;

        /// update & verify OI
        MarketProcess.updateMarketOI(
            MarketProcess.UpdateOIParams(
                true,
                symbolProps.code,
                params.marginToken,
                increaseQty,
                params.indexTokenPrice,
                params.isLong
            )
        );

@>>        LpPoolProcess.holdPoolAmount(symbolProps.stakeToken, position.marginToken, increaseHoldAmount, params.isLong);

        position.emitOpenPositionUpdateEvent(
            params.requestId,
            Position.PositionUpdateFrom.ORDER_INCREASE,
            params.indexTokenPrice,
            fee
        );
    }
```
The above 10,000 USDC collateral could entirely be 10 BTC (or other non-market trading pair tokens). It can be seen that the code in increaseHoldAmount does not distinguish between isolated mode and cross margin mode, which is therefore incorrect.
## Impact
This results in financial loss for the protocol borrow fee.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/IncreasePositionProcess.sol#L34
## Tool used

Manual Review

## Recommendation
Distinguish between the handling methods for isolated mode and cross margin mode.
