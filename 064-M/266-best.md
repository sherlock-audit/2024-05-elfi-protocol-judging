Soft Blonde Hamster

Medium

# The OI ratio limit rather prevents from enhancing OI ratio.

## Summary
The OI ratio limit rather prevents from enhancing OI ratio.

## Vulnerability Detail
The OI ratio limit is in effect only when the smaller side is bigger than `minOpenInterest`.
Therefore, it prevents from enhancing the balance of OI ratio when the bigger side is already way bigger than 
`minOpenInterest` and the smaller side starts to increase over `minOpenInterest`.

For example, suppose `minOpenInterest` is $1M  and `maxLongOpenInterestCap` and `maxShortOpenInterestCap` are $100M.
1. Traders open long positions up to nearly $100M.
2. If other traders want to open short positions over $1M, they can't because of the OI ratio check. 
```solidity
            if (params.isAdd) {
                // ...
                uint256 minOpenInterest = longOpenInterest.min(shortOpenInterest);
                // @audit OIRatioLimited check does not start until minOpenInterest is greater than longShortOiBottomLimit 
                if (minOpenInterest < symbolConfig.longShortOiBottomLimit) {
                    return;
                }
                // @audit If max open interest is way greater than min open interest, the OIRatioLimited check will prevent the smaller side from increasing
                if (
                    longOpenInterest.max(shortOpenInterest) - minOpenInterest >
                    CalUtils.mulRate(longOpenInterest + shortOpenInterest, symbolConfig.longShortRatioLimit)
                ) {
                    revert Errors.OIRatioLimited();
                }
            }
```

## Impact
The OI ratio check rather results in a bad effect in OI ratio.

## Code Snippet
https://github.com/sherlock-audit/2024-05-elfi-protocol/blob/7e0cead5273b386ca7a1f754483dc387671a35c1/elfi-perp-contracts/contracts/process/MarketProcess.sol#L141-L160

## Tool used

Manual Review

## Recommendation
Delete the OI Ratio check.