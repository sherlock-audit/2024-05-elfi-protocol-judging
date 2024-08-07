Active Punch Jellyfish

Medium

# Unbacked tokens can be used for opening positions

## Summary
When a position is opened the leverage is taken from the corresponding pool. If the pools available liquidity is lower than the requested leverage amount then the operation can't be executed. However, there is an edge case where pools available liquidity can be mistakenly high where the opened position can use funds as its in pool but actually the liquidity is not enough.
## Vulnerability Detail
Here's the revised version of your scenario with improved grammar and clarity:

---

**Scenario Explanation:**

Assume there are two participants, Alice and Bob. Alice has $100k in ETH on a 20x long position, and Bob has $50 in ETH on a 10x short position both CROSS. The current ETH price is $1k, meaning Alice borrowed 95 ETH, and Bob borrowed $45k USDC. Also, assume they are the only market participants. 

The ETH pool has 100 ETH (baseAmount) and 0 unsettled, with a maximum borrow limit of 98 ETH (98%) based on the pool's liquidity limit factor.

Since the long open interest is higher than the short open interest, Alice will pay Bob funding fees in the form of ETH. Assume Alice later updates her position by adding a little more ETH, which increases the unsettled amount in the pool. The `IncreasePositionProcess::increasePosition` function calls `FeeProcess::updateFundingFee`, which in turn calls `MarketProcess::updateMarketFundingFee`, adding the unsettled amount to the pool's balance sheet. Let's assume this amount is 10 ETH.

```solidity
function updateMarketFundingFee(
        bytes32 symbol,
        int256 realizedFundingFeeDelta,
        bool isLong,
        bool needUpdateUnsettle,
        address marginToken
    ) external {
        if (needUpdateUnsettle) {
            Symbol.Props storage symbolProps = Symbol.load(symbol);
            LpPool.Props storage pool = LpPool.load(symbolProps.stakeToken);
            if (isLong) {
                -> pool.addUnsettleBaseToken(realizedFundingFeeDelta);
            } else {
                pool.addUnsettleStableToken(marginToken, realizedFundingFeeDelta);
            }
        }
    }
```

After Alice's top-up (very small amount just to update the unrealized fee), the pool now has:
- 100 baseAmount
- 95 holdAmount
- 10 unsettledAmount

Now, the pool's available liquidity is:
(100 + 10) * 98/100 = 107.8 ETH

With 95 holdAmount, the available borrowable amount is:
107.8 - 95 = 12.8 ETH

Assume Carol borrows this 12.8 ETH, which should be impossible since the pool only had 100 ETH initially, with 5 ETH remaining unborrowed. The total borrows are now 95 + 12.8 = 107.8 ETH. The updated pool state is:
- 100 baseAmount
- 107.8 holdAmount
- 10 unsettledAmount

Later, more shorters enter the market, and now shorters pay the longs. Alice decides to close her position when the ETH price is still $1k. Assume, she had -10 ETH in funding fees this time, now adjusted to "0" in total due to shorts paying Alice.

Upon closing her position, the base amount remains the same, but the unsettled amount decreases by 10, resulting in:
- 100 baseAmount
- 12.8 holdAmount
- 0 unsettledAmount

Also, assume that Alice was the only one longing and initially accrued -10 ETH, which means +10 ETH worth of USD in funding fees was credited to Bob. Then, Carol joined as long and borrowed the remaining 12.8 ETH. Additionally, Derek joined as a shorter. Bob and Derek's short open interest became high enough that it paid both Alice and Carol, resetting Alice's funding fees and accruing some to Carol. Overall, Carol is in funding fee positive profits, while Bob and Derek incur losses of some ETH. 

Carol took advantage of Bob's unpaid funding fees to create a leveraged position.
## Impact
Pool's available liquidity can be leveraged. "Realized" funding fees are not actually taken from the user when the position is updated; they are marked and finalized only upon closing the position. Realized funding fees can fluctuate because the transfers have not occurred, and the amounts are merely added on top of the existing balance.
## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/LpPoolQueryProcess.sol#L151-L191

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/IncreasePositionProcess.sol#L83-L104

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/FeeProcess.sol#L102-L137

https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/8a1a01804a7de7f73a04d794bf6b8104528681ad/elfi-perp-contracts/contracts/process/DecreasePositionProcess.sol#L60-L204
## Tool used

Manual Review

## Recommendation
