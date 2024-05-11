# Finding 1

## Lines of code

https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Oracle.sol#L337-L339


## Vulnerability details

### Impact
Liquidations may not be possible at a time when the protocol needs them most. As a result, the value of user's asset may fall below their debts, turning off any liquidation incentive and pushing the protocol into insolvency.

### Proof of Concept
Chainlink has taken oracles offline in extreme cases. For example, during the UST collapse, Chainlink paused the UST/ETH price oracle, to ensure that it wasn't providing inaccurate data to protocols.

In such a situation (or one in which the token's value falls to zero), all liquidations for users holding the frozen asset would revert. since the price verifies from TWAP would not match This is because any call to liquidate() calls isLoanHealthy(), which calls getValue(), which calls the oracle to get the values of all the position's tokens (underlying, debt, and collateral).

Depending on the specifics, one of the following checks would cause the revert:

the call to Chainlink's registry.latestRoundData would fail
```
(, int256 answer,, uint256 updatedAt,) = feedConfig.feed.latestRoundData();
      if (updatedAt + feedConfig.maxFeedAge < block.timestamp || answer < 0) {
          revert ChainlinkPriceError();
      }
```
If the oracle price lookup reverts, liquidations will be frozen, and the user will be immune to liquidations. Although there are not much vulnerabily since presence of another oracle , which could notify the lenders of this liquidation possibility

### Tools Used
Manual Review, Chainlink docs

### Recommended Mitigation Steps
Ensure there is a safeguard in place to protect against this possibility.
Use a try-catch block for dealing with these.


## Assessed type

Oracle



# Finding 2

## Lines of code

https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Oracle.sol#L363


## Vulnerability details

### Impact
Usage of `slot0` is extremely easy to manipulate

### Proof of Concept
Protocol is using `slot0` to calculate tokenPrice in their codebase,
`slot0` is the most recent data point and is therefore extremely easy to manipulate.
```
    uint160 sqrtPriceX96;
       // if twap seconds set to 0 just use pool price
       if (twapSeconds == 0) {
           (sqrtPriceX96,,,,,,) = pool.slot0();
```

### Tools Used
Manual Review, UNI Book

### Recommended Mitigation Steps
completely use a TWAP of higher time interval instead of `slot0.` since smaller time interval can be prone to flashLoan attacks , whereas higher interval is only prone to inaccurate prices , but since the protocol is also using ChainLink so it might not be much problem.


### Assessed type

Uniswap
