---
name: payment-processor-creator
description: Generate Payment Processor V3 creator configuration for NFT marketplace settings -- payment methods, pricing constraints, royalties, trusted channels. Use when setting up an ERC-721 or ERC-1155 collection for trading.
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of creator payment/trading requirements]"
---

# Payment Processor Creator Configuration Generator

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **Payment Processor** (optional): `forge install limitbreakinc/payment-processor` for direct interface imports. Otherwise, scripts use inline interface definitions with the deployed contract addresses.
3. **Local testing**: For local development, use `apptoken-dev` to bootstrap an Anvil node with all Apptoken protocols at their deterministic addresses -- see `apptoken-general/references/project-setup.md`.

## Instructions

1. Parse the creator's requirements from `$ARGUMENTS`.
2. **Resolve ambiguities before generating** -- if $ARGUMENTS mentions a collection without its address, or the desired payment configuration isn't clear (which payment methods, pricing constraints, royalty settings), ask in a single message. If clear, proceed.
3. Determine which operations are needed:
   - Payment settings (default whitelist, custom whitelist, allow any, pricing constraints, pause)
   - Pricing constraints (collection-level floor/ceiling, token-level bounds)
   - Royalty configuration (backfill rate/receiver, backfill-as-source flag)
   - Royalty bounties (bounty numerator, exclusive receiver)
   - Trusted channels (add/remove, block untrusted channels flag)
   - Payment method whitelists (create, add/remove methods, reassign ownership)
   - Permit processors (add/remove trusted processors)
4. Generate a Foundry script or transaction sequence that configures the collection.
5. All settings are managed through the **Collection Settings Registry** at `0x9A1D001A842c5e6C74b33F2aeEdEc07F0Cb20BC4`.
6. Settings are **lazily loaded** into Payment Processor on the first trade, then cached locally in the PP contract. When settings are updated in the registry, pass the PP address in `paymentProcessorsToSync` to push the update immediately.
7. **Who can call:** The token contract itself, its `owner()`, or a holder of `DEFAULT_ADMIN_ROLE`.

## Reference Files

- `references/payment-settings.md` -- Full Collection Settings Registry API reference
- `references/creator-patterns.md` -- Common configuration patterns with Foundry scripts
- Protocol knowledge: `payment-processor-protocol`

## Common Mistakes

- `paymentSettings=2` (CustomPaymentMethodWhitelist) ignores `constrainedPricingPaymentMethod`, `collectionMinimumFloorPrice`, and `collectionMaximumCeilingPrice` -- those fields only work with `paymentSettings=3` or `4`
- Forgetting to pass the PP address in `paymentProcessorsToSync` when updating registry settings -- changes won't take effect until the next lazy sync
- Not whitelisting Payment Processor as an operator in the Transfer Validator -- NFTs become untradeable

## When to Use This vs Other Skills

- **This skill (payment-processor-creator)**: Configure NFT sale rules (pricing, royalties, payment methods). For ERC-721C and ERC-1155C tokens.
- **transfer-validator-config**: Configure which operators can move the NFTs. Do this *first* -- Payment Processor must be whitelisted.
- **payment-processor-exchange**: Generate the exchange-side code (order signing, filling). Use *after* creator settings are configured.

## Related Skills

- **payment-processor-exchange** -- Generate exchange integration code (order signing, filling, PermitC)
- **apptoken-general** -- ERC-721C/ERC-1155C token standards and ecosystem overview
