
# 1. *[Qi Dao](https://docs.mai.finance/)* Invalid TokenUri's

## Severity -> Medium

## Lines of code
https://github.com/0xlaozi/qidao/blob/6f65acc401f649a50e15c4f9f197f37d7316419b/contracts/MyVaultV3.sol#L23
https://github.com/0xlaozi/qidao/blob/6f65acc401f649a50e15c4f9f197f37d7316419b/contracts/MyVaultV3.sol#L29

## Impact
Maxi Vault tokens will have invalid tokenURIs. Offchain tools that read the tokenURI view may break or display malformed data, which might result in wrong display/breaking of user deposits.

## Vulnerability
In Solidity, when you encode a variable like `tokenid` (which is a uint256) using abi.encode, it produces a sequence of raw bytes representing the value of that variable.
This means the raw bytes of the 32-byte ABI encoded integer `tokenid` will be interpolated into the token URI, e.g. 0x0000000000000000000000000000000000000000000000000000000000000001 for ID #1.

But These raw bytes, when interpreted as UTF-8 characters, may not always represent printable or valid characters. Some bytes may correspond to control characters (like the "start of heading" character) or symbols that have special meanings in different contexts.

For token ID 1, the resulting bytes might include 0x01, which corresponds to the "start of heading" control character in UTF-8. This character is invisible and not suitable for display in most contexts.
For token ID 42, the resulting bytes might include characters like * (which corresponds to the ASCII value 0x2A), which is a URI-unsafe character and can break the URI structure.



```
   function tokenURI(uint256 tokenId) public view returns (string memory) {
        require(_exists(tokenId));

        IVaultMetaRegistry registry = IVaultMetaRegistry(_meta);
        IVaultMetaProvider provider = IVaultMetaProvider(registry.getMetaProvider(address(this)));

        return bytes(base).length > 0 ? string(abi.encodePacked(base, provider.getTokenURI(address(this), tokenId))) : "";
    }
```




## Recomendation

Convert the `tokenid` to a string before calling abi.encodePacked. 

``` diff
++ import "solmate/utils/LibString.sol";

   function tokenURI(uint256 tokenId) public view returns (string memory) {
        require(_exists(tokenId));

        IVaultMetaRegistry registry = IVaultMetaRegistry(_meta);
        IVaultMetaProvider provider = IVaultMetaProvider(registry.getMetaProvider(address(this)));

--  return bytes(base).length > 0 ? string(abi.encodePacked(base, provider.getTokenURI(address(this), tokenId))) : "";
++  return bytes(base).length > 0 ? string(abi.encodePacked(base, provider.getTokenURI(address(this), LibString.toString(tokenId)))) : "";
    }
```




# 2.*[Maxity Protocol](https://github.com/MaxiProtocol)* Decimals should be returned as `uin8` instead of `uint256`

## Severity -> Low/Informational

## Lines of Code
https://github.com/MaxiProtocol/frontend-amm/blob/87d4257328e5464b8c8f8c9d733a8d0d00040bea/MaxiProtocol.sol#L287
https://github.com/MaxiProtocol/frontend-amm/blob/87d4257328e5464b8c8f8c9d733a8d0d00040bea/MaxiProtocol.sol#L620

## Impact
Undesired functionality , might lead to wrong value of minting, due to incorrect decimal value returned

## Vulnerability
Decimals returns a uint8. Some tokens incorrectly return a uint256. In these cases, ensure the returned value is below 255.

## Recomendation
Return decimals as `uint8` 