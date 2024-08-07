Overt Fiery Starfish

High

# Traders may decrease the loss via decrease the position's margin

## Summary
When traders decrease the position's margin, `_executeReduceMargin` did not consider current Pnl.

## Vulnerability Detail
In function `updatePositionMargin`, we can decrease one isolated position's margin and transfer some margin to users' account.
The vulnerability exists in function `_executeReduceMargin`. Function `_executeReduceMargin` will calculate `maxReduceMarginInUsd`. Users' `reduceMargin` cannot be larger than `maxReduceMarginInUsd`. The calculation of `maxReduceMarginInUsd` is `initialMarginInUsd` - margins for maximum leverage based on current position.

The vulnerability is that the system does not consider the positions' Pnl. If this position is at the edge of liquidation, he may get back a little funds by closing this position. However, the trader can get back more funds via decreasing margin. In below Poc, even if this position is unhealthy and needed to be liquidated and the user cannot close this position, the user can still get part of funds back via decreasing margin.

```javascript
    function updatePositionMargin(uint256 requestId, UpdatePositionMargin.Request memory request) external {
        Position.Props storage position = Position.load(request.positionKey);
       ......
        Symbol.Props memory symbolProps = Symbol.load(position.symbol);
        Account.Props storage accountProps = Account.load(request.account);
        //add margin, transfer from vault to LP Pool
        if (request.isAdd) {
        ......
        } else {
            // decrease margin, transfer from LP Pool to the trader.
            uint256 reduceMarginAmount = _executeReduceMargin(position, symbolProps, request.updateMarginAmount, true);
            VaultProcess.transferOut(symbolProps.stakeToken, request.marginToken, request.account, reduceMarginAmount);
            position.emitPositionUpdateEvent(requestId, Position.PositionUpdateFrom.DECREASE_MARGIN, 0);
        }
```
```javascript
    function _executeReduceMargin(
        Position.Props storage position,
        Symbol.Props memory symbolProps,
        uint256 reduceMargin,
        bool needUpdateLeverage
    ) internal returns (uint256) {
        AppConfig.SymbolConfig memory symbolConfig = AppConfig.getSymbolConfig(symbolProps.code);
        //calculate minimum margin for max leverage, left margin should not be less than minimum margin
        uint256 maxReduceMarginInUsd = position.initialMarginInUsd -
            CalUtils.divRate(position.qty, symbolConfig.maxLeverage).max(
                AppTradeConfig.getTradeConfig().minOrderMarginUSD
            );
```
### Poc
Add this part into increaseMarketOrder.test.ts. 
```javascript
  it.only('Case2.1: decrease margin to avoid the miss', async function () {
    // Step 1: user0 create one position BTC
    console.log("User0 Long BTC ");
    const orderMargin1 = precision.token(1, 17) // 0.1BTC
    const btcPrice1 = precision.price(50000)
    const btcOracle1 = [{ token: wbtcAddr, minPrice: btcPrice1, maxPrice: btcPrice1 }]
    const executionFee = precision.token(2, 15)

    await handleOrder(fixture, {
      orderMargin: orderMargin1,
      oracle: btcOracle1,
      marginToken: wbtc,
      account: user0,
      symbol: btcUsd,
      executionFee: executionFee,
    })
    // Cannot close this position, because of PositionShouldBeLiquidation
    /*
    const btcPrice2 = precision.price(40000)
    const btcOracle2 = [{ token: wbtcAddr, minPrice: btcPrice2, maxPrice: btcPrice2 }]
    let positionInfo = await positionFacet.getSinglePosition(user0.address, btcUsd, wbtcAddr, false)
     const closeQty1 = positionInfo.qty
     await handleOrder(fixture, {
       symbol: btcUsd,
       marginToken: wbtc,
       orderSide: OrderSide.SHORT,
       posSide: PositionSide.DECREASE,
       qty: closeQty1,
       oracle: btcOracle2,
       executionFee: executionFee,
     });
    */

    // Step 2: decrease margin
    console.log("User0 update position")
    // user0 use weth
    let positionInfo = await positionFacet.getSinglePosition(user0.address, btcUsd, wbtcAddr, false)
    console.log(positionInfo.key);
    console.log(positionInfo.initialMargin);
    let tx = await positionFacet.connect(user0).createUpdatePositionMarginRequest(
      {
        positionKey: positionInfo.key,
        isAdd: false,
        isNativeToken: false,
        marginToken: wbtc,
        updateMarginAmount: precision.token(2462, 18),
        executionFee: executionFee,
      },
      {
        value: executionFee,
      },
    )
    let requestId = await marketFacet.getLastUuid(UPDATE_MARGIN_ID_KEY)
    await tx.wait()
    console.log("Before execute :", await wbtc.balanceOf(user0.address))
    // Step 3: execute user0 update request
    const tokenPrice = precision.price(40000)
    const oracle = [{ token: wbtcAddr, targetToken: ethers.ZeroAddress, minPrice: tokenPrice, maxPrice: tokenPrice }]
    tx = await positionFacet.connect(user3).executeUpdatePositionMarginRequest(requestId, oracle)
    await tx.wait()
    console.log("After execute :", await wbtc.balanceOf(user0.address))
    
  })
```
## Impact
Traders can get back some funds via decreasing one position's margin. LP holders will lose some expected profits.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L370-L383

## Tool used

Manual Review

## Recommendation
Consider the unrealised Pnl when we decrease one position's margin.
