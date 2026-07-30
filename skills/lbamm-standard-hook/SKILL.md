---
name: lbamm-standard-hook
description: Configure AMM Standard Hook via Creator Hook Settings Registry
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of desired token configuration]"
---

# LBAMM Standard Hook Configurator

Generate configuration scripts/contracts for the first-party AMM Standard Hook + Creator Hook Settings Registry.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **LBAMM Core**: `lib/lbamm-core/` must exist. If not: `forge install limitbreakinc/lbamm-core`
3. **LBAMM Hooks and Handlers**: `lib/lbamm-hooks-and-handlers/` must exist. If not: `forge install limitbreakinc/lbamm-hooks-and-handlers`
4. **Remappings**: `remappings.txt` needs both `@limitbreak/lbamm-core/=lib/lbamm-core/` and `@limitbreak/lbamm-hooks-and-handlers/=lib/lbamm-hooks-and-handlers/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: LBAMM is in audit and not yet on public networks. For local testing, use `lbamm-examples` -- see `lbamm-protocol/references/local-dev.md`. Generated scripts should read addresses from environment variables (`vm.envAddress`) so they work with the local Anvil deployment.

## Instructions

1. **Parse desired configuration** from `$ARGUMENTS` -- identify what the token issuer wants to configure (trading fees, whitelists, price bounds, trading pause, pool disabling, etc.)

2. **Resolve ambiguities before generating** -- if $ARGUMENTS mentions a token without its address, references whitelists without specifying which addresses to include, or leaves fee amounts unspecified, ask in a single message. If clear, proceed.

3. **Map requirements to `HookTokenSettings` fields:**
   - `tradingIsPaused` -- Pause all trading
   - `blockDirectSwaps` -- Block direct (P2P) swaps
   - `checkDisabledPools` -- Enable pool disable/enable by token issuer
   - `tokenFeeBuyBPS` / `tokenFeeSellBPS` -- Fees on token buys/sells (in token)
   - `pairedFeeBuyBPS` / `pairedFeeSellBPS` -- Fees on buys/sells (in paired token)
   - `minFeeAmount` / `maxFeeAmount` -- Fee amount bounds
   - `poolTypeWhitelistId` -- Restrict which pool types can be used
   - `pairedTokenWhitelistId` -- Restrict which tokens can be paired
   - `lpWhitelistId` -- Restrict who can provide liquidity

4. **Generate a Foundry script** that:
   - Creates whitelists via the Registry (LP, pair token, pool type) if needed
   - Calls `registry.setTokenSettings()` with the correct `HookTokenSettings`
   - Calls LBAMM core `setTokenSettings()` with correct packed flags
   - Syncs hooks if needed

5. **Include LBAMM core `setTokenSettings` call** with correct packed flags:
   - The AMM Standard Hook supports flags: beforeSwap (0), afterSwap (1), addLiquidity (2), poolCreation (5), flashloans (7), flashloansValidateFee (8), handlerOrderValidate (9)
   - Flags removeLiquidity (3), collectFees (4), and hookManagesFees (6) are NOT supported and will cause reverts if set

6. **Explain each setting** and distinguish:
   - Pool disabling (`checkDisabledPools` + `setPoolDisabled`) vs trading pause (`tradingIsPaused`)
   - Token fees vs paired token fees
   - Whitelist IDs (0 = no restriction)

## Reference Files

- `references/standard-hook-api.md` -- Registry interface, IAMMStandardHook, who can administer
- `references/standard-hook-configs.md` -- Common configuration examples
- Protocol knowledge: `lbamm-protocol`

## When to Use This vs Other Hook Skills

- **This skill (lbamm-standard-hook)**: Configure the pre-deployed first-party AMM Standard Hook via the Creator Hook Settings Registry. No custom Solidity needed -- just configuration scripts. Use when built-in settings (trading fees, whitelists, pause, pool disabling) are sufficient.
- **lbamm-token-hook**: Generate a custom `ILimitBreakAMMTokenHook` implementation. Use when you need behavior beyond what the standard hook provides.
- **lbamm-pool-hook**: Generate a custom `ILimitBreakAMMPoolHook` implementation. Use for per-pool behavior like dynamic fees or oracle pricing.
