---
name: payment-processor-protocol
description: Payment Processor V3 background knowledge -- the NFT marketplace protocol for buying, selling, and trading ERC-721 and ERC-1155 tokens. Covers architecture, data structures, fee model, events. Use this skill whenever the user asks about NFT trading, NFT marketplaces, buying or selling NFTs, listing NFTs for sale, making offers on NFTs, sweeping collections, royalty enforcement, EIP-712 order signing, creator payment settings, or any code referencing Payment Processor at 0x9a1D00000000fC540e2000560054812452eB5366. Payment Processor is the marketplace layer for NFTs -- LBAMM and CLOB are for ERC-20C fungible tokens, not NFTs.
user-invocable: false
---

# Payment Processor V3 Protocol Knowledge Base

You have deep knowledge of Payment Processor V3. Use this skill's reference files whenever the user's question involves NFT trading, order signing, creator payment settings, royalty enforcement, or any of the related Payment Processor skills.

## Architecture Overview

Payment Processor V3 is a **peer-to-peer NFT trading protocol** with no escrow. It is purpose-built for creators with mandatory royalty enforcement via EIP-2981 and backfill royalties for legacy collections. The protocol supports both ERC721 and ERC1155 tokens, and is backwards compatible with non-Creator Standard tokens.

## Module Delegation Pattern

PaymentProcessor is the main entry point contract and delegates execution via assembly `delegatecall` to four specialized modules. All modules share Diamond storage at slot `0x9A1D`.

| Module | Responsibility |
|---|---|
| ModuleBuyListings | Executes listing purchases (single, bulk, sweep) |
| ModuleAcceptOffers | Executes offer acceptance (item, collection, token set) |
| ModuleSweeps | Executes collection sweep operations |
| ModuleOnChainCancellation | Handles nonce revocation and order cancellation |

All trade functions accept a single `bytes calldata data` parameter. This calldata is pre-encoded using the **PaymentProcessorEncoder** contract (called off-chain or on-chain).

## EIP-712 Domain

The protocol uses EIP-712 structured signing with the following domain parameters:

- **name:** `"PaymentProcessor"`
- **version:** `"3.0.0"`
- **chainId:** current chain ID
- **verifyingContract:** PaymentProcessor contract address

## Repository

GitHub: `https://github.com/limitbreakinc/payment-processor`

### Project Setup

1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **Payment Processor**: `lib/payment-processor/` must exist. If not: `forge install limitbreakinc/payment-processor`
3. **Remappings**: `remappings.txt` needs `@limitbreak/payment-processor/=lib/payment-processor/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: For local development, use `apptoken-dev` to bootstrap an Anvil node with all Apptoken protocols at their deterministic addresses -- see `apptoken-general/references/project-setup.md`.

## Contract Addresses

All contracts are deployed at deterministic addresses across supported EVM chains:

| Contract | Address |
|---|---|
| Payment Processor V3 | `0x9a1D00000000fC540e2000560054812452eB5366` |
| Encoder | `0x9A1D00C3a699f491037745393a0592AC6b62421D` |
| Configuration | `0x9A1D00773287950891B8A48Be6f21e951EFF91b3` |
| Module: On Chain Cancellation | `0x9A1D005Da1E3daBCE14bc9734DEe692A8978c71C` |
| Module: Accept Offers | `0x9A1D00E769A108df1cbC3bFfcF867B64ba2E9eFf` |
| Module: Buy Listings | `0x9A1D00A68523b8268414E4406268c32EC83323A9` |
| Module: Sweeps | `0x9A1D008994E8f69C66d99B743BDFc6990a7801aB` |
| Collection Settings Registry | `0x9A1D001A842c5e6C74b33F2aeEdEc07F0Cb20BC4` |

## Reference Files

- `references/data-structures.md` -- Data structures, enums, and module delegation pattern
- `references/fee-model.md` -- Fee model, royalties, and events

## Related Skills

- **payment-processor-creator** -- Generate creator configuration scripts (payment settings, pricing, royalties)
- **payment-processor-exchange** -- Generate exchange integration code (order signing, filling, PermitC)
- **permitc-protocol** -- PermitC approval system used for time-bound NFT listings
- **apptoken-general** -- Ecosystem overview, ERC-721C/ERC-1155C token standards
