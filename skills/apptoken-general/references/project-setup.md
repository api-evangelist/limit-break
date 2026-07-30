# Project Setup

Before generating any Solidity code, ensure the user's project is properly configured. Follow these steps at the start of every generator skill invocation.

## Step 1: Check for Foundry Project

Look for `foundry.toml` in the project root. If it doesn't exist:

```bash
forge init
```

## Step 2: Install Dependencies

Check which libraries are needed based on the skill being used, then verify they're installed in `lib/`.

### Dependency Map

| Protocols | Command | Check Path |
|-----------|---------|------------|
| ERC-20C, ERC-721C, ERC-1155C token bases | `forge install limitbreakinc/creator-token-standards` | `lib/creator-token-standards/` |
| LBAMM core interfaces, hooks, integrators | `forge install limitbreakinc/lbamm-core` | `lib/lbamm-core/` |
| LBAMM standard hook, handlers (permit, CLOB) | `forge install limitbreakinc/lbamm-hooks-and-handlers` | `lib/lbamm-hooks-and-handlers/` |
| LBAMM Fixed pool type | `forge install limitbreakinc/lbamm-pool-type-fixed` | `lib/lbamm-pool-type-fixed/` |
| LBAMM Single Provider pool type | `forge install limitbreakinc/lbamm-pool-type-single-provider` | `lib/lbamm-pool-type-single-provider/` |
| LBAMM Dynamic pool type | `forge install limitbreakinc/amm-pool-type-dynamic` | `lib/amm-pool-type-dynamic/` |
| TokenMaster | `forge install limitbreakinc/tm-tokenmaster` | `lib/tm-tokenmaster/` |
| Transfer Validator | `forge install limitbreakinc/creator-token-transfer-validator` | `lib/creator-token-transfer-validator/` |
| Payment Processor | `forge install limitbreakinc/payment-processor` | `lib/payment-processor/` |
| PermitC (standalone) | `forge install limitbreakinc/PermitC` | `lib/PermitC/` |
| Wrapped Native (WNATIVE) | `forge install limitbreakinc/wrapped-native` | `lib/wrapped-native/` |
| OpenZeppelin (common dependency) | `forge install OpenZeppelin/openzeppelin-contracts` | `lib/openzeppelin-contracts/` |

Only install the libraries needed for the current skill. Forge std is included by `forge init`.

## Step 3: Configure Remappings

Check for `remappings.txt` in the project root. Create or append entries for installed libraries. Only add remappings for libraries that are actually installed.

```
@limitbreak/creator-token-standards/=lib/creator-token-standards/
@limitbreak/lbamm-core/=lib/lbamm-core/
@limitbreak/lbamm-hooks-and-handlers/=lib/lbamm-hooks-and-handlers/
@limitbreak/lbamm-pool-type-fixed/=lib/lbamm-pool-type-fixed/
@limitbreak/lbamm-pool-type-single-provider/=lib/lbamm-pool-type-single-provider/
@limitbreak/amm-pool-type-dynamic/=lib/amm-pool-type-dynamic/
@limitbreak/tm-tokenmaster/=lib/tm-tokenmaster/
@limitbreak/creator-token-transfer-validator/=lib/creator-token-transfer-validator/
@limitbreak/payment-processor/=lib/payment-processor/
@limitbreak/permit-c/=lib/PermitC/
@limitbreak/wrapped-native/=lib/wrapped-native/
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
forge-std/=lib/forge-std/src/
```

**Important:** Do not overwrite existing `remappings.txt` entries. Append any missing remappings.

## Step 4: Verify

After setup, run a quick check:

```bash
forge build
```

If the build fails due to missing imports, check that remappings match the installed library paths.

## Local Development Environment

For local testing against all released Apptoken protocols, use the `apptoken-dev` repository. It starts a local Anvil node and deploys Transfer Validator, PermitC, Trusted Forwarder, Payment Processor, TokenMaster, and Wrapped Native.

### Setup

Install `apptoken-dev` as a git submodule in the project root:

```bash
git submodule add https://github.com/limitbreakinc/apptoken-dev.git
git submodule update --init --recursive
```

### Starting the Local Environment

```bash
./apptoken-dev/local-environment/script/start.sh
```

This starts an Anvil node and deploys all released Apptoken protocols to their deterministic CREATE2 addresses. Generated scripts can use the same well-known addresses listed in `apptoken-general/SKILL.md` (Key Addresses section) -- they are identical on local Anvil and on production chains.

### LBAMM Local Dev

LBAMM is in audit and not yet on public networks. For LBAMM-specific local testing (AMM core, pool types, hooks, handlers), use `lbamm-examples` instead -- see `lbamm-protocol/references/local-dev.md`. The `lbamm-examples` environment deploys everything `apptoken-dev` does, plus the full LBAMM stack.
