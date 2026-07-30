---
name: tokenmaster-protocol
description: TokenMaster background knowledge -- architecture, data structures, pool types, fee distribution, events. Use this skill whenever the user asks about TokenMaster, backed tokens, bonding curves, token pools (Standard/Stable/Promotional), buy/sell/spend flows, or any code referencing the TokenMaster Router at 0x0E00009d00d1000069ed00A908e00081F5006008.
user-invocable: false
---

# TokenMaster Protocol Knowledge Base

You have deep knowledge of the TokenMaster protocol. Use this skill's reference files whenever the user's question involves TokenMaster concepts, ERC-20C token deployment, pool mechanics, buy/sell/spend flows, or any of the related TokenMaster skills.

## Architecture Overview

TokenMaster is a next-generation fungible token protocol for onchain consumer economies, built on ERC-20C.

1. **TokenMaster Router** (`0x0E00009d00d1000069ed00A908e00081F5006008`) -- Central entry point. Inherits RoleSetClient, TrustedForwarderERC2771Context, EIP712, TstorishReentrancyGuard. Delegates to pool contracts deployed via CREATE2 factories.

2. **Pool Types** -- Pluggable modules defining token economics:
   - **Standard** -- Flexible tokenomics with bonding curve, buy/sell spreads, demand fees, and creator fees
   - **Stable** -- Fixed price pool for stablecoins and predictable-value tokens
   - **Promotional** -- No market value, free distribution, monetizable via spend hooks

3. **Order System** -- Basic buy/sell (permissionless) and advanced orders (signed by order signer, support hooks, cosigning, oracles)

4. **Hook System** -- Post-execution hooks for advanced orders:
   - `ITokenMasterBuyHook` -- Executes after buy completes
   - `ITokenMasterSellHook` -- Executes after sell completes
   - `ITokenMasterSpendHook` -- Executes after spend completes

## Repository

GitHub: `https://github.com/limitbreakinc/tm-tokenmaster`

### Project Setup

1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **TokenMaster**: `lib/tm-tokenmaster/` must exist. If not: `forge install limitbreakinc/tm-tokenmaster`
3. **Remappings**: `remappings.txt` needs `@limitbreak/tm-tokenmaster/=lib/tm-tokenmaster/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: For local development, use `apptoken-dev` to bootstrap an Anvil node with all Apptoken protocols at their deterministic addresses -- see `apptoken-general/references/project-setup.md`.

## Contract Addresses

| Contract | Address |
|---|---|
| TokenMaster Router | `0x0E00009d00d1000069ed00A908e00081F5006008` |
| Standard Factory | `0x000000c5F2DF717F497BeAcCE161F8b042310d17` |
| Stable Factory | `0x0000006a50a9c9Efae8875266ff222579fC2F449` |
| Promotional Factory | `0x00000014D04B7d1Cad1960eA8980A9af5De2104e` |

## Reference Files

- `references/data-structures.md` -- Core structs, enums, and parameter encoding
- `references/pool-types.md` -- Standard, Stable, and Promotional pool details, pricing, and fee models
- `references/events.md` -- Event emissions for buy, sell, spend, deployment, and configuration

## Related Skills

- **tokenmaster-deployer** -- Generate token deployment scripts with pool type selection
- **tokenmaster-integrator** -- Generate buy/sell/spend transaction integration code
- **tokenmaster-hook** -- Generate buy, sell, and spend hook contracts
- **apptoken-general** -- Ecosystem overview, ERC-20C token standards, cross-protocol integration
