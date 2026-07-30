---
name: lbamm-protocol
description: LBAMM protocol background knowledge -- architecture, interfaces, pool types, hooks, fees, and execution model. Use this skill whenever the user asks about LBAMM, Limit Break AMM, AMM hooks, swap mechanics, liquidity pools, pool types, token hooks, pool hooks, or any Solidity code that imports from @limitbreak/lbamm-core.
user-invocable: false
---

# LBAMM Protocol Knowledge Base

You have deep knowledge of the Limit Break AMM (LBAMM) protocol. Use this skill's reference files whenever the user's question involves LBAMM concepts, Solidity code targeting LBAMM, or any of the related skills.

## Architecture Overview

LBAMM is a modular AMM framework with these core components:

1. **Core AMM** (`LimitBreakAMM.sol`) -- Owns reserves and fee balances, constructs execution context, invokes hooks in deterministic order, performs token transfers and settlement, enforces protocol-level bounds and invariants.

2. **Pool Types** -- Pluggable modules defining swap math, liquidity accounting, and position identity:
   - **Dynamic** (`DynamicPoolType`) -- Concentrated liquidity with tick-based positions (Uniswap V3-like)
   - **Fixed** (`FixedPoolType`) -- Constant exchange rate with height-based liquidity
   - **Single Provider** (`SingleProviderPoolType`) -- Oracle/hook-determined pricing with one LP per pool

3. **3-Tier Hook System** -- Policy enforcement at three scopes:
   - **Token hooks** (`ILimitBreakAMMTokenHook`) -- Per-token, 10 flags, broadest surface (swaps, liquidity, pool creation, flashloans, handler orders)
   - **Pool hooks** (`ILimitBreakAMMPoolHook`) -- Per-pool, validates pool creation/liquidity ops, dynamic LP fees
   - **Position hooks** (`ILimitBreakAMMLiquidityHook`) -- Per-position identity, validates liquidity add/remove/collect fees

4. **Transfer Handlers** (`ILimitBreakAMMTransferHandler`) -- Modular settlement for swaps (permits, CLOB order books, custom)

5. **Fee Architecture** -- Multi-surface fee system:
   - Execution-level: exchange fee (BPS), fee-on-top (flat)
   - Hop-level: LP fee, token hook swap fees, protocol hop minimum fees
   - Liquidity-level: hook fees on add/remove/collect (capped by maxHookFee0/1)

## Reference Files

- `references/core-interfaces.md` -- Core data types (structs), `ILimitBreakAMMPoolType` interface, pool ID encoding
- `references/core-api.md` -- All AMM entry points: swap, liquidity, fee collection, flashloan, protocol management functions
- `references/events-errors.md` -- All events (`ILimitBreakAMMEvents`) and all errors (`Errors.sol`)
- `references/pool-types.md` -- Dynamic, Fixed, and Single Provider pool details and their data types
- `references/hook-system.md` -- 3-tier hook system, call ordering, flag definitions
- `references/fee-system.md` -- All fee surfaces, calculation logic, settlement model
- `references/execution-model.md` -- Execution context, atomicity, trust boundaries, protocol guarantees
- `references/local-dev.md` -- Local development environment setup via lbamm-examples (Anvil node, deployed addresses, testing workflow)

## Key Constants

- `DYNAMIC_POOL_FEE_BPS = 55_555` -- Sentinel for dynamic fee calculation via pool hook
- `MAX_BPS = 10_000` -- Maximum basis points (100%)
- Token setting flags: bits 0-9 of `packedSettings` (see `references/hook-system.md`)
- Pool ID encoding: bits 0-15 pool fee BPS, bits 16-143 creation hash, bits 144-255 pool type address

## Related Skills

- **lbamm-pool** -- Generate pool creation code (Dynamic, Fixed, Single Provider)
- **lbamm-integrator** -- Generate DEX integration contracts (routers, aggregators)
- **lbamm-token-hook** -- Generate per-token hook implementations (10 flags, broadest surface)
- **lbamm-pool-hook** -- Generate per-pool hook implementations (dynamic fees, access control, oracle pricing)
- **lbamm-standard-hook** -- Configure the first-party AMM Standard Hook via registry
- **lbamm-position-hook** -- Generate per-position hook implementations (time locks, LP gating, deposit fees)
- **lbamm-handler-protocol** -- Transfer handler architecture and custom handler development
- **lbamm-permit-handler** -- Generate PermitTransferHandler integration code (gasless swaps)
- **lbamm-clob** -- Generate CLOBTransferHandler integration code (on-chain limit orders)
- **lbamm-test** -- Generate Foundry test suites following LBAMM patterns
- **apptoken-general** -- Ecosystem overview, ERC-20C token standards, cross-protocol integration
