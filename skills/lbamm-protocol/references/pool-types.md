# LBAMM Pool Types

## Overview

LBAMM supports three pool types, each implementing `ILimitBreakAMMPoolType`. Pool types provide swap math and liquidity accounting but do NOT hold reserves -- the core AMM owns all funds.

## Comparison Matrix

| Feature | Dynamic | Fixed | Single Provider |
|---------|---------|-------|-----------------|
| **Pricing** | Market-driven (tick-based) | Constant exchange rate | Oracle/hook-determined |
| **LPs** | Many, concentrated ranges | Many, height-based | One per pool |
| **Price Movement** | Changes with swaps | Never changes | Set by hook per swap |
| **Pool Hook** | Optional | Optional | **Required** (`ISingleProviderPoolHook`) |
| **Swap Math** | Uniswap V3-like | Consumes liquidity at fixed ratio | Hook sets price per swap |
| **Best For** | Standard DEX trading | Stablecoin swaps, fixed-rate pairs | Market makers, oracle pricing |
| **Complexity** | Medium | Low | High (custom hook needed) |

---

## 1. Dynamic Pool Type (`amm-pool-type-dynamic/`)

Concentrated liquidity pool with tick-based positions, similar to Uniswap V3.

### Pool Creation Params

```solidity
struct DynamicPoolCreationDetails {
    int24 tickSpacing;          // Tick spacing (common values: 1, 10, 60, 200)
    uint160 sqrtPriceRatioX96;  // Initial price as sqrt(price) * 2^96
}
// Encoded as: abi.encode(DynamicPoolCreationDetails)
```

### Liquidity Modification Params

```solidity
struct DynamicLiquidityModificationParams {
    int24 tickLower;          // Lower tick bound
    int24 tickUpper;          // Upper tick bound
    int128 liquidityChange;   // Positive = add, negative = remove
    uint160 snapSqrtPriceX96; // Price to snap to before adding (0 = no snap)
}
// Encoded in LiquidityModificationParams.poolParams as: abi.encode(DynamicLiquidityModificationParams)
```

### Collect Fees Params

```solidity
struct DynamicLiquidityCollectFeesParams {
    int24 tickLower;
    int24 tickUpper;
}
// Encoded in LiquidityCollectFeesParams.poolParams
```

### Pool State

```solidity
struct DynamicPoolState {
    uint256 feeGrowthGlobal0X128;
    uint256 feeGrowthGlobal1X128;
    uint160 sqrtPriceX96;
    int24 tick;
    uint128 liquidity;
}
```

### Key Concepts
- **Ticks**: Price is discretized into ticks. `tickSpacing` determines allowed tick granularity.
- **Concentrated Liquidity**: LPs provide liquidity in `[tickLower, tickUpper)` ranges.
- **sqrtPriceX96**: Price encoded as `sqrt(token1/token0) * 2^96`.
- **Fee Growth**: Global fee growth tracked per token in Q128.128 fixed point.
- **Position Identity**: Derived from provider + liquidityHook + poolId + tickLower + tickUpper.
- **Snap Price**: When `snapSqrtPriceX96` is non-zero, the active tick and all ticks to the snap price must have zero liquidity. Moves pool price before adding liquidity.

### sqrtPriceX96 Calculation

```
sqrtPriceX96 = sqrt(price) * 2^96
```

Where `price = token1_amount / token0_amount` (how many token1 per token0).

For a 1:1 price: `sqrtPriceX96 = 2^96 = 79228162514264337593543950336`

---

## 2. Fixed Pool Type (`lbamm-pool-type-fixed/`)

Constant exchange rate pool with height-based liquidity positions.

### Pool Creation Params

```solidity
struct FixedPoolCreationDetails {
    uint8 spacing0;      // Height spacing for token0 positions
    uint8 spacing1;      // Height spacing for token1 positions
    uint256 packedRatio; // Packed ratio of token0:token1 representing price
}
// Encoded as: abi.encode(FixedPoolCreationDetails)
```

### Liquidity Modification Params

```solidity
struct FixedLiquidityModificationParams {
    uint256 amount0;                  // Max token0 to add / exact to remove
    uint256 amount1;                  // Max token1 to add / exact to remove
    bool addInRange0;                 // Add token0 in-range (consumes token1)
    bool addInRange1;                 // Add token1 in-range (consumes token0)
    uint256 endHeightInsertionHint0;  // Hint for linked list insertion
    uint256 endHeightInsertionHint1;  // Hint for linked list insertion
    uint256 maxStartHeight0;          // Max height for position start
    uint256 maxStartHeight1;          // Max height for position start
}
// Encoded in LiquidityModificationParams.poolParams
```

### Key Concepts
- **Constant Price**: Exchange rate never changes -- swaps consume liquidity at fixed ratio.
- **Height-Based Positions**: Liquidity organized along a "height" dimension. As swaps occur, the pool progresses through heights.
- **Asymmetric Positions**: LPs can provide token0-only, token1-only, or both (in-range).
- **Linked List**: Active heights stored in a doubly linked list for efficient traversal.
- **Spacing**: Height spacing controls position granularity (like tick spacing for dynamic pools).
- **packedRatio**: Encodes the fixed exchange rate between token0 and token1.

---

## 3. Single Provider Pool Type (`lbamm-pool-type-single-provider/`)

Oracle/hook-determined pricing with a single LP per pool.

### Pool Creation Params

```solidity
struct SingleProviderPoolCreationDetails {
    bytes32 salt;               // Unique salt for deterministic pool ID
    uint160 sqrtPriceRatioX96;  // Initial price
}
// Encoded as: abi.encode(SingleProviderPoolCreationDetails)
```

### Liquidity Modification Params

```solidity
struct SingleProviderLiquidityModificationParams {
    uint256 amount0;  // Max token0 to add / exact to remove
    uint256 amount1;  // Max token1 to add / exact to remove
}
// Encoded in LiquidityModificationParams.poolParams
```

### ISingleProviderPoolHook Extension

Single provider pools require a pool hook that implements `ISingleProviderPoolHook`:

```solidity
interface ISingleProviderPoolHook is ILimitBreakAMMPoolHook {
    struct HookPoolPriceParams {
        bool inputSwap;
        bytes32 poolId;
        address tokenIn;
        address tokenOut;
        uint256 amount;
    }

    /// Returns the price for swap execution (called during each swap)
    function getPoolPriceForSwap(
        SwapContext calldata context,
        HookPoolPriceParams calldata poolPriceParams,
        bytes calldata hookData
    ) external returns (uint160 poolPrice);

    /// Returns the sole LP for this pool
    function getPoolLiquidityProvider(bytes32 poolId) external view returns (address provider);
}
```

### Key Concepts
- **Single LP**: Only one liquidity provider per pool. The provider is determined by the pool hook.
- **Oracle Pricing**: Price is determined by the `ISingleProviderPoolHook.getPoolPriceForSwap()` hook on every swap -- not by pool reserves.
- **Mandatory Pool Hook**: Single provider pools MUST have a pool hook implementing `ISingleProviderPoolHook`.
- **Use Cases**: Market-maker pools, oracle-priced pools, protocol-controlled liquidity.
