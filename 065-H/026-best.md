Rough Emerald Gazelle

High

# PositionQueryProcess.getPositionUnPnl() calculate the wrong amount of UnPnl, as a result, a user (or the Elfi protocol) might loss funds.

## Summary
PositionQueryProcess.getPositionUnPnl() calculate the wrong amount of UnPnl, as a result, a user (or the Elfi protocol) might loss funds.

## Vulnerability Detail
When a user decreases/close a position, PositionQueryProcess.getPositionUnPnl() is used to calculate the UnPnl for the position: 

[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionQueryProcess.sol#L80-L106](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/PositionQueryProcess.sol#L80-L106)

However, when calculating the amount for the margin token for UnPnl, the following code uses the price of the index token, not that of the margin token: 

```javascript
 if (position.isLong) {
            int pnlInUsd = position.qty.toInt256().mul(computeIndexPrice.sub(position.entryPrice.toInt256())).div(
                position.entryPrice.toInt256()
            );
            return (
                CalUtils.usdToTokenInt(pnlInUsd, TokenUtils.decimals(position.marginToken), computeIndexPrice),
                pnlInUsd
            );
```
As a result, the token amount for the margin token is wrongly calculated, could be more or less based on the difference of prices of the margin token and index token. As  a result, the user or the Elfi protocol might loss funds. 

The following POC confirms my finding, note how the price of the index token is used to calculate the amount of margin token. Complete code is available upon request. 

```javascript
 function testOrder() public{
         uint256 currentTime = block.timestamp;

         testStake();

         IOrder.PlaceOrderParams memory oparams = IOrder.PlaceOrderParams({
            symbol: keccak256(abi.encode("symbolA")),
            isCrossMargin: false,
            isNativeToken: true,
            orderSide: Order.Side.LONG,
            posSide: Order.PositionSide.INCREASE,
            orderType:  Order.Type.MARKET,
            stopType: Order.StopType.TAKE_PROFIT,
            marginToken: address(weth),
            qty: 1 ether,
            orderMargin: 1 ether,
            leverage: 10 * CalUtils.RATE_PRECISION, // 10X
            triggerPrice: 17000000000,
            acceptablePrice: 18000000000,
            executionFee: 1000,
            placeTime: block.timestamp
         });
         vm.startPrank(user1);
         OrderFacet(payable(address(diamond))).createOrderRequest{value: 1 ether}(oparams);
         vm.stopPrank();


        vm.startPrank(keeper1);
        OrderFacet(payable(address(diamond))).executeOrder(1112, getOracle1());
        vm.stopPrank();

        console2.log("User2 requests a short order...");
        IOrder.PlaceOrderParams memory oparams2 = IOrder.PlaceOrderParams({
            symbol: keccak256(abi.encode("symbolA")), // join this market
            isCrossMargin: false,
            isNativeToken: false,
            orderSide: Order.Side.SHORT,
            posSide: Order.PositionSide.INCREASE,
            orderType:  Order.Type.MARKET,
            stopType: Order.StopType.TAKE_PROFIT,
            marginToken: address(usdc),  // should be a stable token
            qty: 1 ether,
            orderMargin: 170 ether, // 170 dollars
            leverage: 10 * CalUtils.RATE_PRECISION, // 10X
            triggerPrice: 17000000000,
            acceptablePrice: 16000000000,  // must be higher than this, to pass, [17000, 18000]
            executionFee: 1000,
            placeTime: block.timestamp
         });
         vm.startPrank(user2);
         usdc.approve(address(diamond), 170 ether); // approve 1 ether
         OrderFacet(payable(address(diamond))).createOrderRequest{value: 1000}(oparams2);
         vm.stopPrank();

        vm.startPrank(keeper1);
        OrderFacet(payable(address(diamond))).executeOrder(1113, getOracle1());
        vm.stopPrank();

        vm.warp(currentTime + 1 days);
        // close the long order
        console2.log("********************************");
        console2.log("user1 closes his long order...");
        IOrder.PlaceOrderParams memory oparams3 = IOrder.PlaceOrderParams({
            symbol: keccak256(abi.encode("symbolA")),
            isCrossMargin: false,
            isNativeToken: true,
            orderSide: Order.Side.LONG,
            posSide: Order.PositionSide.DECREASE,
            orderType:  Order.Type.MARKET,
            stopType: Order.StopType.TAKE_PROFIT,
            marginToken: address(weth),
            qty: 25973999999999974026000,
            orderMargin: 1 ether,
            leverage: 10 * CalUtils.RATE_PRECISION, // 10X
            triggerPrice: 21000000000,
            acceptablePrice: 20000000000, // sold at least value
            executionFee: 1000,
            placeTime: block.timestamp
         });
         vm.startPrank(user1);
         OrderFacet(payable(address(diamond))).createOrderRequest{value: 1000}(oparams3);
         vm.stopPrank();

        vm.startPrank(keeper1);
        OrderFacet(payable(address(diamond))).executeOrder(1114, getOracle2());
        vm.stopPrank();
    }
```

## Impact
PositionQueryProcess.getPositionUnPnl() calculate the wrong amount of UnPnl, as a result, a user (or the Elfi protocol) might loss funds.

## Code Snippet

## Tool used
foundry

Manual Review

## Recommendation
Use the price of the margin token to calcualte the amount of margin token for unPNl. 
