---
name: transfer-validator-protocol
description: Transfer Validator V5 background knowledge -- modular rulesets, list management, validation logic, security configurations. Use this skill whenever the user asks about transfer validation, operator whitelists, blacklists, soulbound tokens, OTC transfer rules, account freezing, or any code referencing the Transfer Validator at 0x721C008fdff27BF06E7E123956E2Fe03B63342e3.
user-invocable: false
---

# Transfer Validator V5 Protocol Knowledge Base

## Architecture Overview

Transfer Validator V5 replaces the previous monolithic security levels with **modular rulesets** that are entirely read-only (code purity checked). Collections configure their security policy by selecting a ruleset, setting global options, setting ruleset-specific options, and applying lists.

### PermitC Inheritance

The Transfer Validator **inherits from PermitC** -- it is itself a PermitC instance. This means the Transfer Validator at `0x721C008fdff27BF06E7E123956E2Fe03B63342e3` exposes the full PermitC interface (time-bound approvals, signature-based transfers, order fills, lockdown). Combined with `AutomaticValidatorTransferApproval` on Creator Token Standard tokens, this enables zero-approval token launches where users interact with protocols using only signed permits. See `permitc-protocol` for full PermitC details.

## Rulesets

There are 4 rulesets, each implementing different transfer validation logic:

| Ruleset Name | Ruleset ID | Description |
|---|---|---|
| Whitelist | 0 (default) or 4 | Allows transfers only through whitelisted operators/contracts |
| Vanilla | 1 | No restrictions on transfers |
| Soulbound | 2 | Blocks all transfers (tokens are non-transferable) |
| Blacklist | 3 | Blocks transfers through blacklisted operators/contracts |

- **Ruleset ID 0** = default ruleset. This automatically updates when Limit Break deploys new ruleset versions. Collections using ID 0 always get the latest whitelist ruleset logic.
- **Ruleset ID 4** = explicit whitelist ruleset (same logic as default, but does not auto-update).
- **Ruleset ID 255** = opt-out. When using ID 255, you must provide a `customRuleset` address. This pins the collection to a specific ruleset contract version permanently, opting out of automatic updates.

## Global Options (8-bit bitmask)

Global options apply regardless of which ruleset is selected:

| Bit | Name | Value | Description |
|---|---|---|---|
| GO0 | Disable Authorization Mode | 0x01 | When set, authorizers cannot override validation. Transfers must pass ruleset checks. |
| GO1 | No Authorizer Wildcard Operators | 0x02 | When set, wildcard operator authorizers are ignored. Only specific operator authorizers apply. |
| GO2 | Account Freezing Mode | 0x04 | When set, enables per-collection account freezing. Frozen accounts cannot send or receive tokens. |
| GO3 | Default List Extension Mode | 0x08 | When set, the collection's applied list extends (adds to) the default list rather than replacing it. |

## Whitelist Ruleset Options (16-bit bitmask)

These options only apply when using the Whitelist ruleset (ID 0 or 4):

| Bit | Name | Value | Description |
|---|---|---|---|
| WLO0 | Block All OTC | 0x0001 | Blocks all over-the-counter (direct wallet-to-wallet) transfers. All transfers must go through a whitelisted caller. |
| WLO1 | Allow 7702 Delegate OTC | 0x0002 | Allows OTC transfers when the sender has a whitelisted EIP-7702 delegate. Requires WLO0 to be meaningful. |
| WLO2 | Allow Smart Wallet OTC | 0x0004 | Allows OTC transfers when the sender is a whitelisted smart wallet (by code hash). Requires WLO0 to be meaningful. |
| WLO3 | Block Smart Wallet Receivers | 0x0008 | Blocks transfers to smart contract receivers (non-EOA addresses). |
| WLO4 | Block Unverified EOA Receivers | 0x0010 | Blocks transfers to EOA receivers that are not registered in the EOA Registry. |

## Repository

GitHub: `https://github.com/limitbreakinc/creator-token-transfer-validator`

### Project Setup

1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **Transfer Validator**: `lib/creator-token-transfer-validator/` must exist. If not: `forge install limitbreakinc/creator-token-transfer-validator`
3. **Remappings**: `remappings.txt` needs `@limitbreak/creator-token-transfer-validator/=lib/creator-token-transfer-validator/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: For local development, use `apptoken-dev` to bootstrap an Anvil node with all Apptoken protocols at their deterministic addresses -- see `apptoken-general/references/project-setup.md`.

## Contract Addresses

All contracts are deployed at deterministic addresses across supported chains:

| Contract | Address |
|---|---|
| Transfer Validator V5 | `0x721C008fdff27BF06E7E123956E2Fe03B63342e3` |
| EOA Registry | `0xE0A0004Dfa318fc38298aE81a666710eaDCEba5C` |
| Module: Collection And List Management | `0x721C002d2CAe3522602b93a0c48e11dC573A15E3` |
| Module: Ruleset Bindings | `0x721C0086CC4f335407cC84a38cE7dcb1560476B0` |
| Whitelist Ruleset (v5.0) | Deployed per-chain, referenced by ID 0/4 |
| Vanilla Ruleset (v5.0) | Deployed per-chain, referenced by ID 1 |
| Soulbound Ruleset (v5.0) | Deployed per-chain, referenced by ID 2 |
| Blacklist Ruleset (v5.0) | Deployed per-chain, referenced by ID 3 |

## Reference Files

- `references/ruleset-system.md` -- Ruleset system details, validation flow, authorization
- `references/list-management.md` -- List creation, account management, list types

## Related Skills

- **transfer-validator-config** -- Generate configuration scripts for rulesets, lists, and account freezing
- **permitc-protocol** -- PermitC approval system (inherited by the Transfer Validator)
- **apptoken-general** -- Ecosystem overview, token standards, cross-protocol integration
