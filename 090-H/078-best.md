Overt Fiery Starfish

High

# Incorrect pnl calculation in getPoolIntValue

## Summary
In function getPoolIntValue(), `unPnl` calculated by `getMarketUnPnl` is not the actual market's pnl. This could lead to share's price's incorrect calculation.

## Vulnerability Detail
When users want to stake tokens in one pool, the contract will calculate the pool's whole value via `getPoolValue`, and then calculate the share's price according to the pool's whole value.

```javascript
    function computeStakeAmountFromMintToken(
        LpPool.Props storage pool,
        uint256 mintAmount
    ) public view returns (uint256) {
        uint256 totalSupply = TokenUtils.totalSupply(pool.stakeToken);
        uint8 tokenDecimals = TokenUtils.decimals(pool.baseToken);
        uint256 poolValue = pool.getPoolValue();
        ......
            uint256 baseMintAmountInUsd = CalUtils.tokenToUsd(
                mintAmount,
                tokenDecimals,
                // correct, low price
                OracleProcess.getLatestUsdUintPrice(pool.baseToken, true)
            );
            mintStakeTokenAmount = totalSupply.mul(baseMintAmountInUsd).div(poolValue);
        }
        return mintStakeTokenAmount;
    }
``` 
Function `getPoolIntValue` aims to calculate the whole value. The whole value include current pool balance , and unrealised Pnl. The vulnerability is `unPnl`'s calculation is not correct.
In function `getMarketUnPnl`, we calculate `pnlInUsd` according the entry price and current price. However, we should not use `usdToTokenInt` to calculate `pnlToken`. This may be larger or less than the actual `pnlToken` amount.
For example:
- Alice creates one LONG 1 ETH order, leverage is 2. Assume current ETH price is 100 USD. Position size is 200.
- Assume ETH's price increase from 100 to 150.
- pnlInUsd should be 200 * (150 - 100)/ 100 = 100
- pnlInToken should be 100/150 = 0.67 Ether. This result is based on the calculation in `getMarketUnPnl`.
- However, when Alice wants to close her position, the actual `pnlIntoken`'s calculation is different with the calculation in `getMarketUnPnl`.
- The calculation in `_updateDecreasePosition` is as below:
- The result from `_updateDecreasePosition`'s calculation should be (100(initialMarginInUsd) + 100(pnlInUsd))/150 - 1 Ether = 0.33 Ether
- 
```javascript
                cache.settledMargin = CalUtils.usdToTokenInt(
                    cache.position.initialMarginInUsd.toInt256() - _getPosFee(cache) + pnlInUsd,
                    TokenUtils.decimals(cache.position.marginToken),
                    tokenPrice
                );
                cache.recordPnlToken = cache.settledMargin - cache.decreaseMargin.toInt256();
```
```javascript
    function getPoolIntValue(
        LpPool.Props storage pool,
        OracleProcess.OracleParam[] memory oracles
    ) public view returns (int256) {
        int256 value = 0;
        if (pool.baseTokenBalance.amount > 0 || pool.baseTokenBalance.unsettledAmount > 0) {
            int256 unPnl = getMarketUnPnl(pool.symbol, oracles, true, pool.baseToken, true);
            int256 baseTokenPrice = OracleProcess.getIntOraclePrices(oracles, pool.baseToken, true);
            value = CalUtils.tokenToUsdInt(
                (pool.baseTokenBalance.amount.toInt256() + pool.baseTokenBalance.unsettledAmount + unPnl),
                TokenUtils.decimals(pool.baseToken),
                baseTokenPrice
            );
        }
    function getMarketUnPnl(
        bytes32 symbol,
        OracleProcess.OracleParam[] memory oracles,
        bool isLong,
        address marginToken,
        bool pnlToken
    ) public view returns (int256) {
        Market.Props storage market = Market.load(symbol);
        Symbol.Props memory symbolProps = Symbol.load(symbol);
        Market.MarketPosition storage position = isLong ? market.longPosition : market.shortPositionMap[marginToken];
        int256 markPrice = OracleProcess.getIntOraclePrices(oracles, symbolProps.indexToken, true);
        if (isLong) {
            int pnlInUsd = position.openInterest.toInt256().mul(markPrice.sub(position.entryPrice.toInt256())).div(
                position.entryPrice.toInt256()
            );
            if (pnlToken) {
                int256 marginTokenPrice = OracleProcess.getIntOraclePrices(oracles, marginToken, false);
                return -CalUtils.usdToTokenInt(pnlInUsd, TokenUtils.decimals(marginToken), marginTokenPrice);
            }
```

