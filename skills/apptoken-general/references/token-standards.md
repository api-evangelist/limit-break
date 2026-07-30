# Creator Token Standards

## Overview

Creator Token Standards extend standard ERC token contracts with transfer validation hooks. When a token transfer occurs, the `_beforeTokenTransfer` hook calls into the Transfer Validator contract, which checks the transfer against the creator's configured security policy. This gives creators programmable, upgradeable control over how their tokens move -- without redeploying the token contract.

## Installation

```
forge install limitbreakinc/creator-token-standards
```

Add to `remappings.txt`: `@limitbreak/creator-token-standards/=lib/creator-token-standards/`

## ERC-721C (Non-Fungible Tokens)

Standard ERC-721C inherits from OpenZeppelin's ERC-721:

```solidity
import "@limitbreak/creator-token-standards/src/access/OwnableBasic.sol";
import "@limitbreak/creator-token-standards/src/erc721c/ERC721C.sol";

contract MyNFT is OwnableBasic, ERC721C {
    constructor() ERC721OpenZeppelin("MyNFT", "MNFT") {}
}
```

ERC-721A-C variant inherits from Azuki's ERC-721A for gas-efficient batch minting:

```solidity
import "@limitbreak/creator-token-standards/src/access/OwnableBasic.sol";
import "@limitbreak/creator-token-standards/src/erc721c/ERC721AC.sol";

contract MyNFT is OwnableBasic, ERC721AC {
    constructor() ERC721AC("MyNFT", "MNFT") {}
}
```

## ERC-1155C (Multi-Token)

```solidity
import "@limitbreak/creator-token-standards/src/access/OwnableBasic.sol";
import "@limitbreak/creator-token-standards/src/erc1155c/ERC1155C.sol";

contract MyMultiToken is OwnableBasic, ERC1155C {
    constructor() ERC1155OpenZeppelin("https://api.example.com/metadata/{id}") {}
}
```

## ERC-20C (Fungible Tokens)

```solidity
import "@limitbreak/creator-token-standards/src/access/OwnableBasic.sol";
import "@limitbreak/creator-token-standards/src/erc20c/ERC20C.sol";

contract MyToken is OwnableBasic, ERC20C {
    constructor() ERC20OpenZeppelin("MyToken", "MTK", 18) {}
}
```

**Important:** All C-standard contracts are `abstract` because they inherit `OwnablePermissions` which requires implementing `_requireCallerIsContractOwner()`. You **must** also inherit from an ownership implementation like `OwnableBasic` (or `OwnableInitializable`, `OwnableAccessControl`, etc.) to make the contract deployable.

## CRITICAL: Overriding Transfer Hooks

When overriding transfer hooks, you **MUST** call the `super` version to preserve Creator Token Standard validation. Failing to do so will silently disable all transfer validation. The hook signature differs per standard:

**ERC721C** (OpenZeppelin ERC721):
```solidity
function _beforeTokenTransfer(address from, address to, uint256 firstTokenId, uint256 batchSize)
    internal virtual override {
    super._beforeTokenTransfer(from, to, firstTokenId, batchSize);
}
```

**ERC721AC** (ERC721A variant -- note the plural function name):
```solidity
function _beforeTokenTransfers(address from, address to, uint256 startTokenId, uint256 quantity)
    internal virtual override {
    super._beforeTokenTransfers(from, to, startTokenId, quantity);
}
```

**ERC1155C** (includes operator parameter):
```solidity
function _beforeTokenTransfer(address operator, address from, address to,
    uint256[] memory ids, uint256[] memory amounts, bytes memory data)
    internal virtual override {
    super._beforeTokenTransfer(operator, from, to, ids, amounts, data);
}
```

**ERC20C**:
```solidity
function _beforeTokenTransfer(address from, address to, uint256 amount)
    internal virtual override {
    super._beforeTokenTransfer(from, to, amount);
}
```

## Transfer Validator Setup

### New Tokens

New tokens deployed using Creator Token Standards automatically connect to the **default Transfer Validator**. The default validator address is set in the base contract and applies a baseline security policy out of the box. No additional setup is required for basic transfer validation.

### Existing Collections

For existing collections that need to adopt or upgrade to Transfer Validator V5, the contract owner calls:

```solidity
// Set the Transfer Validator to V5
myToken.setTransferValidator(0x721C008fdff27BF06E7E123956E2Fe03B63342e3);
```

**Transfer Validator V5 address**: `0x721C008fdff27BF06E7E123956E2Fe03B63342e3`

This is a one-time call by the token contract owner. Once set, the validator will be consulted for every transfer.

## EOA Registry

**Address**: `0xE0A0004Dfa318fc38298aE81a666710eaDCEba5C`

The EOA Registry allows externally owned accounts (EOAs) to prove they are not smart contracts. This is used by Transfer Validator rulesets that restrict transfers to verified EOAs.

Verification process:
1. User signs the message `"EOA"` using `personal_sign`
2. Call `verifySignatureVRS(v, r, s)` on the EOA Registry contract
3. The account is permanently registered as a verified EOA

## Auto-Approval of Transfer Validator

All C-standard tokens inherit `AutomaticValidatorTransferApproval`, which gives the contract owner a one-call opt-in to automatically approve the Transfer Validator as an operator for all holders:

```solidity
// Contract owner enables auto-approval (one-time call)
myToken.setAutomaticApprovalOfTransfersFromValidator(true);
```

When enabled:
- **ERC-721C / ERC-1155C**: `isApprovedForAll(owner, transferValidator)` returns `true` for all holders
- **ERC-20C**: `allowance(owner, transferValidator)` returns `type(uint256).max` for all holders

Because the Transfer Validator inherits from PermitC, enabling auto-approval means users can interact with protocols using only signed PermitC orders -- no `approve()` or `setApprovalForAll()` transactions needed. This creates a zero-approval token launch experience. See `permitc-protocol` for details.

## Key Integration Points

- All C-standard tokens expose `getTransferValidator()` and `setTransferValidator(address)` for validator management
- The `transferValidator` is consulted on every transfer via the internal hook
- Creator Token Standards are composable with other OpenZeppelin/Solady extensions (AccessControl, Ownable, Pausable, etc.) as long as `super` calls are preserved in hook overrides
- Token contracts themselves hold no security policy state -- all policy lives in the Transfer Validator, making it upgradeable without redeploying tokens
