Wide Mocha Starfish

High

# No modifications to the `accountProps` while updating the position margin.

## Summary

When updating the position margin, the current implementation does not update the `tokenBalances` within the `accountProps`.

## Vulnerability Detail

When adding margin to a position, the user sends tokens to the trade vault when creating the request.

```solidity
    function createUpdatePositionMarginRequest(UpdatePositionMarginParams calldata params) external payable override {
        [...]

        if (params.isAdd) {
            require(!params.isNativeToken || msg.value == params.updateMarginAmount, "Deposit eth amount error!");
37          AssetsProcess.depositToVault(
                AssetsProcess.DepositParams(
                    account,
                    params.isNativeToken ? AppConfig.getChainConfig().wrapperToken : params.marginToken,
                    params.updateMarginAmount,
                    AssetsProcess.DepositFrom.ORDER,
                    params.isNativeToken
                )
            );
        }

        [...]
    }
```

And when reducing the margin of a position, tokens are transferred back to the user.

```solidity
    function updatePositionMargin(uint256 requestId, UpdatePositionMargin.Request memory request) external {
        [...]

            uint256 reduceMarginAmount = _executeReduceMargin(position, symbolProps, request.updateMarginAmount, true);
119         VaultProcess.transferOut(symbolProps.stakeToken, request.marginToken, request.account, reduceMarginAmount);
            position.emitPositionUpdateEvent(requestId, Position.PositionUpdateFrom.DECREASE_MARGIN, 0);
        
        [...]
    }
```

Therefore, the value of `tokenBalances[token].amount` and `tokenBalances[token].usedAmount` in the `accountProps` should be updated accordingly. However, there is currently no logic implemented to handle this during the entire process of updating the position margin. It impacts important functions of the entire protocol, such as the `AccountProcess.getCrossAvailableValue()` function, leading to incorrect behaviors.

## Impact

Lack of modification of `accountProps` during updating position margin results in incorrect behaviors of the entire protocol.

## Code Snippet

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/PositionFacet.sol#L22-L59

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/process/PositionMarginProcess.sol#L91-L132

## Tool used

Manual Review

## Recommendation

Updating the position margin should also result in the corresponding modification of the `accountProps`.