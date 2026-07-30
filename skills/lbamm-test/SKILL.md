---
name: lbamm-test
description: Generate Foundry test suites following LBAMM test patterns
user-invocable: true
disable-model-invocation: true
argument-hint: "[contract to test or test description]"
---

# LBAMM Foundry Test Generator

Generate Foundry test suites following LBAMM test patterns and conventions.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **LBAMM Core**: `lib/lbamm-core/` must exist. If not: `forge install limitbreakinc/lbamm-core`
3. **Remappings**: `remappings.txt` needs `@limitbreak/lbamm-core/=lib/lbamm-core/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: LBAMM is in audit and not yet on public networks. For local testing, use `lbamm-examples` -- see `lbamm-protocol/references/local-dev.md`. Generated scripts should read addresses from environment variables (`vm.envAddress`) so they work with the local Anvil deployment.

## Instructions

1. **Read the contract under test** (if referenced in `$ARGUMENTS`)

2. **Resolve ambiguities before generating** -- if $ARGUMENTS doesn't reference a specific contract or file to test, or it's unclear which aspects to cover (happy paths, reverts, fees, access control), ask in a single message. If clear, proceed.

3. **Select the correct base test class:**
   - `LBAMMCoreBaseTest` -- For testing core AMM operations, token settings, fee management
   - `LBAMMCorePoolBaseTest` -- For testing pool-specific operations (swaps, liquidity, fees) -- extends `LBAMMCoreBaseTest` with pool operation helpers
   - For hooks-and-handlers tests: extend one of the above or create a custom base

4. **Generate `setUp()`** with:
   - Deterministic CREATE2 deployment for pool types (using salts from `TestConstants.t.sol`)
   - Mock token deployment (`ERC20Mock` at well-known addresses via `vm.etch`)
   - Role server and transfer validator setup
   - Pool creation with appropriate parameters
   - Token approvals and initial balances

5. **Generate test cases** for:
   - Happy paths (successful operations)
   - Edge cases (zero amounts, max values, boundary conditions)
   - Revert conditions (with `vm.expectRevert(SelectorName.selector)`)
   - Fee verification (protocol fees, hook fees, exchange fees)

6. **Use established helpers:**
   - `_executeSingleSwap()`, `_executeMultiSwap()`, `_executeDirectSwap()`
   - `_executeAddLiquidity()`, `_executeRemoveLiquidity()`, `_executeCollectFees()`
   - `_createPool()`, `_createPoolWithAddLiquidity()`
   - `_mintAndApprove()`, `_dealDepositApproveNative()`
   - `_setTokenSettings()`, `_setProtocolFees()`
   - `_packSettings()` for building token flag bitmasks

## Reference Files

- `references/test-infrastructure.md` -- Base test classes, helpers, mocks, CREATE2 setup, TestConstants
- `references/test-patterns.md` -- Example test suites per feature area
- Protocol knowledge: `lbamm-protocol`
