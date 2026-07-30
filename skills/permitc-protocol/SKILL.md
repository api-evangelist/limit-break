---
name: permitc-protocol
description: PermitC protocol knowledge -- time-bound token approvals, signature-based transfers, order fills, nonce management, and EIP-712 signing. Use this skill whenever the user asks about PermitC, permit transfers, time-bound approvals, gasless token transfers, signature-based token approvals, permit nonces, lockdown, or any Solidity code that imports from @permitC or references PermitC/IPermitC.
user-invocable: false
---

# PermitC Protocol Knowledge Base

PermitC is Limit Break's advanced approval system for ERC-20, ERC-721, and ERC-1155 tokens. It extends the Uniswap Permit2 pattern with multi-token-standard support, time-bound approvals, order-based partial fills, and additional data validation. Multiple protocols in the Apptoken ecosystem use PermitC as their approval layer.

## Core Concepts

### Why PermitC Exists

Traditional token approvals (`approve` / `setApprovalForAll`) are permanent and broad. PermitC replaces them with:

1. **Time-bound approvals** -- every approval has an expiration timestamp
2. **Granular scoping** -- approvals target a specific token, ID, amount, and operator
3. **Signature-based transfers** -- single-use permits that don't require an on-chain approval transaction
4. **One-click revoke** -- `lockdown()` invalidates all outstanding approvals instantly
5. **Order-based fills** -- reusable signatures for partial fills (ERC-1155, ERC-20 limit orders)

### Three Transfer Modes

| Mode | Use Case | Nonce Type |
|---|---|---|
| **Stored Approvals** | Multi-use, on-chain `approve()` call | N/A (amount-based) |
| **Permit Transfers** | Single-use, signed off-chain | Unordered nonce (bitmap) |
| **Order Fills** | Reusable partial fills | Order ID + salt |

### Token Types

PermitC uses numeric token type constants:

- `TOKEN_TYPE_ERC721 = 721`
- `TOKEN_TYPE_ERC1155 = 1155`
- `TOKEN_TYPE_ERC20 = 20`

For ERC-20: always set `id = 0`. For ERC-721: always set `amount = 1`.

## Transfer Validator IS PermitC

A critical architectural detail: the `CreatorTokenTransferValidator` contract **inherits from PermitC**. The Transfer Validator deployed at `0x721C008fdff27BF06E7E123956E2Fe03B63342e3` is itself a PermitC instance. This means:

1. **Every Creator Token Standard token already has access to PermitC** through its Transfer Validator
2. **Auto-approval eliminates the approval step entirely**: Creator token contracts (ERC-20C, ERC-721C, ERC-1155C) inherit `AutomaticValidatorTransferApproval`, which lets the contract owner call `setAutomaticApprovalOfTransfersFromValidator(true)`. When enabled, the token's `isApprovedForAll` (ERC-721C/ERC-1155C) or `allowance` (ERC-20C) automatically returns approved/max for the Transfer Validator address.
3. **Zero-approval token launches**: A token with auto-approval enabled can immediately participate in protocols (Payment Processor, LBAMM, TokenMaster) using only signed PermitC orders -- no `approve()` or `setApprovalForAll()` transactions needed from users.

The PermitC address for signing is the **Transfer Validator address** (`0x721C008fdff27BF06E7E123956E2Fe03B63342e3`), not the standalone PermitC deployment.

## Cross-Protocol Usage

PermitC is used across the Apptoken ecosystem in protocol-specific ways:

| Protocol | PermitC Usage | Details |
|---|---|---|
| **Transfer Validator** | PermitC inheritance | The validator IS a PermitC instance; tokens with auto-approval get permit functionality with zero user approvals |
| **Payment Processor V3** | NFT listing approvals | Sellers sign PermitC permits instead of `setApprovalForAll`; `SaleApproval` embedded as additional data |
| **LBAMM** | Gasless swap settlement | `PermitTransferHandler` uses PermitC for gasless maker-taker swaps; cosigner system for MEV protection |
| **TokenMaster** | Advanced buy orders | `PermitTransferFromWithAdditionalData` wrapping `AdvancedBuyOrder` for gasless token purchases |

For protocol-specific PermitC integration details, see:
- `payment-processor-exchange/references/permitc-integration.md` -- NFT listings with PermitC
- `lbamm-permit-handler/references/permit-handler-reference.md` -- Gasless AMM swaps via PermitTransferHandler
- `tokenmaster-integrator/references/advanced-orders.md` -- PermitC buy orders

## Reference Files

- `references/permitc-core.md` -- Full interface, data types, EIP-712 typehashes, nonce management, approval flows, signing patterns, common errors

## Repository

GitHub: `https://github.com/limitbreakinc/PermitC`

### Project Setup

1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **PermitC**: `lib/PermitC/` must exist. If not: `forge install limitbreakinc/PermitC`
3. **Remappings**: `remappings.txt` needs `@limitbreak/permit-c/=lib/PermitC/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: For local development, use `apptoken-dev` to bootstrap an Anvil node with all Apptoken protocols at their deterministic addresses -- see `apptoken-general/references/project-setup.md`.

## Key Addresses

| Contract | Address | Notes |
|---|---|---|
| Transfer Validator V5 (IS PermitC) | `0x721C008fdff27BF06E7E123956E2Fe03B63342e3` | Use this as the PermitC address for Creator Token Standard tokens |
| Standalone PermitC | `0x000000fCE53B6fC312838A002362a9336F7ce39B` | Standalone deployment for non-Creator-Standard tokens |

## Related Skills

- **transfer-validator-protocol** -- Transfer Validator details (inherits PermitC, rulesets, auto-approval)
- **payment-processor-exchange** -- Generate exchange integration code using PermitC for NFT listings
- **lbamm-permit-handler** -- Generate PermitTransferHandler integration code (PermitC-based gasless swaps)
- **tokenmaster-integrator** -- Generate transaction integration code including PermitC buy orders
- **apptoken-general** -- Ecosystem overview and cross-protocol integration
