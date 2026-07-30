---
name: tokenmaster-integrator
description: Generate TokenMaster transaction integration code -- buy/sell/spend flows, advanced orders, EIP-712 signing, oracles
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of transaction or integration requirements]"
---

# TokenMaster Integrator

Generate transaction integration code for TokenMaster buy/sell/spend flows.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **TokenMaster**: `lib/tm-tokenmaster/` must exist. If not: `forge install limitbreakinc/tm-tokenmaster`
3. **Remappings**: `remappings.txt` needs `@limitbreak/tm-tokenmaster/=lib/tm-tokenmaster/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: For local development, use `apptoken-dev` to bootstrap an Anvil node with all Apptoken protocols at their deterministic addresses -- see `apptoken-general/references/project-setup.md`.

## Instructions

1. Parse requirements from $ARGUMENTS
2. **Resolve ambiguities before generating** -- if $ARGUMENTS references specific tokens or hooks without addresses, or the transaction type (basic vs advanced, with/without cosigner) isn't clear, ask in a single message. If clear, proceed.
3. Determine transaction type: basic buy/sell, advanced buy/sell with signed orders, spend (always requires signed order)
4. Generate integration code with proper struct construction, approvals, and signing
5. Handle: native token via msg.value, ERC20 approvals, PermitC transfers, cosigning, oracle integration

## Reference Files

- `references/transaction-flows.md` -- Buy, sell, spend flows with code examples
- `references/advanced-orders.md` -- Signed orders, oracle integration, cosigning
- Protocol knowledge: `tokenmaster-protocol`

## Common Mistakes

- `sellTokensAdvanced` ALWAYS calls the hook -- there is no `address(0)` guard. If `signedOrder.hook` is `address(0)`, the call reverts. Every advanced sell order MUST specify a valid hook contract.
- `buyTokensAdvanced` and `spendTokens` DO have `address(0)` guards -- hooks are optional for these.
- `withdrawFees` can be called by EITHER the `TOKENMASTER_FEE_COLLECTOR_ROLE` holder OR the `partnerFeeRecipient` for a specific token.

## When to Use This vs Other Skills

- **This skill (tokenmaster-integrator)**: Generate transaction integration code -- buy, sell, spend flows, EIP-712 signing, advanced orders. Use when building frontends, backends, or contracts that submit transactions to the TokenMaster Router.
- **tokenmaster-hook**: Generate hook contracts for post-execution logic. Use when you need custom behavior triggered after a transaction completes.
- **tokenmaster-deployer**: Generate deployment scripts. Use *before* integration -- you need a deployed token first.

## Related Skills

- **tokenmaster-hook** -- Generate hook contracts that execute after advanced order transactions
- **tokenmaster-deployer** -- Generate token deployment scripts (prerequisite for trading)
- **permitc-protocol** -- PermitC fundamentals (approvals, signing, nonces) used for advanced buy orders
