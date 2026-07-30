---
name: wrapped-native-integrator
description: Generate Solidity contracts and scripts that integrate with Wrapped Native (WNATIVE)
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of integration -- e.g., 'router that wraps ETH and forwards WNATIVE', 'gas sponsorship service with permitted withdrawals']"
---

# Wrapped Native Integration Generator

Generate Solidity contracts that interact with the Wrapped Native (WNATIVE) contract at `0x6000030000842044000077551D00cfc6b4005900`.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **Wrapped Native**: `lib/wrapped-native/` must exist. If not: `forge install limitbreakinc/wrapped-native`
3. **Remappings**: `remappings.txt` needs `@limitbreak/wrapped-native/=lib/wrapped-native/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: For local development, use `apptoken-dev` to bootstrap an Anvil node with all Apptoken protocols at their deterministic addresses -- see `apptoken-general/references/project-setup.md`.

## Instructions

1. **Parse integration requirements** from `$ARGUMENTS`

2. **Resolve ambiguities before generating** -- if $ARGUMENTS doesn't make the integration pattern clear (basic wrapping vs permits vs gas sponsorship), or references external addresses (fee recipients, operators) without specifying them, ask in a single message. If clear, proceed.

3. **Determine the integration pattern** based on what the user needs:
   - **Basic wrapping/unwrapping**: Deposit, withdraw, depositTo, withdrawSplit
   - **Payable ERC-20 operations**: Combined deposit+approve, deposit+transfer
   - **Permit-based gasless transfers**: EIP-712 signatures for `permitTransfer`
   - **Gas sponsorship / permitted withdrawals**: `doPermittedWithdraw` with convenience fees
   - **Protocol integration**: Using WNATIVE as a payment or reserve token

4. **Generate the integration contract** with:
   - Correct interface import (`IWrappedNative` for basic, `IWrappedNativeExtended` for permits/enhanced features)
   - Proper use of `payable` functions when combining deposit + action
   - Native token handling (`receive()` if the contract needs to accept unwrapped native)
   - EIP-712 signature construction if using permits

5. **Handle key integration details:**
   - WNATIVE address is deterministic: `0x6000030000842044000077551D00cfc6b4005900`
   - When calling payable `transferFrom` with `msg.value`, deposits credit the `from` address, NOT `msg.sender`
   - Same for `permitTransfer` -- deposits credit `from`
   - Contracts receiving native tokens from `withdraw`/`withdrawToAccount` must have a `receive()` or `fallback()` function
   - `withdrawSplit` arrays must be equal length

6. **Generate a Foundry test** if the integration involves non-trivial logic (permits, fee calculations, multi-step flows).

## Reference Files

- `references/integration-patterns.md` -- Common integration patterns with annotated Solidity examples
- Protocol knowledge: `wrapped-native-protocol`

## Common Mistakes

- Forgetting that payable `transferFrom` and `permitTransfer` credit deposits to `from`, not `msg.sender` -- operator contracts will not receive the deposited funds
- Not implementing `receive()` on contracts that call `withdraw` or `withdrawToAccount` to themselves -- the native transfer will revert
- Using WETH9's `deposit()` pattern (call + value) when `depositTo` would be more gas-efficient for crediting another address
- Signing permits with the wrong domain separator -- must use name `"Wrapped Native"`, version `"1"`, and the deterministic contract address
- Not checking `isNonceUsed` before submitting a permit transaction -- the nonce may have been revoked or already consumed

## When to Use This vs Other Skills

- **This skill (wrapped-native-integrator)**: Generate contracts that wrap, unwrap, transfer, or permit WNATIVE tokens. Use when the core task is interacting with the Wrapped Native contract.
- **lbamm-integrator**: Use when building swap/liquidity integrations with LBAMM. If the pool uses WNATIVE, the LBAMM integrator handles the wrapping internally.
- **tokenmaster-integrator**: Use when building buy/sell/spend flows against TokenMaster. If the reserve asset is WNATIVE, the TokenMaster integrator handles it.
- **payment-processor-exchange**: Use when building NFT marketplace integrations. Payment Processor accepts WNATIVE directly.

## Related Skills

- **wrapped-native-protocol** -- Deep protocol knowledge, full API reference, permit system details
- **lbamm-integrator** -- DEX integration (LBAMM pools may use WNATIVE)
- **tokenmaster-integrator** -- Token economy integration (pools may use WNATIVE as reserve)
