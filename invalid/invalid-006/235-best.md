Great Maroon Wasp

High

# When a user opens a short position, there is a lack of checks on the liquidity pool, which can result in the user being unable to realize their profits if they succeed.


## Summary
When a user opens a short position, there is a lack of checks on the liquidity pool, which can result in the user being unable to realize their profits if they succeed.
## Vulnerability Detail
```javascript
function executeOrder(
        uint256 orderId,
        Order.OrderInfo memory order
    ) external {
        Symbol.Props memory symbolProps = Symbol.load(order.symbol);

        _validExecuteOrder(order, symbolProps);
        if (Order.PositionSide.INCREASE == order.posSide) {
            _executeIncreaseOrder(orderId, order, symbolProps);
        } else if (Order.PositionSide.DECREASE == order.posSide) {
            _executeDecreaseOrder(orderId, order, symbolProps);
        }
        Order.remove(orderId);
    }
```

```javascript
function _executeIncreaseOrder(
        uint256 orderId,
        Order.OrderInfo memory order,
        Symbol.Props memory symbolProps
    ) internal {
        if (
            order.posSide == Order.PositionSide.INCREASE &&
            order.orderSide == Order.Side.SHORT &&
            PositionQueryProcess.hasOtherShortPosition(
                order.account,
                order.symbol,
                order.marginToken,
                order.isCrossMargin
            )
        ) {
            revert Errors.OnlyOneShortPositionSupport(order.symbol);
        }
        Account.Props storage accountProps = Account.load(order.account);

        ExecuteIncreaseOrderCache memory cache;
        cache.isLong = Order.Side.LONG == order.orderSide;

        MarketProcess.updateMarketFundingFeeRate(symbolProps.code);
        MarketProcess.updatePoolBorrowingFeeRate(
            symbolProps.stakeToken,
            cache.isLong,
            order.marginToken
        );

        cache.executionPrice = _getExecutionPrice(
            order,
            symbolProps.indexToken
        );
        if (symbolProps.indexToken == order.marginToken) {
            cache.marginTokenPrice = cache.executionPrice;
        } else {
            cache.marginTokenPrice = OracleProcess.getLatestUsdUintPrice(
                order.marginToken,
                !cache.isLong
            );
        }

        (
            cache.orderMargin,
            cache.orderMarginFromBalance
        ) = _executeIncreaseOrderMargin(
            order,
            accountProps,
            cache.marginTokenPrice
        );

        if (!order.isCrossMargin) {
            VaultProcess.transferOut(
                IVault(address(this)).getTradeVaultAddress(),
                order.marginToken,
                symbolProps.stakeToken,
                cache.orderMargin
            );
        }

        Position.Props storage positionProps = Position.load(
            order.account,
            symbolProps.code,
            order.marginToken,
            order.isCrossMargin
        );
        if (positionProps.qty == 0) {
            if (
                accountProps.hasOtherOrder(orderId) &&
                _getOrderLeverage(
                    accountProps,
                    symbolProps.code,
                    order.orderSide,
                    order.isCrossMargin,
                    order.leverage
                ) !=
                order.leverage
            ) {
                revert Errors.UpdateLeverageError(
                    order.account,
                    symbolProps.code,
                    Order.Side.LONG == order.orderSide,
                    _getOrderLeverage(
                        accountProps,
                        symbolProps.code,
                        order.orderSide,
                        order.isCrossMargin,
                        order.leverage
                    ),
                    order.leverage
                );
            }
            bytes32 key = Position.getPositionKey(
                order.account,
                symbolProps.code,
                order.marginToken,
                order.isCrossMargin
            );
            positionProps.key = key;
            positionProps.account = order.account;
            positionProps.indexToken = symbolProps.indexToken;
            positionProps.symbol = symbolProps.code;
            positionProps.marginToken = order.marginToken;
            positionProps.leverage = order.leverage;
            positionProps.isLong = cache.isLong;
            positionProps.isCrossMargin = order.isCrossMargin;
            accountProps.addPosition(key);
        } else if (positionProps.leverage != order.leverage) {
            revert Errors.UpdateLeverageError(
                order.account,
                symbolProps.code,
                Order.Side.LONG == order.orderSide,
                positionProps.leverage,
                order.leverage
            );
        }
        positionProps.increasePosition(
            symbolProps,
            IncreasePositionProcess.IncreasePositionParams(
                orderId,
                order.marginToken,
                cache.orderMargin,
                cache.orderMarginFromBalance,
                cache.marginTokenPrice,
                cache.executionPrice,
                order.leverage,
                cache.isLong,
                order.isCrossMargin
            )
        );
        accountProps.delOrder(orderId);

        emit OrderFilledEvent(
            orderId,
            order,
            block.timestamp,
            cache.executionPrice
        );
    }
```
We can see that in executeOrder and its function call chain, there is no check for pool liquidity. The reason long users can ensure their profits are realized is that they can borrow the holdAmount from the pool, ensuring liquidity can be realized.
#### poc
In increaseMarketOrder.test.ts, Comment out the code for minting xEth in `beforeEach`,like this.
```javascript
// const ethTokenPrice = precision.price(1600)
    // const ethOracle = [{ token: wethAddr, minPrice: ethTokenPrice, maxPrice: ethTokenPrice }]
    // await handleMint(fixture, {
    //   requestTokenAmount: precision.token(500),
    //   oracle: ethOracle,
    // })

```
So there will be no liquidity in pool(market) of weth-usdc. we add only to `it.only('Case4: Place Single Market Order Single User/Single Price/Short/USDT', async function () {`, This way, we will only run this one test case.
We get 
```bash
 ✔ Case4: Place Single Market Order Single User/Single Price/Short/USDT (768ms)


  1 passing (10s)
```
It is evident that in the case of zero liquidity, short positions are still being created.
## Impact
Short-position users are unable to realize their profits when they make gains.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L102

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/OrderProcess.sol#L114
## Tool used

Manual Review

## Recommendation
Add relevant liquidity checks.