### Poc
Add below test into decreaseMarketOrder.test.ts, the whole procedure is like as below:
1. stake some tokens in xETH pool. This has already done in setup().
2. create one LONG ETH position.
3. ETH price increases from 1000 to 2000, which means we have some positive market unPnl.
4. stake some tokens in xETH pool, record share's price.
5. Close the LONG ETH Position.
6. stake some tokens in xETH pool again, we find step6 share's price will be larger than step4. 
```javascript
  it.only("Case2: Win profit test", async function () {
      //Mint at first
     let executionFee = precision.token(2, 15)
     const orderMargin1 = precision.token(2); // 2ETH
     const ethPrice0 = precision.price(1000);
     const ethOracle0 = [
       { token: wethAddr, minPrice: ethPrice0, maxPrice: ethPrice0 },
     ];
     let oracle = [{ token: wethAddr, minPrice: ethPrice0, maxPrice: ethPrice0 }]

     const leverage = BigInt(1);
     // new position
     await handleOrder(fixture, {
       symbol: ethUsd,
       orderMargin: orderMargin1,
       oracle: ethOracle0,
       executionFee: executionFee,
       leverage: precision.rate(2)
     });

     const poolInfo = await poolFacet.getPool(xEth);
     const symbolInfo = await marketFacet.getSymbol(ethUsd);
     const leverageMargin = precision.mulRate(
       orderMargin1,
       precision.rate(leverage)
     );
     const openFee = precision.mulRate(
       leverageMargin,
       symbolInfo.config.openFeeRate
     );
     const initialMargin = orderMargin1 - openFee;

     const defaultMarginMode = false;
     let positionInfo = await positionFacet.getSinglePosition(
       user0.address,
       ethUsd,
       wethAddr,
       defaultMarginMode
     );
     expect(true).to.equals(positionInfo.isLong);

     const ethPrice1 = precision.price(3000);
     // stake
    oracle = [{ token: wethAddr, minPrice: ethPrice1, maxPrice: ethPrice1 }]
    // user0 mint
    await handleMint(fixture, {
      requestToken: weth,
      requestTokenAmount: precision.token(300),
      oracle: oracle,
      account: user0,
      isNativeToken: false,
      isCollateral: false,
      executionFee: executionFee,
    })
     const closeQty1 = positionInfo.qty
     await handleOrder(fixture, {
       symbol: ethUsd,
       orderSide: OrderSide.SHORT,
       posSide: PositionSide.DECREASE,
       qty: closeQty1,
       oracle: oracle,
       executionFee: executionFee,
     });
     await handleMint(fixture, {
       requestToken: weth,
       requestTokenAmount: precision.token(300),
       oracle: oracle,
       account: user0,
       isNativeToken: false,
       isCollateral: false,
       executionFee: executionFee,
     })
  });

```
## Impact
Share's price's not correct. Considering share's price may be larger or less than actual share price when there're some unrealised pnl, stakers can stake/withdraw via frontrun/backrun to earn profits.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L110-L123
## Tool used

Manual Review

## Recommendation
Calculate the `unPnl` in `getPoolIntValue` correctly. This will keep the share's price stable if there is not index token price changes.