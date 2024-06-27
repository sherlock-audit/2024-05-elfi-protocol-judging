Overt Fiery Starfish

High

# redeem stake token may be Dos because there is not enough balance in stake pool.

## Summary
Funds will be transferred to portfolio vault if the staker stake via `MINT_COLLATERAL`, and be transferred to stake LP Pool if the staker stake via `MINT`. When LP holdrers redeem tokens, all tokens will come from LP Pool. This can lead to redeem reverted because there is not enough balance. 

## Vulnerability Detail
When liquidity providers want to stake liquidity, liquidity providers can stake via `MINT_COLLATERAL` or `MINT`. The liquidity will be transferred to different vault, depending on mint method. 
Liquidity will be transferred to portfolio vault when `isCollateral` = true, otherwise will be transferred to stake LP Pool at last.

```javascript
    function createMintStakeTokenRequest(MintStakeTokenParams calldata params) external payable override nonReentrant {
        ...
        if (params.walletRequestTokenAmount > 0) {
            require(!params.isNativeToken || msg.value == params.walletRequestTokenAmount, "Deposit eth amount error!");
            AssetsProcess.depositToVault(
                AssetsProcess.DepositParams(
                    account,
                    params.requestToken,
                    params.walletRequestTokenAmount,
                    params.isCollateral ? AssetsProcess.DepositFrom.MINT_COLLATERAL : AssetsProcess.DepositFrom.MINT,
                    params.isNativeToken
                )
            );
        }
```
```javascript
    function depositToVault(DepositParams calldata params) public returns (address) {
        IVault vault = IVault(address(this));
        address targetAddress;
        // get related vault
        if (DepositFrom.MANUAL == params.from || DepositFrom.MINT_COLLATERAL == params.from) {
            targetAddress = vault.getPortfolioVaultAddress();
        } else if (DepositFrom.ORDER == params.from) {
            targetAddress = vault.getTradeVaultAddress();
        } else if (DepositFrom.MINT == params.from) {
            targetAddress = vault.getLpVaultAddress();
        }
```
The vulnerability is that when LP holders try to redeem tokens, all redeem tokens will come from LP Vault. This could lead to redeem reverted because there may not be enough balance.

The hacker can deposit via `isCollateral` = true to transfer tokens to portfolio vault and increase LP pool's share amount. And then the hacker can redeem tokens from LP pool. This will cause other normal LP holders cannot redeem tokens.  Even if there is no hacker, the system may meet this case in normal scenairo.

### Poc
Add this test case into mintStakeToken.test.ts, user0 stake with `isCollateral` = true, and then user1 stakes with `isCollateral` = false. Then user1 redeems tokens, and after that, user0 cannot redeem his tokens. 
```javascript
  it.only('Case3.1: Stake with mint_collateral', async function () {
    const stakeToken = await ethers.getContractAt('StakeToken', xEth)
    const preWEthTokenBalance = BigInt(await weth.balanceOf(user0.address))
    const preEthTokenBalance = BigInt(await ethers.provider.getBalance(user0.address))
    const preWEthVaultBalance = BigInt(await weth.balanceOf(lpVaultAddr))
    const preEthVaultBalance = BigInt(await ethers.provider.getBalance(wethAddr))
    const preWEthMarketBalance = BigInt(await weth.balanceOf(xEth))

    const preStakeTokenBalance = BigInt(await stakeToken.balanceOf(user0.address))

    const tokenPrice = precision.price(1800)
    const oracle = [{ token: wethAddr, minPrice: tokenPrice, maxPrice: tokenPrice }]

    const executionFee = precision.token(2, 15)
    // user0 mint
    await handleMint(fixture, {
      requestToken: weth,
      requestTokenAmount: precision.token(300),
      oracle: oracle,
      account: user0,
      isNativeToken: false,
      isCollateral: true,
      executionFee: executionFee,
    })
    //console.log(weth.balanceOf(stakeToken))
    let stakeWethBalance = BigInt(await weth.balanceOf(stakeToken))
    let portfolioVaultWethBalance = BigInt(await weth.balanceOf(portfolioVaultAddr))
    console.log(stakeWethBalance)
    console.log(portfolioVaultWethBalance)
    // user1 mint
    await handleMint(fixture, {
      requestToken: weth,
      requestTokenAmount: precision.token(300),
      oracle: oracle,
      account: user1,
      isNativeToken: false,
      isCollateral: false,
      executionFee: executionFee,
    })
    stakeWethBalance = BigInt(await weth.balanceOf(stakeToken))
    // Dump information
    console.log(stakeWethBalance)
    console.log(portfolioVaultWethBalance)
    // user1 redeem
    const tokenPrice1 = precision.price(1800)
    console.log(BigInt(await stakeToken.balanceOf(user0))) // 299.xxx, fees
    console.log(BigInt(await stakeToken.balanceOf(user1)))
    await handleRedeem(fixture, {
        unStakeAmount: precision.token(299),
        account: user1,
        receiver: user1.address,
        oracle: [{ token: wethAddr, minPrice: tokenPrice1, maxPrice: tokenPrice1 }],
    })
    console.log()
    // user0 cannot redeem
    stakeWethBalance = BigInt(await weth.balanceOf(stakeToken))
    console.log(BigInt(await stakeToken.balanceOf(user0))) // 299.xxx, fees
    console.log(stakeWethBalance)
    await handleRedeem(fixture, {
        unStakeAmount: precision.token(200),
        account: user0,
        receiver: user0.address,
        oracle: [{ token: wethAddr, minPrice: tokenPrice1, maxPrice: tokenPrice1 }],
    })
  })
```
## Impact
LP holders can not redeem tokens. 

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/StakeFacet.sol#L44-L55

## Tool used

Manual Review

## Recommendation
transfer funds from the portfolio vault to the market vault during the minting process