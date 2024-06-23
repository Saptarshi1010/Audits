# Finding -1 Chainlink Price manipulation

## Severity 
High

## Lines of code
https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/Vault.sol#L102


## Vulnerability details

### Impact
Though the vulnerabile part is not directly in scope, but it affects the code in scope. Since due to limited scope i have gathered all the issues in a single report that's why decided to go with high, reasons of which can be seen in next part.
Protocol expected behavior may not work as expected by the protocol , since the value returned by oracle might be 0 if the price returned by chainlink oracle is 0 or maybe just due to insufficient validation leading to stale prices.

One of the main impact is the collatRatio calculated by the project can deviate by a certain percentage which might result in user redeem less of his collateral than deserving

## Proof of Concept
1.Chainlink has taken oracles offline in some cases. In such a scenario if the oracle price that is fetched in *vault.sol* through`oracle.latestRoundData();`
where it checks this `if (block.timestamp > updatedAt + STALE_DATA_TIMEOUT) revert StaleData()` and then returns the price like this ` return answer.toUint256();`
https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/Vault.sol#L91
But the problem is that if chainlink has taken oracles offline then the answer would be 0 , & the OZ safeCast would pass this value
```
 function toUint256(int256 value) internal pure returns (uint256) {
        require(value >= 0, "SafeCast: value must be positive");
        return uint256(value);
    }
```
Thus when a user wants to withdraw or redeem their asset their amount will become 0, due to division by 0, Making withdrawing & redemtion frozen. And if in that time their collateral value goes below the threshold, they can get liquidated easily

2. For stable coins Chainlink might return minAnswer instead of just returning 0

3. In the protocol implementation the roundId is not validated. [Refer to the chainlink docs for this] ,It should not be assumed that *updatedAt* only returns fresh prices

4. ETH/USD price feed for example have a deviation of 0.5% on mainnet, meaning price will only be updated once the price movement exceeds 0.5% in the heartbeat period. If the market price of WETH is lower than oracle price, it is possible to  decrease the amount of collateral and thus possibly leading to arbitrage


### Tools Used
Manual Analysis

### Recommended Mitigation Steps
1. Add proper safeguard for this case, maybe wrapping them up in a try/catch block

2. Also if the price doesn't go below 0 , if it just crashes like Luna , it's better to have a min/max threshold in check for those events

3. `require(answeredInRound >= roundId)` the protocol should check for stale or incorrect prices

4.Consider a minting fee higher than the deviation




# Finding -2 `Abi.encodePAcked` allows hash collision

## Severity- Medium 

## Lines of code
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/PanopticPool.sol#L462
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/PanopticPool.sol#L1643
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/PanopticPool.sol#L1674
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/PanopticPool.sol#L1674


## Vulnerability details

