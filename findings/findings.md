Green Rabbit

 Green Rabbit

1. Lack of Input validation

https://github.com/syed-ghufran-hassan/Ethereal/blob/main/src/ethereal.sol#L71
https://github.com/syed-ghufran-hassan/Ethereal/blob/main/src/ethereal.sol#L93
https://github.com/syed-ghufran-hassan/Ethereal/blob/main/src/ethereal.sol#L105
https://github.com/syed-ghufran-hassan/Ethereal/blob/main/src/ethereal.sol#L120
https://github.com/syed-ghufran-hassan/Ethereal/blob/main/src/ethereal.sol#L154

Summary:

 The contract does not validate inputs such as _name, _baseURI, _denomination, and _redeemFee. This could lead to unexpected behavior or vulnerabilities if invalid inputs are provided.

Recommendation:

Add input validation checks to ensure that all inputs are within expected ranges and formats.

2. Reentrancy vulnerability in redeem functions

https://github.com/syed-ghufran-hassan/Ethereal/blob/main/src/ethereal.sol#L186
https://github.com/syed-ghufran-hassan/Ethereal/blob/main/src/ethereal.sol#L173

The _redeemEth and _redeemWstEth function transfers Ether to the caller before updating the state (metadata[_tokenId].balance = 0). This could allow a malicious contract to re-enter the function and exploit the contract.

Recommendation:

Please apply CEI pattern

```solidity
function _redeemEth(uint256 _tokenId) internal {
    // Checks
    require(ownerOf(_tokenId) == msg.sender, "Caller is not the owner");
    require(metadata[_tokenId].balance > 0, "No balance to redeem");

    // Effects
    uint256 redeemFee = (metadata[_tokenId].balance * gems[metadata[_tokenId].gem].redeemFee) / 1e4;
    uint256 amount = metadata[_tokenId].balance - redeemFee;
    fees += redeemFee;
    metadata[_tokenId].balance = 0; // Update state before interaction
    circulatingGems--;
    _burn(_tokenId); // Burn the token after updating state

    // Interactions
    (bool success,) = msg.sender.call{value: amount}("");
    require(success, "ETH transfer failed");

    // Emit event
    emit GemRedeemed(_tokenId, msg.sender, amount);
}
```

```solidity
function _redeemWstEth(uint256 _tokenId) internal {
    // Checks
    require(ownerOf(_tokenId) == msg.sender, "Caller is not the owner");
    require(metadata[_tokenId].balance > 0, "No balance to redeem");

    // Effects
    uint256 redeemFee = metadata[_tokenId].balance * gems[metadata[_tokenId].gem].redeemFee / 1e4;
    uint256 amount = metadata[_tokenId].balance - redeemFee;
    fees += redeemFee;
    metadata[_tokenId].balance = 0; // Update state before interaction
    circulatingGems--;
    _burn(_tokenId); // Burn the token after updating state

    // Interactions
    bool transferSuccess = IwstETH(wstETH).transfer(msg.sender, amount);
    require(transferSuccess, "wstETH transfer failed");

    // Emit event
    emit GemRedeemed(_tokenId, msg.sender, amount);
}
```


