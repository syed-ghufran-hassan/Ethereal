Green Rabbit

#  Owner will change wstETH contract address without emitting event

## Summary

The `setWstEth()` function allows the owner to change the wstETH contract address but does not emit an event, making it impossible for off-chain monitoring systems to detect this critical configuration change.

## Severity

Low

## Root Cause

The function updates the wstETH state variable without emitting an event .

 ## Attack Path

- Owner calls setWstEth(newAddress) to update to a new wstETH contract
- No event is emitted
- Off-chain monitoring systems continue using the old address
- Users mint NFTs expecting them to be backed by the original wstETH contract
- NFTs are actually backed by a different contract (potentially malicious)
- No one detects the change until funds are lost

## Impact

The users suffer a loss of all wstETH collateral if the address is changed to a malicious contract. Off-chain monitoring systems cannot track the change.

## Fix

Add event SetWstEth(address indexed oldAddress, address indexed newAddress); and emit it in the function.

# Protocol will burn NFT before confirming ETH transfer

## Summary

The _redeemEth() function burns the NFT and sets the balance to zero before confirming the ETH transfer succeeds. While the transaction reverts on failure, this violates the CEI pattern and wastes gas.

## Severity

Medium

## Root Cause

The function calls _burn() at line 175 and sets metadata[_tokenId].balance = 0 at line 180 before the ETH transfer at line 181 .

## Internal Pre-conditions

Internal Pre-conditions

- User needs to call redeem() on an ETH-backed NFT
- User's address needs to reject ETH or contract needs to have insufficient balance

## External Pre-conditions

None

## Attack Path

- User calls redeem(tokenId) with NFT worth 100 ETH
- NFT transferred to contract and burned (line 174-175)
- Balance set to zero (line 180)
- ETH transfer fails (line 181) - user is contract that rejects ETH
- Transaction reverts (line 182)
- All state changes revert, including NFT burn
- User wastes gas on failed transaction

## Impact

The user suffers gas loss on failed redemption attempts. The protocol wastes gas on state changes that revert.

## Fix

Move state updates (burn, balance set to zero) after the ETH transfer confirmation, following proper CEI pattern.

# Protocol will track fees as ETH value but hold wstETH tokens

## Summary

When users redeem wstETH-backed NFTs, the fee is calculated in wstETH tokens but added to the fees variable which is treated as ETH value. The withdrawFees() function attempts to withdraw ETH, but the actual fees are held in wstETH tokens.

## Severity

High

## Root Cause

_redeemWstEth() adds the fee (in wstETH tokens) to the fees variable . In src/ethereal.sol:259-263, withdrawFees() attempts to transfer ETH using the fees variable .

## Internal Pre-conditions

- Users need to redeem wstETH-backed NFTs
- Owner needs to call withdrawFees()

## External Pre-conditions

- wstETH to ETH exchange rate needs to change between minting and withdrawal

## Attack Path 

- User mints wstETH-backed NFT when 1 wstETH = 1 ETH
- User redeems NFT, paying 1 wstETH fee
- fees variable increases by 1 (treated as ETH value)
- wstETH appreciates to 1 wstETH = 2 ETH
- Owner calls withdrawFees() to withdraw 1 ETH
- Contract has 1 wstETH (worth 2 ETH) but can only withdraw 1 ETH
- Protocol loses 1 ETH in value

## Impact

The protocol suffers a loss of value due to exchange rate differences between ETH and wstETH. The owner cannot withdraw the full value of accumulated fees.

## Fix

Track fees separately by collateral type, or convert wstETH fees to ETH immediately upon collection.
