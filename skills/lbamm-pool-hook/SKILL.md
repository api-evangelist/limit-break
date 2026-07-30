---
name: lbamm-pool-hook
description: Generate complete ILimitBreakAMMPoolHook Solidity implementations
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of desired pool hook behavior]"
---

# LBAMM Pool Hook Generator

Generate a complete `ILimitBreakAMMPoolHook` implementation based on the user's requirements.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **LBAMM Core**: `lib/lbamm-core/` must exist. If not: `forge install limitbreakinc/lbamm-core`
3. **Remappings**: `remappings.txt` needs `@limitbreak/lbamm-core/=lib/lbamm-core/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: LBAMM is in audit and not yet on public networks. For local testing, use `lbamm-examples` -- see `lbamm-protocol/references/local-dev.md`. Generated scripts should read addresses from environment variables (`vm.envAddress`) so they work with the local Anvil deployment.

## Instructions

1. **Parse requirements** from `$ARGUMENTS` -- identify the pool hook behavior (dynamic fees, pool creation gating, LP access control, oracle pricing for single provider pools, etc.)

2. **Resolve ambiguities before generating** -- if $ARGUMENTS mentions external contracts without addresses (e.g., "a price oracle", "a registry", "a token"), ask the user whether they have an existing contract (and get the address) or want a new one generated. Similarly, clarify pool type (Dynamic vs Single Provider) if not obvious, or specific configuration values that would change the implementation. Ask everything in a single message. If clear, proceed.

3. **Determine capabilities needed:**
   - Dynamic fee calculation via `getPoolFeeForSwap()` (pool must be created with fee = `55_555`)
   - Pool creation validation via `validatePoolCreation()`
   - Liquidity operation validation via `validatePoolAddLiquidity()`, `validatePoolRemoveLiquidity()`, `validatePoolCollectFees()`
   - If targeting Single Provider pools: implement `ISingleProviderPoolHook` extension with `getPoolPriceForSwap()` and `getPoolLiquidityProvider()`

4. **Generate a complete Solidity contract** that:
   - Uses `pragma solidity 0.8.24;`
   - Implements `ILimitBreakAMMPoolHook` (or `ISingleProviderPoolHook` if targeting single provider pools)
   - Implements `poolHookManifestUri()`
   - Includes appropriate state management and access control

5. **Note fee bounds:**
   - `poolFeeBPS <= 10_000` for input-based swaps
   - `poolFeeBPS < 10_000` (strictly less) for output-based swaps
   - Hook fees on liquidity ops are capped by caller's `maxHookFee0`/`maxHookFee1`

## Reference Files

- `references/pool-hook-interface.md` -- Full interface, `ISingleProviderPoolHook` extension, dynamic fee sentinel
- `references/pool-hook-patterns.md` -- Dynamic fee hook, access-controlled pool, oracle pricing hook examples
- Protocol knowledge: `lbamm-protocol`

## When to Use This vs Other Hook Skills

- **This skill (lbamm-pool-hook)**: Per-pool hooks set at pool creation. Use for dynamic fees (`55_555` sentinel), LP access control, or oracle pricing. **Required** for Single Provider pools (must implement `ISingleProviderPoolHook`).
- **lbamm-token-hook**: Per-token hooks set globally. Use when behavior should apply to ALL pools involving a token (swap gating, fee collection, flashloan control).
- **lbamm-standard-hook**: Configure the pre-deployed first-party hook. Use when built-in settings are sufficient.

## Dependencies

```
@limitbreak/lbamm-core/src/interfaces/hooks/ILimitBreakAMMPoolHook.sol
lbamm-pool-type-single-provider/src/interfaces/ISingleProviderPoolHook.sol
@limitbreak/lbamm-core/src/DataTypes.sol
```
