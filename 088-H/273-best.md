Great Maroon Wasp

High

# In Cross Margin mode, the user’s profit calculation is incorrect.


## Summary
In Cross Margin mode, the user’s profit calculation is incorrect.
## Vulnerability Detail
We know that isolated and cross margin are different. When a position is created, in isolated mode, the corresponding assets need to be transferred from the user’s wallet to the MarketVault, while in cross margin mode, the user only needs to have sufficient collateral in the PortfolioVault (any supported collateral will do).
For example, with 1x leverage going long on WETH-USDC, the position size is 1 WETH, and the price of WETH is 1000 USD.
- In isolated mode, when establishing the position, 1 WETH is transferred to the MarketVault, so the borrowing is 0.
- In cross margin mode, assuming the collateral in the PortfolioVault is 10,000 USDC, no funds are transferred when creating the position. 
When the price of WETH rises to 2000 USD, closing the position makes it more evident.

- In isolated mode: The user profits 1000 USD (2000 USD - 1000 USD initial capital), and finally still gets their original 1 WETH (2000 USD), which is used for trading.

- In cross margin mode: The user profits 1000 USD (2000 USD - 1000 USD initial borrowed funds), and finally gets 0.5 WETH.

```javascript
function decreasePosition(Position.Props storage position, DecreasePositionParams calldata params) external {
        int256 totalPnlInUsd = PositionQueryProcess.getPositionUnPnl(position, params.executePrice.toInt256(), false);
        Symbol.Props memory symbolProps = Symbol.load(params.symbol);
        AppConfig.SymbolConfig memory symbolConfig = AppConfig.getSymbolConfig(params.symbol);
        FeeProcess.updateBorrowingFee(position, symbolProps.stakeToken);
        FeeProcess.updateFundingFee(position);
@>>        DecreasePositionCache memory cache = _updateDecreasePosition(
            position,
            params.decreaseQty,
            totalPnlInUsd,
            params.executePrice.toInt256(),
            symbolConfig.closeFeeRate,
            params.isLiquidation,
            params.isCrossMargin
        );
       //skip .........
       
    }
```

```javascript
   function _updateDecreasePosition(
        Position.Props storage position,
        uint256 decreaseQty,
        int256 pnlInUsd,
        int256 executePrice,
        uint256 closeFeeRate,
        bool isLiquidation,
        bool isCrossMargin
    ) internal view returns (DecreasePositionCache memory cache) {
        cache.position = position;
        cache.executePrice = executePrice;
        int256 tokenPrice = OracleProcess.getLatestUsdPrice(position.marginToken, false);
        cache.marginTokenPrice = tokenPrice.toUint256();
        uint8 tokenDecimals = TokenUtils.decimals(position.marginToken);
        if (position.qty == decreaseQty) {
@>>            cache.decreaseMargin = cache.position.initialMargin;
            cache.decreaseMarginInUsd = cache.position.initialMarginInUsd;
            cache.unHoldPoolAmount = cache.position.holdPoolAmount;
            (cache.settledBorrowingFee, cache.settledBorrowingFeeInUsd) = FeeQueryProcess.calcBorrowingFee(
                decreaseQty,
                position
            );
            cache.settledFundingFee = cache.position.positionFee.realizedFundingFee;
            cache.settledFundingFeeInUsd = cache.position.positionFee.realizedFundingFeeInUsd;

            cache.closeFeeInUsd = cache.position.positionFee.closeFeeInUsd;
            cache.closeFee = FeeQueryProcess.calcCloseFee(tokenDecimals, cache.closeFeeInUsd, tokenPrice.toUint256());
            cache.settledFee =
                cache.settledBorrowingFee.toInt256() +
                cache.settledFundingFee +
                cache.closeFee.toInt256();
//skip .......
            {
                cache.settledMargin = CalUtils.usdToTokenInt(
                    cache.position.initialMarginInUsd.toInt256() - _getPosFee(cache) + pnlInUsd,
                    TokenUtils.decimals(cache.position.marginToken),
                    tokenPrice
                );
@>>             cache.recordPnlToken = cache.settledMargin - cache.decreaseMargin.toInt256();
                cache.poolPnlToken =
                    cache.decreaseMargin.toInt256() -
                    CalUtils.usdToTokenInt(
                        cache.position.initialMarginInUsd.toInt256() + pnlInUsd,
                        TokenUtils.decimals(cache.position.marginToken),
                        tokenPrice
                    );
            }
            cache.realizedPnl = CalUtils.tokenToUsdInt(
                cache.recordPnlToken,
                TokenUtils.decimals(cache.position.marginToken),
                tokenPrice
            );
            console2.log("cache.position.initialMarginInUsd is", cache.position.initialMarginInUsd);
            console2.log("cache.settledMargin is ", cache.settledMargin);
            
            console2.log("cache.recordPnlToken is ", cache.recordPnlToken);
            console2.log("cache.poolPnlToken is ", cache.poolPnlToken);
            console2.log("cache.realizedPnl is ", cache.realizedPnl);

        } 
        //skip ......    

        return cache;
    }

```
However, in _updateDecreasePosition, cache.recordPnlToken = cache.settledMargin - cache.decreaseMargin.toInt256(), where cache.decreaseMargin = cache.position.initialMargin, causing cache.recordPnlToken to be nearly zero. 
This is incorrect in cross margin mode, because in cross margin mode, the initialMargin (with) is not invested in the market. Therefore, cache.recordPnlToken = cache.settledMargin.
#### poc
For example, with 1x leverage going long on WETH-USDC, the position size is 1 WETH, and the price of WETH is 1000 USD.
When the price of WETH rises to 2000 USD, closing the position makes it more evident.

```javascript
function testCrossMarginOrderExecute() public{
        ethPrice = 1000e8;
        usdcPrice = 101e6;
        OracleProcess.OracleParam[] memory oracles = getOracles(ethPrice, usdcPrice);

        userDeposit();
        depositWETH();

        openCrossMarginOrder();

        //after a day
        skip(1 days);
        ethPrice = 2000e8;
        closeCrossMarginLongPosition();

        getPoolWithOracle(oracles);
    
    }
```
Test the base code to verify this.
```bash
    totalPnlInUsd is  998900000000000000000
    cache.position.initialMarginInUsd is 998900000000000000000
    cache.settledMargin is  997387665400000000
    cache.recordPnlToken is  -1512334600000000
     cache.poolPnlToken is  0
    cache.realizedPnl is  -3024669200000000000
  
```
It can be seen that the profit is a negative value close to zero, which is obviously incorrect.
## Impact
This causes financial loss for either the user or the protocol.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L60

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L206
## Tool used

Manual Review

## Recommendation
Distinguish between the handling methods for isolated mode and cross margin mode.