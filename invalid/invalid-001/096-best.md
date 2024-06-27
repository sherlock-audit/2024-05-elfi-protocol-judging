Soaring Tawny Fly

Medium

# Malicious Users Can Create Fake Withdrawal Requests, Leading to Potential Keeper Fund Losses and Increased Network Traffic.

## Summary
Lack of Deposit Verification in `AccountFacet.sol:createWithdrawRequest` Allows Attackers to Create Fake Withdrawal Requests, Resulting in Keeper Fee Losses.
## Vulnerability Detail
The createWithdrawRequest function in AccountFacet.sol is designed to create a withdrawal request for a specified token and amount. However, the function does not verify whether the user has sufficient deposited funds before creating the withdrawal request. This omission can be exploited by malicious users to create multiple fake withdrawal requests, leading to potential losses for keepers and increased network traffic.
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/AccountFacet.sol#L40-L46
### Issue: 
The function does not check if the user has deposited any funds or if the user’s deposited amount is sufficient to cover the withdrawal request.
## Impact
Malicious users can create numerous fake withdrawal requests without having the necessary deposits. This can lead to several issues:
1. Financial Loss: Keepers executing these invalid requests will incur transaction costs and potentially lose funds.
2. Network Congestion: Increased traffic due to fake withdrawal requests can lead to network congestion and higher gas fees for legitimate transactions.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/main/elfi-perp-contracts/contracts/facets/AccountFacet.sol#L40-L46
## Tool used

Manual Review

## Recommendation
Add some check's to verify that user has some deposits in the vault.