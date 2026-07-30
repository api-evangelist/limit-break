# LBAMM Liquidity Interface Reference

## createPool

```solidity
function createPool(
    PoolCreationDetails memory details,
    bytes calldata token0HookData,     // Data for token0's hook
    bytes calldata token1HookData,     // Data for token1's hook
    bytes calldata poolHookData,       // Data for pool hook
    bytes calldata liquidityData       // Optional: addLiquidity calldata
) external payable returns (bytes32 poolId, uint256 deposit0, uint256 deposit1);
```

### Pool Creation Details

```solidity
PoolCreationDetails memory details = PoolCreationDetails({
    poolType: address(poolTypeContract),
    fee: 30,                 // 30 BPS = 0.30% LP fee (or 55_555 for dynamic)
    token0: tokenA,          // MUST be < token1
    token1: tokenB,
    poolHook: address(0),    // Optional pool hook
    poolParams: poolTypeSpecificParams  // abi.encode(pool-type-specific struct)
});
```

### Token Ordering
Tokens MUST be ordered: `token0 < token1`. Helper:
```solidity
(address token0, address token1) = tokenA < tokenB
    ? (tokenA, tokenB)
    : (tokenB, tokenA);
```

### Atomic Create + Add Liquidity
Pass `liquidityData` as the ABI-encoded calldata for `addLiquidity()`:
```solidity
bytes memory liquidityData = abi.encodeCall(
    ILimitBreakAMMLiquidity.addLiquidity,
    (liquidityParams, liquidityHooksExtraData)
);
```

## getPoolState

```solidity
function getPoolState(bytes32 poolId) external view returns (PoolState memory state);
```

Returns the complete pool state for a given pool ID:

```solidity
struct PoolState {
    address token0;
    address token1;
    address poolHook;
    uint128 reserve0;
    uint128 reserve1;
    uint128 feeBalance0;
    uint128 feeBalance1;
}
```

Use this to query pool reserves, fee balances, and associated hook/token addresses.

## addLiquidity

```solidity
function addLiquidity(
    LiquidityModificationParams calldata liquidityParams,
    LiquidityHooksExtraData calldata liquidityHooksExtraData
) external payable returns (uint256 deposit0, uint256 deposit1, uint256 fees0, uint256 fees1);
```

### LiquidityModificationParams

```solidity
LiquidityModificationParams memory params = LiquidityModificationParams({
    liquidityHook: address(0),         // Position hook (zero = no hook)
    poolId: poolId,
    minLiquidityAmount0: 0,            // Min deposit for token0
    minLiquidityAmount1: 0,            // Min deposit for token1
    maxLiquidityAmount0: type(uint256).max,  // Max deposit for token0
    maxLiquidityAmount1: type(uint256).max,  // Max deposit for token1
    maxHookFee0: 0,                    // Max hook fee cap for token0
    maxHookFee1: 0,                    // Max hook fee cap for token1
    poolParams: poolTypeSpecificParams // Pool-type-specific encoded params
});
```

### Pool-Type-Specific Params

**Dynamic Pool:**
```solidity
bytes memory poolParams = abi.encode(DynamicLiquidityModificationParams({
    tickLower: -887220,
    tickUpper: 887220,
    liquidityChange: int128(1e18),
    snapSqrtPriceX96: 0
}));
```

**Fixed Pool:**
```solidity
bytes memory poolParams = abi.encode(FixedLiquidityModificationParams({
    amount0: 1000e18,
    amount1: 1000e18,
    addInRange0: false,
    addInRange1: false,
    endHeightInsertionHint0: 0,
    endHeightInsertionHint1: 0,
    maxStartHeight0: type(uint256).max,
    maxStartHeight1: type(uint256).max
}));
```

**Single Provider Pool:**
```solidity
bytes memory poolParams = abi.encode(SingleProviderLiquidityModificationParams({
    amount0: 100e18,
    amount1: 300_000e6
}));
```

## removeLiquidity

```solidity
function removeLiquidity(
    LiquidityModificationParams calldata liquidityParams,
    LiquidityHooksExtraData calldata liquidityHooksExtraData
) external returns (uint256 withdraw0, uint256 withdraw1, uint256 fees0, uint256 fees1);
```

For dynamic pools, use negative `liquidityChange`:
```solidity
abi.encode(DynamicLiquidityModificationParams({
    tickLower: -887220,
    tickUpper: 887220,
    liquidityChange: -int128(1e18),  // Negative = remove
    snapSqrtPriceX96: 0
}));
```

## collectFees

```solidity
function collectFees(
    LiquidityCollectFeesParams calldata liquidityParams,
    LiquidityHooksExtraData calldata liquidityHooksExtraData
) external returns (uint256 fees0, uint256 fees1);
```

```solidity
LiquidityCollectFeesParams memory params = LiquidityCollectFeesParams({
    liquidityHook: address(0),
    poolId: poolId,
    maxHookFee0: 0,
    maxHookFee1: 0,
    poolParams: poolTypeSpecificParams  // e.g., DynamicLiquidityCollectFeesParams
});
```

## LiquidityHooksExtraData

```solidity
LiquidityHooksExtraData memory hookData = LiquidityHooksExtraData({
    token0Hook: "",       // Data for token0's hook
    token1Hook: "",       // Data for token1's hook
    liquidityHook: "",    // Data for position hook
    poolHook: ""          // Data for pool hook
});
```

## Token Approvals

Before calling addLiquidity/createPool, approve the AMM for both tokens:
```solidity
IERC20(token0).approve(address(amm), amount0);
IERC20(token1).approve(address(amm), amount1);
```

For native token (ETH), send as `msg.value` instead of approving.
