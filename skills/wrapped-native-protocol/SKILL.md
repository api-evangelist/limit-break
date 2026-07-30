---
name: wrapped-native-protocol
description: Wrapped Native (WNATIVE) protocol background knowledge -- architecture, interfaces, permits, gas benchmarks, and integration patterns. Use this skill whenever the user asks about Wrapped Native, WNATIVE, WETH replacement, native token wrapping, payable approve/transfer, depositTo, withdrawSplit, permitted withdrawals, gasless transfers, or any Solidity code that imports IWrappedNative or IWrappedNativeExtended, or references the address 0x6000030000842044000077551D00cfc6b4005900.
user-invocable: false
---

# Wrapped Native Protocol Knowledge Base

You have deep knowledge of the Wrapped Native (WNATIVE) protocol. Use this skill's reference files whenever the user's question involves WNATIVE concepts, Solidity code interacting with WNATIVE, or any of the related skills.

## Architecture Overview

Wrapped Native is a modern, gas-efficient ERC-20 wrapper for native blockchain tokens (e.g., Ether on Ethereum). It is designed as a canonical replacement for WETH9 with significant improvements:

1. **Deterministic Cross-Chain Address** -- Deployed at `0x6000030000842044000077551D00cfc6b4005900` on all EVM chains via CREATE2. Anyone can deploy it to a new chain using Limit Break's infrastructure deployment tool.

2. **Gas Efficiency** -- Every shared function is cheaper than WETH9. Transfer saves 627 gas, transferFrom (operator) saves 1,080 gas, withdraw saves 395 gas. Highly optimized with inline assembly.

3. **Payable ERC-20 Functions** -- `approve`, `transfer`, and `transferFrom` are all `payable`. When `msg.value > 0`, the function auto-deposits before executing the ERC-20 action, combining two transactions into one.

4. **Enhanced Deposit/Withdrawal** -- `depositTo(address)` credits a different account, `withdrawToAccount(address, uint256)` withdraws to a different address, `withdrawSplit(address[], uint256[])` batch-withdraws to multiple recipients.

5. **EIP-712 Permit System** -- Gasless transfer and withdrawal permits using typed data signatures. Supports both EOA (ECDSA) and smart contract (EIP-1271) signers. Permits use a nonce bitmap system with master nonce revocation.

6. **Convenience Fee Model** -- Permitted withdrawals support convenience fees for gas sponsorship services, with a 10% infrastructure tax on fees.

## Repository

GitHub: `https://github.com/limitbreakinc/wrapped-native`

### Project Setup

1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **Wrapped Native**: `lib/wrapped-native/` must exist. If not: `forge install limitbreakinc/wrapped-native`
3. **Remappings**: `remappings.txt` needs `@limitbreak/wrapped-native/=lib/wrapped-native/`. Append if missing -- do not overwrite existing entries.

## Contract Address

| Item | Value |
|---|---|
| Wrapped Native | `0x6000030000842044000077551D00cfc6b4005900` |
| Deployment Salt | `0x622bb53cf6d1e1110caa2e1e03d8dc7d4bd2e27dbd654c5c8206c0a4c167f934` |
| Deterministic Proxy | `0x4e59b44847b379578588920cA78FbF26c0B4956C` |

## Interfaces

Two interfaces define the full API surface:

- **`IWrappedNative`** -- WETH9-compatible base: `deposit`, `withdraw`, standard ERC-20 (`transfer`, `transferFrom`, `approve`, `balanceOf`, `allowance`, `totalSupply`), plus `name`, `symbol`, `decimals`. Events: `Transfer`, `Approval`, `Deposit`, `Withdrawal`.

- **`IWrappedNativeExtended`** -- Extended features: `depositTo`, `withdrawToAccount`, `withdrawSplit`, `permitTransfer`, `doPermittedWithdraw`, `revokeMyOutstandingPermits`, `revokeMyNonce`, `isNonceUsed`, `masterNonces`, `domainSeparatorV4`. Events: `PermitNonceInvalidated`, `MasterNonceInvalidated`.

## Key Behavioral Notes

- **Payable transferFrom deposits credit `from`, not `msg.sender`** -- Integrating protocols must understand that when `transferFrom` is called with `msg.value > 0`, the deposit credits the `from` address. This is also true for `permitTransfer`.

- **Event logging caveats** -- `depositTo` logs the `to` address in the Deposit event (not `msg.sender`). `withdrawToAccount` and `withdrawSplit` log `msg.sender` in Withdrawal events (not the receiver). `permitTransfer` and `doPermittedWithdraw` log the `from` address.

- **Backward compatible with WETH9** -- All WETH9 interfaces are preserved. Existing integrations that use the basic `deposit`/`withdraw`/ERC-20 interface work without modification.

## Reference Files

- `references/api.md` -- Complete function signatures with postconditions for all deposit, withdrawal, ERC-20, and permit functions
- `references/permit-system.md` -- EIP-712 typed data details, type hashes, domain separator, nonce management, convenience fee model
- `references/benchmarks.md` -- Gas savings per function vs WETH9, aggregate L1 savings data

## Related Skills

- **wrapped-native-integrator** -- Generate Solidity contracts that integrate with Wrapped Native
- **apptoken-general** -- Ecosystem overview, cross-protocol integration model
- **lbamm-protocol** -- LBAMM pools can pair tokens against Wrapped Native
- **tokenmaster-protocol** -- TokenMaster pools can use Wrapped Native as a reserve asset
- **payment-processor-protocol** -- Payment Processor can accept Wrapped Native for purchases
