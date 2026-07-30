---
name: tokenmaster-deployer
description: Generate TokenMaster token deployment scripts -- pool type selection, parameter construction, deterministic addressing
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of token requirements - pool type, paired token, pricing, fees]"
---

# TokenMaster Deployer

Generate deployment scripts for TokenMaster tokens.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **TokenMaster**: `lib/tm-tokenmaster/` must exist. If not: `forge install limitbreakinc/tm-tokenmaster`
3. **Remappings**: `remappings.txt` needs `@limitbreak/tm-tokenmaster/=lib/tm-tokenmaster/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: For local development, use `apptoken-dev` to bootstrap an Anvil node with all Apptoken protocols at their deterministic addresses -- see `apptoken-general/references/project-setup.md`.

## Instructions

1. Parse requirements from $ARGUMENTS
2. **Resolve ambiguities before generating** -- if $ARGUMENTS doesn't specify pool type (Standard vs Stable vs Promotional) and it's not obvious from context, or mentions a paired token without an address, ask in a single message. Also clarify fee percentages, creator shares, or infrastructure fee caps if they'd significantly change the deployment. If clear, proceed.
3. Determine pool type: Standard (flexible tokenomics), Stable (fixed price), Promotional (no market value)
4. Construct DeploymentParameters + PoolDeploymentParameters with encoded pool-specific initialization args
5. Compute deterministic address via `factory.computeDeploymentAddress()`
6. Generate Foundry deployment script calling `router.deployToken()`
7. Post-deployment: configure pool settings, set order signers, configure trusted channels if needed

## Reference Files

- `references/deployment-workflow.md` -- Step-by-step deployment with code examples
- `references/pool-configuration.md` -- Pool-specific parameter construction
- Protocol knowledge: `tokenmaster-protocol`

## Common Mistakes

- Forgetting to whitelist the TokenMaster Router in the Transfer Validator -- tokens won't be mintable/burnable
- Using `address(0)` for `pairedToken` without sending `msg.value` -- native token pairing requires ETH deposit
- `DEPLOYMENT_TYPEHASH` only covers 5 fields (not `poolParams` or `maxInfrastructureFeeBPS`) -- don't assume the signer covers pool configuration
- StablePool does NOT support `transferCreatorShareToMarket` -- it reverts with `TokenMasterERC20__OperationNotSupportedByPool()`
- `setTokenAllowedTrustedChannel` exists on the Router but is NOT in `ITokenMasterRouter.sol` -- use the concrete type for ABI encoding

## When to Use This vs Other Skills

- **This skill (tokenmaster-deployer)**: Deploy a new ERC-20C token with a TokenMaster pool. Start here for backed/priced tokens.
- **transfer-validator-config**: Configure which operators can move the token. Do this *before or after* deployment -- the token needs LBAMM and TokenMaster whitelisted.
- **lbamm-pool**: Create an LBAMM trading pool for the token. Do this *after* TokenMaster deployment if you want both backing and AMM trading.

## Related Skills

- **tokenmaster-integrator** -- Generate buy/sell/spend transaction code after deployment
- **tokenmaster-hook** -- Generate hook contracts for advanced order post-execution logic
