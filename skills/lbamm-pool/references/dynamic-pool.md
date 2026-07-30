# Dynamic Pool Reference

Source: `amm-pool-type-dynamic/src/`

## Pool Creation

```solidity
struct DynamicPoolCreationDetails {
    int24 tickSpacing;          // Tick spacing (1, 10, 60, 200 are common)
    uint160 sqrtPriceRatioX96;  // Initial price as sqrt(price) * 2^96
}

// Encode as poolParams:
bytes memory poolParams = abi.encode(
    DynamicPoolCreationDetails({
        tickSpacing: 60,
        sqrtPriceRatioX96: 79228162514264337593543950336 // 1:1 price
    })
);

PoolCreationDetails memory details = PoolCreationDetails({
    poolType: address(dynamicPoolType),
    fee: 30,    // 30 BPS = 0.30% LP fee (or 55_555 for dynamic)
    token0: tokenA,  // Must be < token1
    token1: tokenB,
    poolHook: address(0),
    poolParams: poolParams
});
```

## sqrtPriceX96 Calculation

```
sqrtPriceX96 = sqrt(price) * 2^96
```

Where `price = token1_amount / token0_amount` (how many token1 per 1 token0).

Common values:
- 1:1 price: `79228162514264337593543950336` (2^96)
- For USDC/ETH at $3000 (6 vs 18 decimals): `sqrt(3000 * 1e6 / 1e18) * 2^96`

Helper formula:
```solidity
function calculateSqrtPriceX96(uint256 amount1, uint256 amount0) internal pure returns (uint160) {
    return uint160(sqrt(amount1 * (2**192) / amount0));
}
```

## Tick Spacing

| Spacing | Use Case | Fee Range |
|---------|----------|-----------|
| 1 | Stablecoin pairs | Very tight |
| 10 | Common pairs | Low to medium |
| 60 | Standard volatile | Medium |
| 200 | High volatility | High |

Tick spacing constrains position bounds: `tickLower` and `tickUpper` must be multiples of `tickSpacing`.

## Adding Liquidity

```solidity
struct DynamicLiquidityModificationParams {
    int24 tickLower;          // Must be multiple of tickSpacing
    int24 tickUpper;          // Must be multiple of tickSpacing, > tickLower
    int128 liquidityChange;   // Positive = add, negative = remove
    uint160 snapSqrtPriceX96; // 0 = no snap; non-zero = move price first
}

// Encode as poolParams in LiquidityModificationParams:
bytes memory liquidityPoolParams = abi.encode(
    DynamicLiquidityModificationParams({
        tickLower: -887220,   // Near min tick
        tickUpper: 887220,    // Near max tick
        liquidityChange: int128(1e18),
        snapSqrtPriceX96: 0
    })
);
```

## Collecting Fees

```solidity
struct DynamicLiquidityCollectFeesParams {
    int24 tickLower;
    int24 tickUpper;
}
```

## Position Identity

For dynamic pools, position is derived from:
```
positionId = hash(basePositionId, tickLower, tickUpper)
```
Where `basePositionId = keccak256(provider, liquidityHook, poolId)`.

## Tick Math Constants

```solidity
int24 constant MIN_TICK = -887_272;
int24 constant MAX_TICK = 887_272;
uint160 constant MIN_SQRT_RATIO = 4_295_128_739;
uint160 constant MAX_SQRT_RATIO = 1_461_446_703_485_210_103_287_273_052_203_988_822_378_723_970_342;
int24 constant MIN_TICK_SPACING = 1;
int24 constant MAX_TICK_SPACING = 16_384;
```

## Complete Pool Creation Example (Foundry Script)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {Script} from "forge-std/Script.sol";
import {ILimitBreakAMMLiquidity} from "@limitbreak/lbamm-core/src/interfaces/core/ILimitBreakAMMLiquidity.sol";
import {PoolCreationDetails, LiquidityModificationParams, LiquidityHooksExtraData} from "@limitbreak/lbamm-core/src/DataTypes.sol";
import {DynamicPoolCreationDetails, DynamicLiquidityModificationParams} from "amm-pool-type-dynamic/src/DataTypes.sol";
import {IERC20} from "forge-std/interfaces/IERC20.sol";

contract CreateDynamicPool is Script {
    function run() external {
        vm.startBroadcast();

        address amm = 0x...; // LBAMM address
        address dynamicPoolType = 0x...; // Dynamic pool type address
        address tokenA = 0x...; // Ensure tokenA < tokenB
        address tokenB = 0x...;

        // 1. Create pool
        PoolCreationDetails memory details = PoolCreationDetails({
            poolType: dynamicPoolType,
            fee: 30,   // 30 BPS = 0.30%
            token0: tokenA,
            token1: tokenB,
            poolHook: address(0),
            poolParams: abi.encode(DynamicPoolCreationDetails({
                tickSpacing: 60,
                sqrtPriceRatioX96: 79228162514264337593543950336
            }))
        });

        (bytes32 poolId,,) = ILimitBreakAMMLiquidity(amm).createPool(
            details, "", "", "", ""
        );

        // 2. Add initial liquidity (optional -- can also pass as liquidityData in createPool)
        IERC20(tokenA).approve(amm, type(uint256).max);
        IERC20(tokenB).approve(amm, type(uint256).max);

        LiquidityModificationParams memory liqParams = LiquidityModificationParams({
            liquidityHook: address(0),
            poolId: poolId,
            minLiquidityAmount0: 0,
            minLiquidityAmount1: 0,
            maxLiquidityAmount0: type(uint256).max,
            maxLiquidityAmount1: type(uint256).max,
            maxHookFee0: 0,
            maxHookFee1: 0,
            poolParams: abi.encode(DynamicLiquidityModificationParams({
                tickLower: -887220,
                tickUpper: 887220,
                liquidityChange: int128(1e18),
                snapSqrtPriceX96: 0
            }))
        });

        LiquidityHooksExtraData memory hooksData;
        ILimitBreakAMMLiquidity(amm).addLiquidity(liqParams, hooksData);

        vm.stopBroadcast();
    }
}
```
