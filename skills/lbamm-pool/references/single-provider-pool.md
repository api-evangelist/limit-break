# Single Provider Pool Reference

Source: `lbamm-pool-type-single-provider/src/`

## Pool Creation

```solidity
struct SingleProviderPoolCreationDetails {
    bytes32 salt;               // Unique salt for deterministic pool ID
    uint160 sqrtPriceRatioX96;  // Initial price
}

bytes memory poolParams = abi.encode(
    SingleProviderPoolCreationDetails({
        salt: keccak256("my-pool-1"),
        sqrtPriceRatioX96: 79228162514264337593543950336 // 1:1
    })
);

PoolCreationDetails memory details = PoolCreationDetails({
    poolType: address(singleProviderPoolType),
    fee: 30,       // 30 BPS = 0.30% (or 55_555 for dynamic fees)
    token0: tokenA,
    token1: tokenB,
    poolHook: address(oracleHook),  // REQUIRED: must implement ISingleProviderPoolHook
    poolParams: poolParams
});
```

## Key Requirements

### Mandatory Pool Hook
Single provider pools MUST have a pool hook implementing `ISingleProviderPoolHook`:

```solidity
interface ISingleProviderPoolHook is ILimitBreakAMMPoolHook {
    struct HookPoolPriceParams {
        bool inputSwap;
        bytes32 poolId;
        address tokenIn;
        address tokenOut;
        uint256 amount;
    }

    // Called during every swap to get the execution price
    function getPoolPriceForSwap(
        SwapContext calldata context,
        HookPoolPriceParams calldata poolPriceParams,
        bytes calldata hookData
    ) external returns (uint160 poolPrice);

    // Returns the sole LP for this pool
    function getPoolLiquidityProvider(
        bytes32 poolId
    ) external view returns (address provider);
}
```

### Single LP Constraint
Only one liquidity provider per pool:
- The provider is determined by `ISingleProviderPoolHook.getPoolLiquidityProvider()`
- Typically set to the pool creator during `validatePoolCreation()`
- Only the designated provider can add/remove liquidity

### Oracle-Based Pricing
Price is set by the hook on every swap via `getPoolPriceForSwap()`:
- Returns `sqrtPriceX96` format
- Price can change between swaps (dynamic market making)
- The pool type uses this price instead of reserve-based pricing

## Adding Liquidity

```solidity
struct SingleProviderLiquidityModificationParams {
    uint256 amount0;  // Max token0 to add / exact to remove
    uint256 amount1;  // Max token1 to add / exact to remove
}

bytes memory liquidityPoolParams = abi.encode(
    SingleProviderLiquidityModificationParams({
        amount0: 1000e18,
        amount1: 1000e6
    })
);
```

## Pool State

```solidity
struct SingleProviderPoolState {
    uint160 lastSqrtPriceX96;  // Last recorded price
}
```

## Use Cases
- **Market Maker Pools**: Professional MMs set prices via oracle hook
- **Oracle-Priced Pools**: Use Chainlink/Pyth/etc. for real-time pricing
- **Protocol-Controlled Liquidity**: DAOs managing their own trading pools
- **RFQ-Like Pools**: Combining with custom hooks for request-for-quote flows

## Complete Example: Single Provider Pool with Oracle Hook

```solidity
// 1. Deploy the oracle hook
OraclePricingHook hook = new OraclePricingHook(chainlinkOracle);

// 2. Create the pool (hook must be ISingleProviderPoolHook)
PoolCreationDetails memory details = PoolCreationDetails({
    poolType: address(singleProviderPoolType),
    fee: 55_555,  // Dynamic fee via hook
    token0: weth,
    token1: usdc,
    poolHook: address(hook),
    poolParams: abi.encode(SingleProviderPoolCreationDetails({
        salt: keccak256("weth-usdc-oracle"),
        sqrtPriceRatioX96: initialPrice
    }))
});

(bytes32 poolId,,) = amm.createPool(details, "", "", "", "");

// 3. Provider adds liquidity
IERC20(weth).approve(address(amm), 100e18);
IERC20(usdc).approve(address(amm), 300_000e6);

LiquidityModificationParams memory liqParams = LiquidityModificationParams({
    liquidityHook: address(0),
    poolId: poolId,
    minLiquidityAmount0: 0,
    minLiquidityAmount1: 0,
    maxLiquidityAmount0: type(uint256).max,
    maxLiquidityAmount1: type(uint256).max,
    maxHookFee0: 0,
    maxHookFee1: 0,
    poolParams: abi.encode(SingleProviderLiquidityModificationParams({
        amount0: 100e18,
        amount1: 300_000e6
    }))
});

amm.addLiquidity(liqParams, LiquidityHooksExtraData("", "", "", ""));
```
