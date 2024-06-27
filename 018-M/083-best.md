Overt Fiery Starfish

Medium

# Lack of oracle setting in autoReducePositions

## Summary
Function `autoReducePositions` will be always reverted because the price is not set.

## Vulnerability Detail
Function `autoReducePositions` is one key part of the whole system risk control. When some positions win too much profit in some extreme market conditions, the keeper will close these positions to decrease the whole system's risk.

The vulnerability is that we lack `setOraclePrice` in the function `autoReducePositions`. And the tokens' price is necessary when we close some positions. So this `autoReducePositions` will be reverted because of PriceIsZero().

```javascript
    function autoReducePositions(bytes32[] calldata positionKeys) external override {
        uint256 startGas = gasleft();
        RoleAccessControl.checkRole(RoleAccessControl.ROLE_KEEPER);
        uint256 requestId = UuidCreator.nextId(AUTO_REDUCE_ID_KEY);
        for (uint256 i; i < positionKeys.length; i++) {
            Position.Props storage position = Position.load(positionKeys[i]);
            position.checkExists();
            position.decreasePosition(
                DecreasePositionProcess.DecreasePositionParams(
                    requestId,
                    position.symbol,
                    false,
                    position.isCrossMargin,
                    position.marginToken,
                    position.qty,
                    OracleProcess.getLatestUsdUintPrice(position.indexToken, position.isLong)
                )
            );
        }
        GasProcess.addLossExecutionFee(startGas);
    }

```
```javascript
    function _getLatestUsdPriceWithOracle(address token) internal view returns (OraclePrice.Data memory) {
        OraclePrice.Props storage oracle = OraclePrice.load();
        OraclePrice.Data memory tokenPrice = oracle.getPrice(token);
        if (tokenPrice.min == 0 || tokenPrice.max == 0) {
            revert Errors.PriceIsZero();
        }
        return tokenPrice;
    }
```
### Poc
`autoReducePositions` will be reverted.
```javascript
  it.only('Case2.0: autoReducePositions', async function () {
    // Step 1: user0 create one position BTC
    console.log("User0 Long BTC ");
    const orderMargin1 = precision.token(1, 17) // 0.1BTC
    const btcPrice1 = precision.price(50000)
    const btcOracle1 = [{ token: wbtcAddr, minPrice: btcPrice1, maxPrice: btcPrice1 }]
    const executionFee = precision.token(2, 15)
    // Create one BTC position
    await handleOrder(fixture, {
      orderMargin: orderMargin1,
      oracle: btcOracle1,
      marginToken: wbtc,
      account: user0,
      symbol: btcUsd,
      executionFee: executionFee,
    })
    //
    let positionInfo = await positionFacet.getSinglePosition(user0.address, btcUsd, wbtcAddr, false)
    console.log(positionInfo.key)
    let tx = await positionFacet.connect(user3).autoReducePositions([positionInfo.key])
    })
```
#### Output
```shell
     Error: VM Exception while processing transaction: reverted with custom error 'PriceIsZero()'
    at OracleProcess._getLatestUsdPriceWithOracle (contracts/process/OracleProcess.sol:124)
    at OracleProcess.getLatestUsdPrice (contracts/process/OracleProcess.sol:102)
    at PositionFacet.autoReducePositions (contracts/facets/PositionFacet.sol:220)
    at Diamond.<fallback> (contracts/router/Diamond.sol:61)
    at processTicksAndRejections (node:internal/process/task_queues:95:5)
    at async HardhatNode._mineBlockWithPendingTxs (node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:1854:23)
    at async HardhatNode.mineBlock (node_modules/hardhat/src/internal/hardhat-network/provider/node.ts:524:16)
    at async EthModule._sendTransactionAndReturnHash (node_modules/hardhat/src/internal/hardhat-network/provider/modules/eth.ts:1546:18)
    at async HardhatNetworkProvider.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:124:18)
    at async HardhatEthersSigner.sendTransaction (node_modules/@nomicfoundation/hardhat-ethers/src/signers.ts:125:18)
    at async send (node_modules/ethers/src.ts/contract/contract.ts:299:20)

```
## Impact
`autoReducePositions` is one key part of the whole system's risk control. If `autoReducePositions` does not work, the whole system need to face more risk in one extreme market condition. Although the keeper role can call `OracleFacet::setOraclePrices` and `autoReducePositions` in one transaction to avoid this revert, I've already confirmed with the sponsor, the keeper role will call `autoReducePositions` directly.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/PositionFacet.sol#L196-L216

## Tool used

Manual Review

## Recommendation
Add `OracleProcess.setOraclePrice(oracles);` and `OracleProcess.clearOraclePrice();` in function `autoReducePositions`