---
name: lbamm-token-hook
description: Generate complete ILimitBreakAMMTokenHook Solidity implementations
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of desired hook behavior]"
---

# LBAMM Token Hook Generator

Generate a complete `ILimitBreakAMMTokenHook` implementation based on the user's requirements.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **LBAMM Core**: `lib/lbamm-core/` must exist. If not: `forge install limitbreakinc/lbamm-core`
3. **Remappings**: `remappings.txt` needs `@limitbreak/lbamm-core/=lib/lbamm-core/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: LBAMM is in audit and not yet on public networks. For local testing, use `lbamm-examples` -- see `lbamm-protocol/references/local-dev.md`. Generated scripts should read addresses from environment variables (`vm.envAddress`) so they work with the local Anvil deployment.

## Instructions

1. **Parse requirements** from `$ARGUMENTS` -- identify what the hook should do (fee collection, swap gating, LP restrictions, flashloan control, handler order validation, etc.)

2. **Resolve ambiguities before generating** -- if $ARGUMENTS mentions external contracts without addresses (e.g., "an NFT collection", "an oracle", "a whitelist registry"), ask the user whether they want to use an existing deployed contract (and get the address) or have a new one created as part of the example. Similarly, clarify fee percentages, admin roles, or other values if they're unspecified and would meaningfully change the implementation. Ask everything in a single message. If the prompt is specific enough, skip this step and proceed.

3. **Determine which of the 10 hook flags are needed:**
   - Bit 0: `beforeSwap` -- validate/gate swaps, charge input-side fees
   - Bit 1: `afterSwap` -- validate post-swap state, charge output-side fees
   - Bit 2: `addLiquidity` -- validate/gate LP deposits, charge deposit fees
   - Bit 3: `removeLiquidity` -- validate/gate LP withdrawals, charge withdrawal fees
   - Bit 4: `collectFees` -- validate fee collection, charge fees on fee claims
   - Bit 5: `poolCreation` -- validate/gate pool creation for this token
   - Bit 6: `hookManagesFees` -- hook manages its own fee accounting (fees tracked per-hook in core)
   - Bit 7: `flashloans` -- enable flashloans and set fees via `beforeFlashloan`
   - Bit 8: `flashloansValidateFee` -- validate when another token uses this as flashloan fee token
   - Bit 9: `handlerOrderValidate` -- validate transfer handler order creation

4. **Generate a complete Solidity contract** that:
   - Uses `pragma solidity 0.8.24;`
   - Implements `ILimitBreakAMMTokenHook`
   - Implements `hookFlags()` returning correct `(requiredFlags, supportedFlags)` bitmask
   - Implements all required callback functions (even unused ones should have minimal implementations)
   - Implements `tokenHookManifestUri()`
   - Includes appropriate access control (Ownable or role-based as needed)
   - Stores the LBAMM core address for callbacks that need it

5. **Explain flag choices** and security considerations

## Reference Files

- `references/token-hook-interface.md` -- Full interface, all 10 flags, `hookFlags()` negotiation rules
- `references/token-hook-patterns.md` -- Annotated examples for common token hook patterns
- Protocol knowledge: `lbamm-protocol`

## When to Use This vs Other Hook Skills

- **This skill (lbamm-token-hook)**: Per-token hooks set globally via `setTokenSettings`. Broadest surface -- 10 flags covering swaps, liquidity, pool creation, flashloans, and handler orders. Use when the behavior should apply to ALL pools involving a token.
- **lbamm-pool-hook**: Per-pool hooks set at pool creation. Use for pool-specific behavior like dynamic fees, LP access control, or oracle pricing (required for Single Provider pools).
- **lbamm-standard-hook**: Configure the pre-deployed first-party hook. Use when the built-in settings (trading fees, whitelists, pause, pool disabling) are sufficient -- no custom Solidity needed.

## Key Rules

- `hookFlags()` must return `(requiredFlags, supportedFlags)` where all required flags are also supported
- `beforeSwap` fee is on input token (input swap) or output token (output swap)
- `afterSwap` fee is on output token (input swap) or input token (output swap)
- Hooks MUST revert to block an operation -- returning 0 fees still allows it
- Hook fees are accrued in core and collected later
- The `hookForToken0` / `hookForInputToken` booleans tell you which side of the pool/swap this callback is for
- All liquidity hook callbacks can return `(hookFee0, hookFee1)` -- these are capped by caller's `maxHookFee0`/`maxHookFee1`

## Dependencies

```
@limitbreak/lbamm-core/src/interfaces/hooks/ILimitBreakAMMTokenHook.sol
@limitbreak/lbamm-core/src/DataTypes.sol
```
