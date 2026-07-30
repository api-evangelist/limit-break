# Fixed Pool Reference

Source: `lbamm-pool-type-fixed/src/`

## Pool Creation

```solidity
struct FixedPoolCreationDetails {
    uint8 spacing0;       // Height spacing for token0 positions
    uint8 spacing1;       // Height spacing for token1 positions
    uint256 packedRatio;  // Packed ratio of token0:token1 representing price
}

bytes memory poolParams = abi.encode(
    FixedPoolCreationDetails({
        spacing0: 1,
        spacing1: 1,
        packedRatio: computePackedRatio(1e18, 1e18) // 1:1
    })
);

PoolCreationDetails memory details = PoolCreationDetails({
    poolType: address(fixedPoolType),
    fee: 30,    // 30 BPS = 0.30% LP fee
    token0: tokenA,
    token1: tokenB,
    poolHook: address(0),
    poolParams: poolParams
});
```

## Key Concepts

### Constant Exchange Rate
Fixed pools maintain a constant price. Swaps consume liquidity at the fixed ratio instead of moving the price. This is fundamentally different from dynamic pools where price changes with each swap.

### Height-Based Positions
Liquidity is organized along a "height" dimension:
- Each LP provides liquidity at a range of heights
- As swaps occur, the pool consumes liquidity progressing through heights
- When all liquidity at a height is consumed, the pool moves to the next height

### Height Spacing
Similar to tick spacing in dynamic pools, controls position granularity. Common values: 1 (maximum granularity).

### Asymmetric Positions
LPs can provide:
- **Token0 only**: Single-sided liquidity
- **Token1 only**: Single-sided liquidity
- **In-range**: Combination using `addInRange0`/`addInRange1` flags

## Adding Liquidity

```solidity
struct FixedLiquidityModificationParams {
    uint256 amount0;                  // Max token0 to add / exact to remove
    uint256 amount1;                  // Max token1 to add / exact to remove
    bool addInRange0;                 // Add token0 in-range (converts some token1)
    bool addInRange1;                 // Add token1 in-range (converts some token0)
    uint256 endHeightInsertionHint0;  // Hint for linked list (0 for none)
    uint256 endHeightInsertionHint1;  // Hint for linked list (0 for none)
    uint256 maxStartHeight0;          // Max starting height for token0
    uint256 maxStartHeight1;          // Max starting height for token1
}

bytes memory liquidityPoolParams = abi.encode(
    FixedLiquidityModificationParams({
        amount0: 1000e18,
        amount1: 1000e18,
        addInRange0: false,
        addInRange1: false,
        endHeightInsertionHint0: 0,
        endHeightInsertionHint1: 0,
        maxStartHeight0: type(uint256).max,
        maxStartHeight1: type(uint256).max
    })
);
```

## Removing Liquidity

Fixed pools support two withdrawal modes:

### Partial Withdrawal
Use `FixedLiquidityModificationParams` with specific amounts.

### Full Withdrawal
```solidity
struct FixedLiquidityWithdrawalParams {
    bool withdrawAll;
    bytes params;  // abi.encode(FixedLiquidityWithdrawAllParams) if withdrawAll=true
}

struct FixedLiquidityWithdrawAllParams {
    uint256 minAmount0;
    uint256 minAmount1;
}
```

## Pool State

```solidity
struct FixedPoolStateView {
    uint160 sqrtPriceX96;           // Fixed price
    uint256 packedRatio;            // Token ratio
    uint128 position0ShareOf0;      // Token0 from token0 positions
    uint128 position1ShareOf1;      // Token1 from token1 positions
    uint256 currentHeight0;         // Current height for token0
    uint256 currentHeight1;         // Current height for token1
    uint256 consumedLiquidity0;     // Total consumed token0 liquidity
    uint256 consumedLiquidity1;     // Total consumed token1 liquidity
    uint128 liquidity0;             // Available token0 liquidity
    uint128 liquidity1;             // Available token1 liquidity
    uint128 remainingAtHeight0;     // Remaining at current token0 height
    uint128 remainingAtHeight1;     // Remaining at current token1 height
    uint256 dust0;                  // Rounding dust for token0
    uint256 dust1;                  // Rounding dust for token1
}
```

## Constants (`lbamm-pool-type-fixed/src/Constants.sol`)

```solidity
uint128 constant RATIO_BASE = 10**38;           // Base ratio for fixed price conversion
uint8 constant MAX_HEIGHT_SPACING = 24;          // Max height spacing (avoids excessive jumps)
uint24 constant SPACING_PRECISION_BIT_MASK = 0xFF; // Mask for extracting spacing precision
uint8 constant SPACING_PRECISION_SHIFT_FOR_ZERO = 8; // Bit shift for token0 spacing precision

// Pool ID packing shifts (fixed pool uses separate shifts for each token's spacing)
uint8 constant POOL_ID_SPACING_SHIFT_ZERO = 24;  // Bit shift for token0 height spacing in poolId
uint8 constant POOL_ID_SPACING_SHIFT_ONE = 16;   // Bit shift for token1 height spacing in poolId
```

## Use Cases
- Stablecoin pairs (USDC/USDT) where price should stay exactly 1:1
- Wrapped token pairs (WETH/stETH) at fixed exchange rate
- Token launches with fixed initial pricing
- Bonding curve alternatives with predictable pricing
