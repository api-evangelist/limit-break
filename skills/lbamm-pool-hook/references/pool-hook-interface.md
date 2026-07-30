# Pool Hook Interface Reference

## ILimitBreakAMMPoolHook

Source: `lbamm-core/src/interfaces/hooks/ILimitBreakAMMPoolHook.sol`

```solidity
interface ILimitBreakAMMPoolHook {
    event PoolHookManifestUriUpdated(string uri);

    /// @notice Validates pool creation. Revert to block.
    function validatePoolCreation(
        bytes32 poolId,
        address creator,
        PoolCreationDetails calldata details,
        bytes calldata hookData
    ) external;

    /// @notice Returns the LP fee for a swap when pool uses dynamic fees (fee == 55_555)
    /// @dev poolFeeBPS <= 10_000 for input swaps, < 10_000 for output swaps
    function getPoolFeeForSwap(
        SwapContext calldata context,
        HookPoolFeeParams calldata poolFeeParams,
        bytes calldata hookData
    ) external returns (uint256 poolFeeBPS);

    /// @notice Validates fee collection. Returns hook fees.
    function validatePoolCollectFees(
        LiquidityContext calldata context,
        LiquidityCollectFeesParams calldata liquidityParams,
        uint256 fees0,
        uint256 fees1,
        bytes calldata hookData
    ) external returns (uint256 hookFee0, uint256 hookFee1);

    /// @notice Validates liquidity add. Returns hook fees.
    function validatePoolAddLiquidity(
        LiquidityContext calldata context,
        LiquidityModificationParams calldata liquidityParams,
        uint256 deposit0,
        uint256 deposit1,
        uint256 fees0,
        uint256 fees1,
        bytes calldata hookData
    ) external returns (uint256 hookFee0, uint256 hookFee1);

    /// @notice Validates liquidity removal. Returns hook fees.
    function validatePoolRemoveLiquidity(
        LiquidityContext calldata context,
        LiquidityModificationParams calldata liquidityParams,
        uint256 withdraw0,
        uint256 withdraw1,
        uint256 fees0,
        uint256 fees1,
        bytes calldata hookData
    ) external returns (uint256 hookFee0, uint256 hookFee1);

    /// @notice Returns the manifest URI for the pool hook
    function poolHookManifestUri() external view returns (string memory manifestUri);
}
```

## ISingleProviderPoolHook Extension

Source: `lbamm-pool-type-single-provider/src/interfaces/ISingleProviderPoolHook.sol`

For single provider pools, the pool hook MUST implement this extended interface:

```solidity
interface ISingleProviderPoolHook is ILimitBreakAMMPoolHook {
    struct HookPoolPriceParams {
        bool inputSwap;
        bytes32 poolId;
        address tokenIn;
        address tokenOut;
        uint256 amount;
    }

    /// @notice Returns the price for swap execution (called during each swap)
    function getPoolPriceForSwap(
        SwapContext calldata context,
        HookPoolPriceParams calldata poolPriceParams,
        bytes calldata hookData
    ) external returns (uint160 poolPrice);

    /// @notice Returns the sole LP for this pool
    function getPoolLiquidityProvider(
        bytes32 poolId
    ) external view returns (address provider);
}
```

## Dynamic Fee Sentinel

```solidity
uint16 constant DYNAMIC_POOL_FEE_BPS = 55_555;
```

When a pool is created with `fee = 55_555`, the AMM calls `getPoolFeeForSwap()` on the pool hook for every swap instead of using a fixed LP fee. The pool MUST have a pool hook that implements this function.

## Fee Bounds for `getPoolFeeForSwap()`

- Input-based swaps: `poolFeeBPS <= 10_000` (can equal 100%)
- Output-based swaps: `poolFeeBPS < 10_000` (strictly less than 100%)

The AMM enforces this at `AMMModule.sol:1717`: the revert condition is `(inputSwap && poolFeeBPS > MAX_BPS) || poolFeeBPS >= MAX_BPS`. For input swaps, the fee reverts only when **greater than** 10,000. For output swaps (and the non-input branch), the fee reverts when **greater than or equal to** 10,000.

If the returned fee exceeds these bounds, the AMM reverts.

## Hook Call Ordering in Pool Operations

1. Pool creation: `validatePoolCreation()` called after both token hooks validate
2. Swap: `getPoolFeeForSwap()` called after both token hooks' `beforeSwap()`
3. Add liquidity: `validatePoolAddLiquidity()` called last (after token hooks + position hook)
4. Remove liquidity: `validatePoolRemoveLiquidity()` called last
5. Collect fees: `validatePoolCollectFees()` called last

## HookPoolFeeParams

```solidity
struct HookPoolFeeParams {
    bool inputSwap;     // True = input swap, false = output swap
    uint256 hopIndex;   // Zero-based hop index
    bytes32 poolId;     // Pool identifier
    address tokenIn;    // Input token
    address tokenOut;   // Output token
    uint256 amount;     // Swap amount (for input swaps: amountIn MINUS beforeSwap token hook fees)
}
```
