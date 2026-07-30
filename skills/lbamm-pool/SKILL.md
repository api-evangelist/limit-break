---
name: lbamm-pool
description: Generate pool creation code for Dynamic, Fixed, and Single Provider pools
user-invocable: true
disable-model-invocation: true
argument-hint: "[pool type and configuration description]"
---

# LBAMM Pool Creation Generator

Generate complete pool creation code -- Foundry scripts or helper contracts -- for any of the 3 LBAMM pool types.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **LBAMM Core**: `lib/lbamm-core/` must exist. If not: `forge install limitbreakinc/lbamm-core`
3. **Pool type library** (install the one matching the target pool type):
   - Fixed: `forge install limitbreakinc/lbamm-pool-type-fixed`
   - Single Provider: `forge install limitbreakinc/lbamm-pool-type-single-provider`
   - Dynamic: `forge install limitbreakinc/amm-pool-type-dynamic`
4. **Remappings**: `remappings.txt` needs `@limitbreak/lbamm-core/=lib/lbamm-core/` plus the pool type remapping. Append if missing -- do not overwrite existing entries.
   - Fixed: `@limitbreak/lbamm-pool-type-fixed/=lib/lbamm-pool-type-fixed/`
   - Single Provider: `@limitbreak/lbamm-pool-type-single-provider/=lib/lbamm-pool-type-single-provider/`
   - Dynamic: `@limitbreak/amm-pool-type-dynamic/=lib/amm-pool-type-dynamic/`
4. **Local testing**: LBAMM is in audit and not yet on public networks. For local testing, use `lbamm-examples` -- see `lbamm-protocol/references/local-dev.md`. Generated scripts should read addresses from environment variables (`vm.envAddress`) so they work with the local Anvil deployment.

## Instructions

1. **Parse requirements** from `$ARGUMENTS` -- determine pool type and configuration

2. **Resolve ambiguities before generating** -- if $ARGUMENTS doesn't specify token addresses, pool type (Dynamic vs Fixed vs Single Provider) isn't clear from context, or initial price/liquidity details are missing, ask in a single message. If clear, proceed.

3. **Determine the pool type:**
   - **Dynamic**: Concentrated liquidity (Uniswap V3-like) with tick-based positions
   - **Fixed**: Constant exchange rate with height-based liquidity
   - **Single Provider**: Oracle/hook-determined pricing with one LP per pool

4. **Generate complete Foundry script or contract** that:
   - Constructs `PoolCreationDetails` with correct `poolType` address, `fee` (or `55_555` for dynamic fees), `token0`/`token1` (correctly ordered: token0 < token1), `poolHook`, and `poolParams`
   - Encodes pool-type-specific params:
     - Dynamic: `abi.encode(DynamicPoolCreationDetails(tickSpacing, sqrtPriceRatioX96))`
     - Fixed: `abi.encode(FixedPoolCreationDetails(spacing0, spacing1, packedRatio))`
     - Single Provider: `abi.encode(SingleProviderPoolCreationDetails(salt, sqrtPriceRatioX96))`
   - Calls `createPool()` with optional initial liquidity
   - Handles token approvals and native token wrapping

5. **Explain key parameters:**
   - For Dynamic: tick spacing choices, initial sqrtPriceX96 calculation, concentrated liquidity ranges
   - For Fixed: height spacing, constant price mechanics, asymmetric position setup
   - For Single Provider: oracle hook requirements, single LP constraint

6. **Compute the deterministic poolId** and explain its encoding

## Reference Files

- `references/dynamic-pool.md` -- Dynamic pool creation, tick spacing, sqrtPriceX96, liquidity modification
- `references/fixed-pool.md` -- Fixed pool creation, height spacing, constant rate, asymmetric positions
- `references/single-provider-pool.md` -- Single provider creation, oracle hook, single LP mechanics
- Protocol knowledge: `lbamm-protocol`

## Common Mistakes

- Token ordering: `token0` must be less than `token1` (numerically lower address) -- will revert with `LBAMM__CannotPairIdenticalTokens` or produce wrong pool ID
- Fee is in basis points (10000 = 100%). Typical production fees: 5-100 BPS (0.05%-1.0%). A value of `3000` means 30% fee, which is almost never what you want
- Using `fee = 55_555` without a pool hook that implements `getPoolFeeForSwap()` -- swaps will revert
- Pool type addresses must have 6 leading zero bytes to fit in 112 bits of the pool ID

## Dependencies

```
@limitbreak/lbamm-core/src/interfaces/core/ILimitBreakAMMLiquidity.sol
@limitbreak/lbamm-core/src/DataTypes.sol
```

## Related Skills

- **lbamm-pool-hook** -- Required for Single Provider pools; also used for dynamic fees and LP access control
- **lbamm-position-hook** -- Generate per-position hooks for time locks, LP gating, or deposit/withdrawal fees
- **lbamm-test** -- Generate Foundry test suites for pool creation and operations
