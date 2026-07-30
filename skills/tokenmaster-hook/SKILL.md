---
name: tokenmaster-hook
description: Generate TokenMaster hook contracts -- buy hooks, sell hooks, spend hooks for advanced order post-execution logic
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of hook requirements - what should happen after buy/sell/spend]"
---

# TokenMaster Hook Generator

Generate hook contracts for TokenMaster advanced order post-execution logic.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **TokenMaster**: `lib/tm-tokenmaster/` must exist. If not: `forge install limitbreakinc/tm-tokenmaster`
3. **Remappings**: `remappings.txt` needs `@limitbreak/tm-tokenmaster/=lib/tm-tokenmaster/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: For local development, use `apptoken-dev` to bootstrap an Anvil node with all Apptoken protocols at their deterministic addresses -- see `apptoken-general/references/project-setup.md`.

## Instructions

1. Parse requirements from $ARGUMENTS
2. **Resolve ambiguities before generating** -- if $ARGUMENTS references external contracts (reward tokens, staking contracts, NFT collections) without addresses, ask whether to use existing deployments (get addresses) or generate placeholders. Clarify hook type if multiple could fit. Ask in a single message. If clear, proceed.
3. Determine hook type(s): ITokenMasterBuyHook, ITokenMasterSellHook, ITokenMasterSpendHook
4. Generate complete hook contract with: interface implementation, caller validation (must be Router), token/identifier validation, custom logic
5. Hooks execute AFTER the core transaction completes
6. hookExtraData decoding is the hook's responsibility

## Reference Files

- `references/hook-interfaces.md` -- ITokenMasterBuyHook, ITokenMasterSellHook, ITokenMasterSpendHook signatures
- `references/hook-patterns.md` -- Annotated hook implementation examples
- Protocol knowledge: `tokenmaster-protocol`

## Common Mistakes

- Sell hooks are NOT optional for `sellTokensAdvanced` -- if the order specifies `address(0)` as hook, the transaction reverts. Always deploy a sell hook contract even if it's a no-op.
- Hooks must validate `msg.sender == TOKENMASTER_ROUTER` -- anyone can call the hook function directly otherwise.
- Hooks must validate the `tokenMasterToken` parameter matches the expected token -- a single hook contract may serve multiple tokens, and without validation an attacker could invoke it with a different token.

## When to Use This vs Other Skills

- **This skill (tokenmaster-hook)**: Generate hook contracts that execute post-transaction logic for advanced orders. Use when you need custom behavior after buys, sells, or spends (e.g., reward distribution, staking, item delivery).
- **tokenmaster-integrator**: Generate transaction code that calls the Router. Use when building the frontend/backend that submits buy/sell/spend transactions.
- **tokenmaster-deployer**: Generate deployment scripts. Use *before* hooks -- you need a deployed token first.

## Related Skills

- **tokenmaster-integrator** -- Generate transaction code that invokes hooks via advanced orders
- **tokenmaster-deployer** -- Generate deployment scripts (hooks are configured post-deployment)
