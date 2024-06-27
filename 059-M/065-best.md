Rough Emerald Gazelle

Medium

# OrderProcess._validatePlaceOrder() fails to support limit sell order.

### 
## Summary
``OrderProcess._validatePlaceOrder()`` has a logic error as a result it fails to support limit sell order.

## Vulnerability Detail
``OrderProcess._validatePlaceOrder()`` is used to validate the correctness of an order parameters when an order request is made. 

[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L363-L412](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L363-L412)

However, the following code snipper has a logic errors that prevents it from supporting a limit sell order:

```javascript
 if (Order.Type.LIMIT == params.orderType && Order.PositionSide.DECREASE == params.posSide) {
            revert Errors.PlaceOrderWithParamsError();
        }
```

Our POC confirms this finding, complete code is available upon request:

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
        console2.log("user1 closes his long order with a limit sell order...");
        IOrder.PlaceOrderParams memory oparams3 = IOrder.PlaceOrderParams({
            symbol: keccak256(abi.encode("symbolA")),
            isCrossMargin: false,
            isNativeToken: true,
            orderSide: Order.Side.LONG,
            posSide: Order.PositionSide.DECREASE,
            orderType:  Order.Type.LIMIT,
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

        vm.warp(currentTime + 2 days);
        // close the long order to stop loss
        console2.log("********************************");
        console2.log("user2 closes his short order...");
        IOrder.PlaceOrderParams memory oparams4 = IOrder.PlaceOrderParams({
            symbol: keccak256(abi.encode("symbolA")),
            isCrossMargin: false,
            isNativeToken: true,
            orderSide: Order.Side.SHORT,
            posSide: Order.PositionSide.DECREASE,
            orderType:  Order.Type.MARKET,
            stopType: Order.StopType.TAKE_PROFIT,
            marginToken: address(usdc),
            qty: 1681317000000000000000,
            orderMargin: 1 ether,
            leverage: 10 * CalUtils.RATE_PRECISION, // 10X
            triggerPrice: 9000000000,
            acceptablePrice: 16900000000, // sold lower than this price
            executionFee: 1000,
            placeTime: block.timestamp
         });
         vm.startPrank(user2);
         OrderFacet(payable(address(diamond))).createOrderRequest{value: 1000}(oparams4);
         vm.stopPrank();

        vm.startPrank(keeper1);
        OrderFacet(payable(address(diamond))).executeOrder(1115, getOracle3());
        vm.stopPrank();
    }
```

## Impact
``OrderProcess._validatePlaceOrder()`` has a logic error as a result it fails to support limit sell order.


## Code Snippet

## Tool used
Foundry

Manual Review

## Recommendation
Delete this piece of code. 