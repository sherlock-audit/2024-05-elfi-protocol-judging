Active Punch Jellyfish

High

# Canceling a mint stake token can result in the execution fee being sent from the wrong vault

## Summary
When mint orders are cancelled, the user's deposit and the execution fees are returned. However, there is a scenario where the user's execution fee is taken from the LP vault instead of the portfolio vault, resulting in incorrect accounting.
## Vulnerability Detail
When mint orders are created with `params.isCollateral` set to `true` and `walletRequestTokenAmount` is a non-zero value, the funds will be taken from the `msg.sender` and deposited into the portfolio vault:

```solidity
function createMintStakeTokenRequest(MintStakeTokenParams calldata params) external payable override nonReentrant {
       .
       .
        -> if (params.walletRequestTokenAmount > 0) {
            require(!params.isNativeToken || msg.value == params.walletRequestTokenAmount, "Deposit eth amount error!");
            AssetsProcess.depositToVault(
                AssetsProcess.DepositParams(
                    account,
                    params.requestToken,
                    params.walletRequestTokenAmount,
                    -> params.isCollateral ? AssetsProcess.DepositFrom.MINT_COLLATERAL : AssetsProcess.DepositFrom.MINT,
                    params.isNativeToken
                )
            );
        }
        .
        (uint256 walletRequestTokenAmount, bool isExecutionFeeFromLpVault) = MintProcess
            .validateAndDepositMintExecutionFee(account, params);
        if (params.requestTokenAmount < walletRequestTokenAmount) {
            revert Errors.MintWithParamError();
        }
        .
    }
```

If the token is also a native token, the execution fee will be charged from the amount instead of a separate transfer:

```solidity
function validateAndDepositMintExecutionFee(
        address account,
        IStake.MintStakeTokenParams calldata params
    ) external returns (uint256, bool) {
        .
       ->  if (params.isNativeToken && params.walletRequestTokenAmount >= params.executionFee) {
            return (params.walletRequestTokenAmount - params.executionFee, true);
        }
        .
        .
    }
```

The return values will be the new net `walletRequestTokenAmount` and a `isExecutionFeeFromLpVault` boolean which is true.

If the user decides to cancel the order before execution, they can call the cancel mint function. Since `isExecutionFeeFromLpVault` was true for the request, the execution fee will be taken from the LP vault to send it to the user, which is incorrect since the user's funds never entered the LP pool but only the portfolio pool because the user deposited it as collateral.

```solidity
function cancelMintStakeToken(uint256 requestId, bytes32 reasonCode) external {
        .
        .
        -> GasProcess.processExecutionFee(
            GasProcess.PayExecutionFeeParams(
                mintRequest.isExecutionFeeFromLpVault
                    ? IVault(address(this)).getLpVaultAddress()
                    : IVault(address(this)).getPortfolioVaultAddress(),
                mintRequest.executionFee,
                startGas,
                msg.sender,
                mintRequest.account
            )
        );
    }
```


## Impact
"When funds are taken from the LP vault instead of the portfolio vault, some other user's request will fail because the LP vault must have the exact amount to perform the transaction. For instance, if a user has a deposit of 10 WETH in the LP vault, when it's executed, 10 WETH will be taken from the LP vault to stake the token. However, if someone withdraws the execution fee as described in the scenario above, then the LP vault will have only 9.998 WETH. This discrepancy means the other user's order will never go through and will also never be cancellable because the system will always assume 10 WETH is available in the LP vault. Considering all these factors, high severity.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/StakeFacet.sol#L21-L70

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/AssetsProcess.sol#L58-L79

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/MintProcess.sol#L108-L128

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/StakeFacet.sol#L101-L122

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/GasProcess.sol#L17-L41
## Tool used

Manual Review

## Recommendation
