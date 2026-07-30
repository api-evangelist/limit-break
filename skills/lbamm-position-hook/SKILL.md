---
name: lbamm-position-hook
description: Generate complete ILimitBreakAMMLiquidityHook Solidity implementations for per-position liquidity policies
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of desired position hook behavior]"
---

# LBAMM Position Hook Generator

Generate a complete `ILimitBreakAMMLiquidityHook` implementation based on the user's requirements. Position hooks enforce per-position policies on liquidity operations (add, remove, collect fees).

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **LBAMM Core**: `lib/lbamm-core/` must exist. If not: `forge install limitbreakinc/lbamm-core`
3. **Remappings**: `remappings.txt` needs `@limitbreak/lbamm-core/=lib/lbamm-core/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: LBAMM is in audit and not yet on public networks. For local testing, use `lbamm-examples` -- see `lbamm-protocol/references/local-dev.md`. Generated scripts should read addresses from environment variables (`vm.envAddress`) so they work with the local Anvil deployment.

## Instructions

1. **Parse requirements** from `$ARGUMENTS` -- identify what the position hook should do (time-lock liquidity, KYC-gate LP positions, charge deposit/withdrawal fees, track rewards, etc.)

2. **Resolve ambiguities before generating** -- if $ARGUMENTS references external contracts without addresses (e.g., "a KYC registry", "a reward token", "an NFT collection"), ask whether to use an existing contract (get the address) or create one. Clarify time-lock durations, fee amounts, or other values if unspecified and impactful. Ask in a single message. If clear, proceed.

3. **Determine which callbacks are needed:**
   - `validatePositionAddLiquidity` -- validate/gate LP deposits, charge deposit fees
   - `validatePositionRemoveLiquidity` -- validate/gate LP withdrawals, charge withdrawal fees
   - `validatePositionCollectFees` -- validate fee collection, charge fees on fee claims

4. **Generate a complete Solidity contract** that:
   - Uses `pragma solidity 0.8.24;`
   - Implements `ILimitBreakAMMLiquidityHook`
   - Implements all three callback functions (unused ones can be minimal pass-through)
   - Implements `liquidityHookManifestUri()`
   - Includes appropriate state management and access control
   - Stores the LBAMM core address for caller validation

5. **Explain the position identity impact** -- the hook address becomes part of the base position ID:
   ```
   basePositionId = keccak256(abi.encodePacked(provider, liquidityHook, poolId))
   ```
   Different hooks create different positions even for the same provider and pool. This means positions created with this hook are distinct from positions without it.

6. **Explain fee behavior:**
   - All three callbacks return `(hookFee0, hookFee1)`
   - Hook fees are accrued in core, not transferred immediately
   - Fees across all hook tiers are aggregated and compared against `maxHookFee0`/`maxHookFee1` set by the caller
   - Return `(0, 0)` from callbacks that don't charge fees

## Reference Files

- `references/position-hook-interface.md` -- Full interface, callback signatures, position identity, hook call ordering
- `references/position-hook-patterns.md` -- Annotated examples: time-lock, KYC-gated, fee-on-deposit
- Protocol knowledge: `lbamm-protocol`

## When to Use This vs Other Hook Skills

- **This skill (lbamm-position-hook)**: Per-position hooks set per liquidity operation via `LiquidityModificationParams.liquidityHook`. Use for position-specific policies -- time locks, per-position access control, position-level fee collection, reward tracking. The hook becomes part of the position identity.
- **lbamm-token-hook**: Per-token hooks set globally via `setTokenSettings`. Broadest surface -- 10 flags covering swaps, liquidity, pool creation, flashloans, and handler orders. Use when behavior should apply to ALL pools involving a token.
- **lbamm-pool-hook**: Per-pool hooks set at pool creation. Use for pool-specific behavior like dynamic fees, LP access control, or oracle pricing (required for Single Provider pools).
- **lbamm-standard-hook**: Configure the pre-deployed first-party hook. Use when built-in settings (trading fees, whitelists, pause, pool disabling) are sufficient -- no custom Solidity needed.

## Key Rules

- Position hooks are called as **Tier 3** in the hook call ordering for liquidity ops: token hooks (Tier 1) -> position hooks (Tier 3) -> pool hooks (Tier 2)
- Hooks MUST revert to block an operation -- returning `(0, 0)` fees still allows it
- The `LiquidityContext` provides `provider`, `token0`, `token1`, and `positionId`
- Hook fees are capped by the caller's `maxHookFee0`/`maxHookFee1` -- if the hook returns more, the transaction reverts
- Validate `msg.sender == LBAMM_CORE` in callbacks to prevent unauthorized invocation

## Dependencies

```
@limitbreak/lbamm-core/src/interfaces/hooks/ILimitBreakAMMLiquidityHook.sol
@limitbreak/lbamm-core/src/DataTypes.sol
```

## Related Skills

- **lbamm-pool** -- Generate pool creation code (positions are created within pools)
- **lbamm-test** -- Generate Foundry test suites for position hook contracts
