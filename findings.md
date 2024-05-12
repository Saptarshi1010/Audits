# Finding -1

## Severity 
High

## Lines of code
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/SemiFungiblePositionManager.sol#L837


## Vulnerability details

### Impact
AMMs provide their users with an option to limit the execution of their pending actions, such as swaps or adding and removing liquidity. The most common solution is to include a deadline timestamp as a parameter (for example see Uniswap V2 and Uniswap V3). If such an option is not present, users can unknowingly perform bad trades:
I am giving a simple example(not protocol related, but just for simplicity) of not having a deadline check, for just swapping 

Alice wants to swap 100 tokens for 1 ETH and later sell the 1 ETH for 1000 DAI.

The transaction is submitted to the mempool, however, Alice chose a transaction fee that is too low for miners to be interested in including her transaction in a block. The transaction stays pending in the mempool for extended periods, which could be hours, days, weeks, or even longer.

When the average gas fee dropped far enough for Alice's transaction to become interesting again for miners to include it, her swap will be executed. In the meantime, the price of ETH could have drastically changed. She will still get 1 ETH but the DAI value of that output might be significantly lower.

She has unknowingly performed a bad trade due to the pending transaction she forgot about.
An even worse way this issue can be maliciously exploited is through MEV:
The swap transaction is still pending in the mempool. Average fees are still too high for miners to be interested in it. The price of tokens has gone up significantly since the transaction was signed, meaning Alice would receive a lot more ETH when the swap is executed.Although the protocol calculates the slippage off chain , But still  that also means that her maximum slippage value can be outdated and would allow for significant slippage.

A MEV bot detects the pending transaction. Since the outdated maximum slippage value now allows for high slippage, the bot sandwiches Alice, resulting in significant profit for the bot and significant loss for Alice.


### Proof of Concept
```
(int256 swap0, int256 swap1) = _univ3pool.swap(//@audit-issue no deadline check or not swapping with router
               msg.sender,
               zeroForOne,
               swapAmount,
               zeroForOne
                   ? Constants.MIN_V3POOL_SQRT_RATIO + 1
                   : Constants.MAX_V3POOL_SQRT_RATIO - 1,
               data
           );
```
there is no deadline check set for swapping. and the slippage check is done off chain and we don't know the slippage parameters, And there is also no expected minAmount check specified by the protocol.
As said in the impact a simple example can be since the protocol is associated with options trading, the impact could be much bigger because if the tx. doesn't go through at a particular period of time the users might face a loss.

Let's say Alice wants to open a long/short put/call position she submits a tx specifing the strike price,  & other necessary things in the tokenId positionList, but due to her low tx. fee it doesn't go though or a bot might sandwich the position, taking away alice profit from that position & alice makes a loss instead of profit
If the option is also not executed under a certain expiry date of the option,it might be a case of DOS also

### Tools Used
Manual Review

### Recommended Mitigation Steps
We recommend the protocol use deadline check (not block.timestamp, since tx can be sitting in the memepool & executed after a long time) for swapping deadline and swap with Unsiwap Router V3 instead of the pool directly!
Maybe specifing a `minAMountOut` after the swap,


### Assessed type
Uniswap


# Finding 2 

## Severity 
 Medium
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



# Finding 3

## Severity 
 Medium
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




# Finding 4

## Severity 
 Medium
 
 # Lines of code
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/CollateralTracker.sol#L542
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/tokens/ERC20Minimal.sol#L49
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/CollateralTracker.sol#L886


## Vulnerability details

### Impact
1. Some tokens such as BNB revert on if the amount approved to be transferred is 0
2. Some tokens made from OZ will revert if trying to approve the zero address to spend tokens
3. Some Tokens have different ways of signalling success and failure, While some tokens revert upon failure, others consistently return boolean flags( 0x's ZRX Token contract,e.g.: no tokens transferred due to insufficient balance) but the error would not be detected by the Panoptic contracts. to indicate success or failure, and many others have mixed behaviours.

### Proof of Concept

1,2 Panoptic users can withdraw/redeem a adequate amount of tokens if he is given allowance by the token owner to withdraw or redeem it , but if the amount approved it is 0, or it is sent to address 0 the function will revert.

Sellers who want to exercise other buyers but don't have that specific set of tokens can delegate that to other users, if one such token delegateAmount is USDT  or ZRX it is a case of option 3

And also  The function will revert due to solidity 0.8.0, checks if there is a overflow/underflow

### Tools Used
Manual Review

### Recommended Mitigation Steps
Add this check
1.`require(amount > 0, "allow a minimumamount")`

2. ` approvedAddrress != address(0)`

3.
```
IERC20 token = whatever_token;

(bool success, bytes memory returndata) = address(token).call(abi.encodeWithSelector(IERC20.transferFrom.selector, sender, recipient, amount));

// if success == false, without any doubts there was an error and callee reverted
require(success, "Transfer failed!");

// if success == true, we need to check whether we got a return value or not (like in the case of USDT)
if (returndata.length > 0) {
        // we got a return value, it must be a boolean and it should be true
        require(abi.decode(returndata, (bool)), "Transfer failed!");
} else {
        // since we got no return value it can be one of two cases:
        // 1. the transferFrom does not return a boolean and it did succeed
        // 2. the token address is not a contract address therefore call() always return success = true as per EVM design
        // To discriminate between 1 and 2, we need to check if the address actually points to a contract
        require(address(token).code.length > 0, "Not a token address!");
}

```

But there are tokens such as Tether Gold , which returns *false* even if the transfer was successfull

> *Note* Best part will be not to support all tokens




# Finding 5

## Severity
Low/QA

## Lines of code
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/libraries/FeesCalc.sol#L49


## Vulnerability details

### Impact
The developer has subconsciously assumed that `getPortfolioValue` will have an array of `positionIdList` , but any person can pass an empty array and bypass the array.

### Proof of Concept
```
function getPortfolioValue(
      int24 atTick,
      mapping(TokenId tokenId => LeftRightUnsigned balance) storage userBalance,
      TokenId[] calldata positionIdList
  )
```

### Tools Used
Manual Review

### Recommended Mitigation Steps
```
function getPortfolioValue(
      int24 atTick,
      mapping(TokenId tokenId => LeftRightUnsigned balance) storage userBalance,
      TokenId[] calldata positionIdList
  ){
+ require(positionIdList.length > 0,"invalid tokenid");
```


## Assessed type
DoS


# Finding 6

## Severity 
Low/QA

## Lines of code
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/libraries/Math.sol#L391
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/libraries/Math.sol#L131


## Vulnerability details

## Impact
Uniswap math libraries rely on wrapping behaviour for conducting arithmetic operations. Solidity version 0.8.0 introduced checked arithmetic by default where operations that cause an overflow would revert.
Since the code was adapted from Uniswap and written in Solidity version 0.8, these arithmetic operations should be wrapped in an unchecked block

### Proof of Concept
`if (absTick > uint256(int256(Constants.MAX_V3POOL_TICK))) revert Errors.InvalidTick();`
`uint256 twos = (0 - denominator) & denominator;`

### Tools Used
Manual Review

### Recommended Mitigation Steps
Add an unchecked block to the following functions in Math.sol:
```
> • getSqrtRatioAtTick()
> • mulDiv()
```

> [$NOTE:$ There might be other parts in Math.sol or PanopticMath.sol,Due to extensive use of math op. i haven't gone through the, some of the functions ,which i believe might have some]


### Assessed type
Uniswap
