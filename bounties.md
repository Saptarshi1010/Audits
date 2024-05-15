
# Bug 1 Silkywap
## Title
Failed transfer may be overlooked due to lack of contract existence check

### Bug Description
Because the pool fails to check that a contract exists, the pool may assume that failed transactions involving destructed tokens are successful. TransferHelper safeTransfer performs a transfer with a low-level call without confirming the contract's existance.

This function is utilized in a few different places in the contract. According to the Solidity docs, "The low-level functions call, delegatecall and staticcall return true as their first return value if the account called is non-existent, as part of the design of the EVM. Account existence must be checked prior to calling if needed".

As a result, if the tokens have not yet been deployed or have been destroyed, safeTransfer will return success even though no transfer was executed. If the token has not yet been deployed, no liquidity can be added. However, if the token has been destroyed, the pool will act as if the assets were sent even though they were not In particular, it is possible that the address to is a deleted contract (perhaps a security flaw was found and selfdestruct was called so that users know to use an updated smart contract), but safeTransferETH will not revert.

### Impact
The main inpact is that victim users will loose their tokens, due to malicious deployed contract.


### Recommendation
Check for contract existence on low-level calls, so that failures are not missed. check address token is not address(0). Imple It would be also better to review the solidity docs for implementation and mitigation for this.


### Proof Of Concept
addLiquidity(), removeLiquidity(), swap() functions has under the hood used below private/internal functions which are given as below,

```
  function safeTransfer(
        address token,
        address to,
        uint256 value
    ) internal {
        // bytes4(keccak256(bytes('transfer(address,uint256)')));
        (bool success, bytes memory data) = token.call(abi.encodeWithSelector(0xa9059cbb, to, value));
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            'TransferHelper::safeTransfer: transfer failed'
        );
    }
```
```
 function safeTransferETH(address to, uint256 value) internal {
        (bool success, ) = to.call{value: value}(new bytes(0));
        require(success, 'TransferHelper::safeTransferETH: ETH transfer failed');
    }
```
The attack ways is simplified and described here, And the description is described earlier .

The pool contains tokens A and B. Token A has a bug, and the contract is destroyed. Bob is not aware of the issue and swaps 1,000 B tokens for A tokens. Bob successfully transfers 1,000 B tokens to the pool but does not receive any A tokens in return. As a result, Bob loses 1,000 B token

This vulnerability was also present in UniswapV3 which was presented to them in an audit, which was also coined as high and it was also the reason for an $80M exploit. in quibit finance [ safe TransferFrom doesn't revert on null address ] [I think certik has a writeup on this issue]







# Bug 2 SilkySwap
## Title 
RemoveLiquidity function CAN BE BROKEN WHEN USING TOKENS THAT DO NOT FOLLOW THE ERC2612 STANDARD

## Bug Description
Certain tokens may not adhere to the IERC20Permit standard. For example, the DAI Stablecoin utilizes a permit() function that deviates from the reference implementation. This lack of verification may lead to inconsistencies and unexpected behavior when interacting with non-conforming tokens.
More importantly the permit function can be frontrun , by copying the permit

## Impact
There are actually two types vulnerability that arises from these

Dai token implementation with permit
Front running leading to DOS


## Recommendation
Add proper verification to the permit() function call. We recommend

1.. For the special case of DAI token, allow a different implementation of the permit function which allows a nonce variable.

adding a check that msg.sender is this owner.

## References
DAI exceptional
https://solodit.xyz/issues/m-08-permit-doesnt-work-with-dai-code4rena-pooltogether-pooltogether-git

Frontrunning
https://solodit.xyz/issues/a-swap-with-permit-can-be-blocked-if-a-frontrunner-swaps-using-copied-permit-mixbytes-none-barter-dao-markdown

## Proof Of Concept
Since i didn't see any POC requirements. So, ATM i am providing links for more information about this[see in refernece section i have given some links regarding those issues], and since this is a very widely known issue .

The problem lies on this functions of UNIV2Router02:removeLiquidityWithPermit , UNIV2Router02:removeLiquidityETHWithPermit, UNIV20:removeLiquidityETHWithPermitSupportingFeeOnTransferTokens

The line which is the main problem
`
IUniswapV2Pair(pair).permit(msg.sender, address(this), value, deadline, v, r, s);
`







# Bug 3 Ammet Finance
## Title
Not able to initialize bond

### Bug Description
Issuer Address is separately set in the constructor while deploying, while the owner of the contract is set as the msg.sender The problem here becomes if somehow while deploying the issuerAddress != Owner's address i.e issuerAddress != msg.sender in the Vault.initializeBond(), Then the function reverts and the bond cannot be initialized. the problem becomes if the function is called by issuer whose address is not as same as msg.sender. https://github.com/Amet-Finance/contracts/blob/92630c9092d949f61ead1dcaa67ad6917eb1b603/contracts/fixed-flex/Vault.sol#L45

There is also a issue with the bondAddress being initialized to 0 address. Vault.initializeBond()

https://github.com/Amet-Finance/contracts/blob/92630c9092d949f61ead1dcaa67ad6917eb1b603/contracts/fixed-flex/Vault.sol#L52

### Impact
The main impact of the code is high that the bond cannot be initialized which breaks the main working of the project, that also due to input validation. The reason of having the medium severity is the occurance of this is very low .


### Recommendation
Mention in the document about the deployment, and deploy carefuly , a best way would be to have the issuerAddress as also msg.sender since it is already set as immutable. and ig the project working idea is the owner being the issuer. _issuerContract = IIssuer(msg.sender) And also apply 0 address checks in the constructor for issuerAddress and also for bond address

### Proof Of Concept
I am attaching a simple custom made POC, I checked it through remix [Before approaching the POC, i did not write much of custom error messages, pardon me for that]

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;


import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "contracts/helper bounty.sol";

contract A is Ownable{
    using SafeERC20 for IERC20;

    
IERC20 public token;

 struct BondFeeDetails {
        uint8 purchaseRate; // Fee rate for bond purchases
        uint8 earlyRedemptionRate; // Fee rate for early bond redemption
        uint8 referrerRewardRate; // Reward rate for referrers
        bool isInitiated; // Tells if the bond was created or not
    }

mapping (address => BondFeeDetails ) public bonds;

address public immutable issuer;
BondFeeDetails public initialBondFeeDetails ;

constructor ( address initialIssuer,uint8 purchaseRate, uint8 earlyRedemptionRate,uint8 referrerRewardRate) Ownable(msg.sender){
    initialIssuer = issuer;
    initialBondFeeDetails = BondFeeDetails(purchaseRate, earlyRedemptionRate, referrerRewardRate, true);
}

function see2() public  {
    require(issuer == msg.sender,"revert, this is not owner");
    //require(owner == msg.sender,"revert, this is not the owner");
       address bond = address(0);
    bonds[bond]=initialBondFeeDetails;
}
function see3(address owner) public  {
    //require(issuer == msg.sender,"revert, this is not owner");
    require(owner == msg.sender,"revert, this is not the owner");
       address bond = address(0);
    bonds[bond]=initialBondFeeDetails;
}
// function changeIssuer(address Issuer) public {
//     issuer = Issuer;
// }

}
```

Steps of checking it

Set the issuer address a bit different than the Owner

first try to execute see2 function it checks the issuer is the owner or not

then execute the see3 function which checks if the owner is the caller or not , And this will go through and it also doesn;t revert for 0 bond address