### Impact
[From the solidity documentation](https://docs.soliditylang.org/en/v0.8.17/abi-spec.html?highlight=collisions#non-standard-packed-mode) > If you use keccak256(abi.encodePacked(a, b)) and both a and b are dynamic types, it is easy to craft collisions in the hash value by moving parts of a into b and vice-versa. More specifically, `abi.encodePacked("a", "bc")` == `abi.encodePacked("ab", "c")`.

### Proof of Concept
The problem lies in the following lines :
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/PanopticPool.sol#L462
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/PanopticPool.sol#L1643
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/PanopticPool.sol#L1674
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/PanopticPool.sol#L1674
these dynamic values are user-specified function arguments in functions, meaning anyone can specify the value of these arguments when calling the function.
The abi.encodePacked method is called where there are multiple dynamically-sized types. Trying to pack these dynamically-sized types leads to ambiguity in unpacking. This makes it difficult to impossible to unpack, especially in a safe manner. It is made more difficult, in a case like this, because users have some limited input control over the contents passed via the name and symbol parameters

> There can be many more instances of these for eg. in SFPM.sol many such instances are used.

### Tools Used
Manual Analysis

### Recommended Mitigation Steps
Use `abi.encode` instead of `abi.encodePAcked` if not necessary

### Assessed type
Invalid Validation




# Create methods are suspicious of the reorg attack

## Severity
Medium Risk

## Relevant GitHub Links
https://github.com/Cyfrin/2024-05-Sablier/blob/43d7e752a68bba2a1d73d3d6466c3059079ed0c6/v2-periphery/src/SablierV2MerkleLockupFactory.sol#L36

https://github.com/Cyfrin/2024-05-Sablier/blob/43d7e752a68bba2a1d73d3d6466c3059079ed0c6/v2-periphery/src/SablierV2MerkleLockupFactory.sol#L71

## Summary
Any airdrop amount sent to the contract could potentially be withdrawn by anyone else. All in all, it could lead to the theft of user funds. Since the project is going to get deployed in all evm compatible chains L2's such as Optimistic rollups (Optimism/Arbitrum) are also suspect to reorgs since if someone finds a fraud the blocks will be reverted, even though the user receives a confirmation.

## Vulnerability Details
Imagine that Alice deploys a new SablierV2MerkleLL & SablierV2MerkleLt, and then sends funds to it. Bob sees that the network block reorg happens and calls createMerkleLL & createMerkleLT. Thus, it creates SablierV2MerkleLL/SablierV2MerkleLt with an address to which Alice sends funds. Then Alices’ transactions are executed and Alice transfers funds to Bob’s controlled contract.
Polygon re-org reference: Polygon blocks forked:

## Impact
The airdrop amount will be sent to wrong contract due to block reorg

## Tools Used
Manual Review

## Recommendations
Deploy such contracts via create2 with salt that includes msg.sender.


# Finding 4 Tokens that transfer less than amount

## Severity- Medium

## Lines of code
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/PanopticPool.sol#L326
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/libraries/InteractionHelper.sol#L24
https://github.com/code-423n4/2024-04-panoptic/blob/833312ebd600665b577fbd9c03ffa0daf250ed24/contracts/PanopticFactory.sol#L251


## Vulnerability details

### Impact
The report is not about fee on transfer tokens
And the protocol also specifies that pools will not have huge amount if tokens,
that the tokens amount will not exceed 2^127 - 1. in one pool.

But the protocol has a approval for uint256.max in `InteractionHelper:doapprovals`
Some tokens such as cUSDCv3 contain a special case for amount == type(uint256).max in their transfer functions that results in only the user's balance being transferred. This can be used to shut down several pool operations.

### Proof of Concept
When someone calls `Panopticfactory:deployNewPool` to deploy a new pool which calls `PanopticPool:startPool` and then ultimately calls the helper contract  to deploy the pool with the tokens

So, An attacker can put dust of this token in a wallet, and then call deployNewPool() with type(uint256).max of this token. If the pool has not already been funded, then poolAmount will be at type(uint256).max despite nothing being in the pool. It is now not possible to fund the pool.

So, anyone wanting to interact or trade with one such pool will fail

### Tools Used
Manual Review

### Recommended Mitigation Steps
Explicitly do not support these tokens

### Assessed type
ERC20




# Finding -5 MEV Sandwich Attack

## Severity 
Medium

## Lines of code
https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L184


## Vulnerability details

### Impact
An attacker can perform a sandwich attack on calls to redeemDYAD to make an instantaneous profit from the protocol. This effectively steals funds away from other legitimate users of the protocol.
A malicious user could listen to the mempool for calls to redeemDYAD, at which point they can perform a sandwich attack by calling Deposit -> mint before the redeem transaction and then redeem themselves after the withdrawal transaction. They can accomplish this using a tool like flashbots and make an instantaneous profit due to changes in exchange rates.

### Proof of Concept
The protocol actually has a same block check for this issue , but the issue still remains because the user will still profit in the same manner due to the fluctuation in exchange rates.
If a user wants to redeem their funds they call redeemDYAD , so now a flashbot may just deposit some kerosine and mint some DYAD where they have to pass only this check ..
```
uint newDyadMinted = dyad.mintedDyad(address(this), id) + amount;
if (getNonKeroseneValue(id) < newDyadMinted)     revert NotEnoughExoCollat();
```
which they can pass if they already had some deposited weth as collateral
And now since kerosine price is determined by
`kerosine price = (tvl - dyad.totalsupply) / total kerosine supply`
So the flashbots depositing will make dyad.totalsupply go up and total kerosine supply in the vault also high resulting in decrement of kerosine price
Remember there are also chainlink stale prices for kerosine which can make these things even worse & make the profit for the user much more.

### Tools Used
Manual review

### Recommended Mitigation Steps
Not really sure since if we even implement time based withdraw system it still might not be totally mitigated

### Assessed type
Error




# Finding -6 Both reserves not check while removing liquidity

## Severity Medium
## Lines of code
https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/pools/Pools.sol#L187

## Vulnerability details

### Impact
The Liquidation process happens something like this `CollateralAndLiquidity::liquidator` -> `Pools::removeLiquidity` , `StakingRewards::_decreaseUserShare`

### Proof of Concept
So, During liquidation process, while `removeLiquidity` is called which checks the amount to remove from each reserves, that if the reserves are greater than the Dust amount, but while checking the reserves it doesn't check both the reserves it only checks `reserve0` twice
```
function removeLiquidity( IERC20 tokenA, IERC20 tokenB, uint256 liquidityToRemove, uint256 minReclaimedA, uint256 minReclaimedB, uint256 totalLiquidity ) external nonReentrant returns (uint256 reclaimedA, uint256 reclaimedB)
                {
                require( msg.sender == address(collateralAndLiquidity), "Pools.removeLiquidity is only callable from the CollateralAndLiquidity contract" );
                require( liquidityToRemove > 0, "The amount of liquidityToRemove cannot be zero" );

                (bytes32 poolID, bool flipped) = PoolUtils._poolIDAndFlipped(tokenA, tokenB);

                // Determine how much liquidity is being withdrawn and round down in favor of the protocol
                PoolReserves storage reserves = _poolReserves[poolID];
                reclaimedA = ( reserves.reserve0 * liquidityToRemove ) / totalLiquidity;
                reclaimedB = ( reserves.reserve1 * liquidityToRemove ) / totalLiquidity;

                reserves.reserve0 -= uint128(reclaimedA);
                reserves.reserve1 -= uint128(reclaimedB);

                // Make sure that removing liquidity doesn't drive either of the reserves below DUST.
                // This is to ensure that ratios remain relatively constant even after a maximum withdrawal.
      require((reserves.reserve0 >= PoolUtils.DUST) && (reserves.reserve0 >= PoolUtils.DUST), "Insufficient reserves after liquidity removal");

                // Switch reclaimed amounts back to the order that was specified in the call arguments so they make sense to the caller
                if (flipped)
                        (reclaimedA,reclaimedB) = (reclaimedB,reclaimedA);

                require( (reclaimedA >= minReclaimedA) && (reclaimedB >= minReclaimedB), "Insufficient underlying tokens returned" );

                // Send the reclaimed tokens to the user
                tokenA.safeTransfer( msg.sender, reclaimedA );
                tokenB.safeTransfer( msg.sender, reclaimedB );

                emit LiquidityRemoved(tokenA, tokenB, reclaimedA, reclaimedB, liquidityToRemove);
                }
```


### Tools Used
VS Code

### Recommended Mitigation Steps
Replace this with the original code
require((reserves.reserve0 >= PoolUtils.DUST) && (reserves.reserve1 >= PoolUtils.DUST), "Insufficient reserves after liquidity removal");

### Assessed type
Invalid Validation




# Finding 7 Chainlink oracle

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



# Finding 8 Slot0 manipulation

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




# Finding  9 ERC 20 tokens

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




# Finding 1A Invalid validation

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


# Finding 1b Math

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


# Finding -1C Zero address check

risk = QA (Quality Assurance)

## [L-01] Contract may not be initialized coorectly since no explicitly checking the address initialized 
Lines of code
https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/VaultManagerV2.sol#L49
https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/Vault.kerosine.sol#L26
https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/Vault.kerosine.bounded.sol#L17
https://github.com/code-423n4/2024-04-dyad/blob/4a987e536576139793a1c04690336d06c93fca90/src/core/Vault.kerosine.unbounded.sol#L21


### Recommended Mitigation Steps
Add a 0 address check for each of the instances

