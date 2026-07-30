---
name: lbamm-permit-handler
description: Generate integration code for the LBAMM PermitTransferHandler -- gasless swap execution via EIP-712 signed permits with optional cosigner. Generates Foundry scripts, signing code, and transferData encoding for fill-or-kill and partial fill permit swaps.
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of permit-based swap integration]"
---

# LBAMM Permit Handler Integration Generator

Generate complete integration code for executing gasless swaps through the LBAMM PermitTransferHandler. The PermitTransferHandler enables a maker to sign a permit offline, and an executor to fill it on-chain by passing the signed permit data through the AMM swap call.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **LBAMM Core**: `lib/lbamm-core/` must exist. If not: `forge install limitbreakinc/lbamm-core`
3. **LBAMM Hooks and Handlers**: `lib/lbamm-hooks-and-handlers/` must exist. If not: `forge install limitbreakinc/lbamm-hooks-and-handlers`
4. **Remappings**: `remappings.txt` needs both `@limitbreak/lbamm-core/=lib/lbamm-core/` and `@limitbreak/lbamm-hooks-and-handlers/=lib/lbamm-hooks-and-handlers/`. Append if missing.
5. **Local testing**: LBAMM is in audit and not yet on public networks. For local testing, use `lbamm-examples` -- see `lbamm-protocol/references/local-dev.md`. Generated scripts should read addresses from environment variables (`vm.envAddress`) so they work with the local Anvil deployment.

## Instructions

1. **Parse requirements** from `$ARGUMENTS` -- determine what kind of permit integration the user needs

2. **Resolve ambiguities before generating** -- if $ARGUMENTS doesn't specify token/pool addresses, permit type (fill-or-kill vs partial fill) isn't clear, or whether cosigning is needed, ask in a single message. If clear, proceed.

3. **Determine the permit type:**
   - **Fill-or-Kill**: Atomic, all-or-nothing execution. Entire amount must be filled or transaction reverts. Use for single exact swaps.
   - **Partial Fill**: Supports incremental fills across multiple executions. PermitC tracks cumulative fill state. Use for limit orders or large orders that may fill over time.

4. **Generate complete integration code** covering:
   - Struct construction for `FillOrKillPermitTransfer` or `PartialFillPermitTransfer`
   - EIP-712 signing of the PermitC permit (domain-bound to the **PermitC contract**)
   - Optional cosignature generation (domain-bound to the **PermitTransferHandler contract** -- different EIP-712 domain)
   - `transferData` encoding using `bytes.concat` (NOT `abi.encode`)
   - AMM swap call with the encoded transfer data
   - Hook integration if validation is needed

5. **Explain the two-domain signing model:**
   - The maker's permit signature uses the PermitC EIP-712 domain
   - The cosigner's cosignature uses the PermitTransferHandler EIP-712 domain (`name: "PermitTransferHandler", version: "1"`)
   - These are different contracts with different domain separators

6. **Handle edge cases:**
   - Cosigner setup: `address(0)` means no cosigner required
   - Partial fill mode matching: `permitAmountSpecified` sign must match `swapOrder.amountSpecified` sign
   - `feeOnTop` is NOT signed -- only `exchangeFee` (recipient + BPS) is part of the signed data
   - Fill-or-kill cosignature nonce is always 0 (reusable) since the permit itself can only be used once

## Reference Files

- `references/permit-handler-reference.md` -- Complete struct definitions, EIP-712 typehashes, signing flow, cosigner system, encoding examples, error reference
- Handler architecture: `lbamm-handler-protocol`
- PermitC fundamentals: `permitc-protocol`
- Protocol knowledge: `lbamm-protocol`

## Common Mistakes

- Using `abi.encode(address, bytes)` for transferData instead of `bytes.concat` -- the AMM reads raw bytes, not ABI-encoded
- Signing the permit with the PermitTransferHandler domain instead of the PermitC domain (or vice versa for the cosignature)
- Forgetting the `bytes1` permit type discriminator as the first byte of handlerExtraData
- Setting `permitAmountSpecified` positive for an output-based swap (must be negative for output-based)
- Not querying the current `masterNonce` from the PermitC contract before signing

## Dependencies

```
@limitbreak/lbamm-core/src/interfaces/ILimitBreakAMMTransferHandler.sol
@limitbreak/lbamm-core/src/DataTypes.sol
@limitbreak/lbamm-hooks-and-handlers/src/handlers/permit/DataTypes.sol
@limitbreak/lbamm-hooks-and-handlers/src/handlers/permit/Constants.sol
```

## Related Skills

- **lbamm-handler-protocol** -- Transfer handler architecture and custom handler development
- **permitc-protocol** -- PermitC fundamentals (time-bound approvals, signing patterns, nonce management)
- **lbamm-integrator** -- Generate router contracts that use handlers via directSwap
- **lbamm-test** -- Generate Foundry test suites for permit-based swap flows
