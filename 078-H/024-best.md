Rough Emerald Gazelle

High

# OrderProcess._executeDecreaseOrder has the wrong logic for checking ``DecreaseOrderSideInvalid`` error, as a result, a user cannot close/decrease an existing long position and thus will lose funds.

## Summary
OrderProcess._execution/DecreaseOrder has the wrong logic for checking ``DecreaseOrderSideInvalid`` error, as a result, a user cannot close/decrease an existing long position and thus will lose funds. 

## Vulnerability Detail

OrderProcess._executionDecreaseOrder will be called in the following flow when a user tries to close/decrease, for example, a long position: OrderFacet.executeOrder() > OrderProcess.executeOrder() -> OrderProcess.__executeDecreaseOrder().

[https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L414-L451](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/OrderProcess.sol#L414-L451)

Obviously, a user can only close/decrease a position with the same side (long/short). Therefore the following lines are used to perform such a check: 

```javascript
if (position.isLong == isLong) {
            revert Errors.DecreaseOrderSideInvalid();
        }
```
Unfortunately, the condition is wrong: the condition should be ``position.isLong != isLong" instead of ``position.isLong == isLong`` for the error case.

As a result of this logic error, a user cannot close/decrease a position (with the same side). The user will lose funds.

## Impact

OrderProcess._executeDecreaseOrder has the wrong logic for checking ``DecreaseOrderSideInvalid`` error, as a result, a user cannot close/decrease an existing long position and thus will lose funds. 

The following POC confirms my finding (the complete file is avaiable upon request): the user opens a long and then a short position, but fails to close the long position due to the logical error above. 

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
            qty: 1 ether,
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

## Code Snippet

## Tool used

Foundry

Manual Review

## Recommendation
change the condition to its opposite:
```javascript
if (position.isLong != isLong) {
            revert Errors.DecreaseOrderSideInvalid();
        }
```
