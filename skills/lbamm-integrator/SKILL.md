---
name: lbamm-integrator
description: Generate DEX integration contracts that call LBAMM
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of integration requirements]"
---

# LBAMM Integration Generator

Generate router/aggregator/frontend contracts that integrate with LBAMM.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **LBAMM Core**: `lib/lbamm-core/` must exist. If not: `forge install limitbreakinc/lbamm-core`
3. **Remappings**: `remappings.txt` needs `@limitbreak/lbamm-core/=lib/lbamm-core/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: LBAMM is in audit and not yet on public networks. For local testing, use `lbamm-examples` -- see `lbamm-protocol/references/local-dev.md`. Generated scripts should read addresses from environment variables (`vm.envAddress`) so they work with the local Anvil deployment.

## Instructions

1. **Parse integration requirements** from `$ARGUMENTS`

2. **Resolve ambiguities before generating** -- if $ARGUMENTS doesn't specify key addresses (pool IDs, token addresses, router deployment target) or leaves the integration pattern unclear (single swap vs multi-hop, basic vs with handler), ask in a single message. If clear, proceed.

3. **Determine which LBAMM entry points are needed:**
   - `singleSwap` -- Single-pool swaps
   - `multiSwap` -- Multi-hop routes
   - `directSwap` -- Peer-to-peer swaps (with transfer handlers)
   - `createPool` -- Pool deployment
   - `addLiquidity` / `removeLiquidity` / `collectFees` -- Liquidity management

4. **Generate complete integration contract** with:
   - Proper struct construction (`SwapOrder`, `BPSFeeWithRecipient`, `FlatFeeWithRecipient`, etc.)
   - Token approval handling
   - Native token (ETH) wrapping if needed
   - Transfer handler data encoding if needed
   - Hook data encoding if needed

5. **Handle key integration details:**
   - The router becomes the **executor** -- hooks see it as `msg.sender`
   - Token ordering: `token0 < token1` always
   - `amountSpecified > 0` = input swap, `< 0` = output swap
   - `deadline` must be in the future
   - `recipient` cannot be `address(0)`

6. **Note**: The router is the executor for hook-policy purposes. If a token hook restricts executors, the router must be whitelisted.

## Reference Files

- `references/swap-interface.md` -- singleSwap, multiSwap, directSwap details and struct construction
- `references/liquidity-interface.md` -- createPool, addLiquidity, removeLiquidity, collectFees
- `references/integration-patterns.md` -- Router, aggregator, liquidity manager examples
- Protocol knowledge: `lbamm-protocol`

## Common Mistakes

- Not whitelisting LBAMM as an operator in the Transfer Validator for the token -- swaps will revert
- Forgetting that the router contract is the executor -- hooks see `msg.sender = router`, not the end user
- Using `amountSpecified = 0` -- this is invalid. Positive = input swap, negative = output swap.
- Setting fee recipient to `address(0)` when fee amount is non-zero -- will revert with `LBAMM__FeeRecipientCannotBeAddressZero`

## Dependencies

```
@limitbreak/lbamm-core/src/interfaces/core/ILimitBreakAMMSwap.sol
@limitbreak/lbamm-core/src/interfaces/core/ILimitBreakAMMLiquidity.sol
@limitbreak/lbamm-core/src/DataTypes.sol
```

## Related Skills

- **lbamm-handler-protocol** -- Transfer handler architecture and custom handler development
- **lbamm-permit-handler** -- Generate PermitTransferHandler integration (gasless swaps via directSwap)
- **lbamm-clob** -- Generate CLOBTransferHandler integration (limit orders via directSwap)
- **lbamm-test** -- Generate Foundry test suites for integration contracts
