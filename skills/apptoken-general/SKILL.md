---
name: apptoken-general
description: Apptoken ecosystem knowledge -- token standards (ERC-721C, ERC-1155C, ERC-20C), apptoken categories, cross-protocol integration patterns. Use this skill whenever the user asks about apptokens, Creator Token Standards, ERC-20C/ERC-721C/ERC-1155C, Limit Break tokens, cross-protocol integration, or how the apptoken ecosystem fits together.
user-invocable: false
---

# Apptoken Ecosystem Knowledge Base

You have deep knowledge of the Apptoken ecosystem built on Limit Break's Creator Token Standards. Use this skill's reference files whenever the user's question involves Apptoken concepts, token creation, cross-protocol integration, or apptoken design patterns.

## Ecosystem Overview

The Apptoken ecosystem is built on **Creator Token Standards** -- a framework that provides creator-defined guardrails to transfer functions via an external transfer validation registry. Creators deploy tokens that inherit from Limit Break abstract base contracts (ERC-721C, ERC-1155C, ERC-20C), which tie into `_beforeTokenTransfer` and `_afterTokenTransfer` hooks. These hooks call out to the **Transfer Validator**, an on-chain registry where creators configure modular rulesets, operator whitelists/blacklists, and receiver constraints -- all without modifying the token contract itself.

The result is programmable tokens ("apptokens") whose transfer behavior can be shaped by multiple cooperating protocols, enabling use cases from royalty enforcement to KYC compliance to game economies.

## Token Standards

| Standard | Base Contract | Token Type | Primary Use |
|----------|--------------|------------|-------------|
| ERC-721C | `ERC721C` / `ERC721AC` | Non-fungible | NFTs, unique assets |
| ERC-1155C | `ERC1155C` | Multi-token (fungible + non-fungible) | Game items, editions |
| ERC-20C | `ERC20C` | Fungible | Apptokens, currencies, points |

All standards share the same Transfer Validator integration pattern. Install via `forge install limitbreakinc/creator-token-standards`. GitHub: `https://github.com/limitbreakinc/creator-token-standards`.

## Protocol Ecosystem

| Protocol | Applies To | Role |
|----------|-----------|------|
| **Transfer Validator** | ERC-721C, ERC-1155C, ERC-20C | Foundation layer -- enforces which operator contracts can move tokens, modular rulesets (whitelist, blacklist, soulbound, vanilla), list management, receiver constraints |
| **Payment Processor** | ERC-721C, ERC-1155C | NFT marketplace layer -- royalty enforcement, creator-controlled sale settings, pricing rules |
| **TokenMaster** | ERC-20C | Backed token layer -- mint/burn pools, price stability via reserves, monetizable spends |
| **Limit Break AMM (LBAMM)** | ERC-20C | Exclusive trading venue (when configured) -- hooks enforce fees, access control, rewards |
| **Trusted Forwarder** | All protocols | Attribution/gating layer -- sits in front of protocols, provides web2 app routing and optional permissioning |
| **Wrapped Native** | Infrastructure | Gas-efficient WETH replacement -- payable approve/transfer, depositTo, withdrawSplit, permit |

## Key Addresses

- **Transfer Validator V5**: `0x721C008fdff27BF06E7E123956E2Fe03B63342e3`
- **Payment Processor V3**: `0x9a1D00000000fC540e2000560054812452eB5366`
- **TokenMaster Router**: `0x0E00009d00d1000069ed00A908e00081F5006008`
- **EOA Registry**: `0xE0A0004Dfa318fc38298aE81a666710eaDCEba5C`
- **PermitC**: `0x000000fCE53B6fC312838A002362a9336F7ce39B`
- **Trusted Forwarder Factory**: `0xFF0000B6c4352714cCe809000d0cd30A0E0c8DcE`
- **Wrapped Native**: `0x6000030000842044000077551D00cfc6b4005900`

## Reference Files

- `references/token-standards.md` -- How to create ERC-721C, ERC-1155C, and ERC-20C tokens, critical override patterns, validator setup
- `references/ecosystem.md` -- Cross-protocol integration model, Trusted Forwarder, Wrapped Native, EOA Registry
- `references/apptoken-patterns.md` -- The 6 apptoken categories (client-side, trade, temporal, spend-only, compliant, backed) and their implementation patterns
- `references/chains.md` -- Supported chains, deterministic deployment addresses, forge dependencies
- `references/project-setup.md` -- Foundry project detection, dependency installation, remapping configuration

## Related Skills

- **apptoken-designer** -- Design an apptoken: map requirements to protocols, configuration, and implementation steps
- **transfer-validator-protocol** -- Transfer Validator protocol details: modular rulesets, list management, operator/receiver policies
- **payment-processor-protocol** -- Payment Processor protocol details: sale configurations, royalty enforcement, pricing modes
- **tokenmaster-protocol** -- TokenMaster protocol details: mint/burn pools, price curves, spend mechanics
- **lbamm-protocol** -- LBAMM protocol details: pool types, hook system, swap/liquidity interfaces, fee architecture
- **permitc-protocol** -- PermitC approval system: time-bound approvals, permit transfers, signing patterns
- **wrapped-native-protocol** -- Wrapped Native token: gas-efficient WETH replacement, permits, deterministic deployment
- **lbamm-handler-protocol** -- LBAMM transfer handler architecture: custom settlement, permit handler, CLOB
