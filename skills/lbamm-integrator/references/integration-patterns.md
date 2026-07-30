# LBAMM Integration Patterns

## Pattern 1: Simple Swap Router

A basic router that wraps LBAMM swap calls with user-friendly interfaces.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {ILimitBreakAMMSwap} from "@limitbreak/lbamm-core/src/interfaces/core/ILimitBreakAMMSwap.sol";
import {SwapOrder, BPSFeeWithRecipient, FlatFeeWithRecipient, SwapHooksExtraData} from "@limitbreak/lbamm-core/src/DataTypes.sol";
import {IERC20} from "forge-std/interfaces/IERC20.sol";

contract LBAMMRouter {
    ILimitBreakAMMSwap public immutable amm;

    constructor(address amm_) {
        amm = ILimitBreakAMMSwap(amm_);
    }

    /// @notice Swap by input -- specify input amount, receive output
    function swapByInput(
        address tokenIn,
        address tokenOut,
        bytes32 poolId,
        uint256 amountIn,
        uint256 minAmountOut,
        address recipient,
        uint256 deadline
    ) external returns (uint256 amountOut) {
        // Transfer tokens from user to router
        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        IERC20(tokenIn).approve(address(amm), amountIn);

        SwapOrder memory order = SwapOrder({
            deadline: deadline,
            recipient: recipient,
            amountSpecified: int256(amountIn),
            minAmountSpecified: 0,
            limitAmount: minAmountOut,
            tokenIn: tokenIn,
            tokenOut: tokenOut
        });

        (, amountOut) = amm.singleSwap(
            order,
            poolId,
            BPSFeeWithRecipient({recipient: address(0), BPS: 0}),
            FlatFeeWithRecipient({recipient: address(0), amount: 0}),
            SwapHooksExtraData({tokenInHook: "", tokenOutHook: "", poolHook: "", poolType: ""}),
            ""
        );
    }

    /// @notice Swap by output -- specify output amount, provide input
    function swapByOutput(
        address tokenIn,
        address tokenOut,
        bytes32 poolId,
        uint256 amountOut,
        uint256 maxAmountIn,
        address recipient,
        uint256 deadline
    ) external returns (uint256 amountIn) {
        // Transfer max input from user
        IERC20(tokenIn).transferFrom(msg.sender, address(this), maxAmountIn);
        IERC20(tokenIn).approve(address(amm), maxAmountIn);

        SwapOrder memory order = SwapOrder({
            deadline: deadline,
            recipient: recipient,
            amountSpecified: -int256(amountOut),  // Negative = output swap
            minAmountSpecified: 0,
            limitAmount: maxAmountIn,
            tokenIn: tokenIn,
            tokenOut: tokenOut
        });

        (amountIn,) = amm.singleSwap(
            order, poolId,
            BPSFeeWithRecipient({recipient: address(0), BPS: 0}),
            FlatFeeWithRecipient({recipient: address(0), amount: 0}),
            SwapHooksExtraData({tokenInHook: "", tokenOutHook: "", poolHook: "", poolType: ""}),
            ""
        );

        // Refund excess input
        uint256 remaining = maxAmountIn - amountIn;
        if (remaining > 0) {
            IERC20(tokenIn).transfer(msg.sender, remaining);
        }
    }
}
```

## Pattern 2: Multi-Hop Router with Exchange Fees

Router that supports multi-hop swaps and charges exchange fees.

```solidity
contract MultiHopRouter {
    ILimitBreakAMMSwap public immutable amm;
    address public feeRecipient;
    uint256 public exchangeFeeBPS = 25; // 0.25% exchange fee

    constructor(address amm_, address feeRecipient_) {
        amm = ILimitBreakAMMSwap(amm_);
        feeRecipient = feeRecipient_;
    }

    function multiHopSwap(
        address tokenIn,
        address tokenOut,
        bytes32[] calldata poolIds,
        uint256 amountIn,
        uint256 minAmountOut,
        address recipient,
        uint256 deadline
    ) external returns (uint256 actualAmountIn, uint256 amountOut) {
        IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
        IERC20(tokenIn).approve(address(amm), amountIn);

        SwapOrder memory order = SwapOrder({
            deadline: deadline,
            recipient: recipient,
            amountSpecified: int256(amountIn),
            minAmountSpecified: 0,
            limitAmount: minAmountOut,
            tokenIn: tokenIn,
            tokenOut: tokenOut
        });

        // Build hook data array (one per hop)
        SwapHooksExtraData[] memory hookDatas = new SwapHooksExtraData[](poolIds.length);
        for (uint256 i; i < poolIds.length; i++) {
            hookDatas[i] = SwapHooksExtraData({
                tokenInHook: "", tokenOutHook: "", poolHook: "", poolType: ""
            });
        }

        (actualAmountIn, amountOut) = amm.multiSwap(
            order,
            poolIds,
            BPSFeeWithRecipient({recipient: feeRecipient, BPS: exchangeFeeBPS}),
            FlatFeeWithRecipient({recipient: address(0), amount: 0}),
            hookDatas,
            ""
        );
    }
}
```

## Pattern 3: Liquidity Manager

Contract that manages pool creation and liquidity operations.

```solidity
import {ILimitBreakAMMLiquidity} from "@limitbreak/lbamm-core/src/interfaces/core/ILimitBreakAMMLiquidity.sol";

contract LiquidityManager {
    ILimitBreakAMMLiquidity public immutable amm;

    constructor(address amm_) {
        amm = ILimitBreakAMMLiquidity(amm_);
    }

    function createDynamicPool(
        address tokenA,
        address tokenB,
        address poolType,
        int24 tickSpacing,
        uint16 feeBPS,
        uint160 sqrtPriceX96
    ) external returns (bytes32 poolId) {
        // Ensure correct token ordering
        (address token0, address token1) = tokenA < tokenB
            ? (tokenA, tokenB) : (tokenB, tokenA);

        PoolCreationDetails memory details = PoolCreationDetails({
            poolType: poolType,
            fee: feeBPS,
            token0: token0,
            token1: token1,
            poolHook: address(0),
            poolParams: abi.encode(DynamicPoolCreationDetails({
                tickSpacing: tickSpacing,
                sqrtPriceRatioX96: sqrtPriceX96
            }))
        });

        (poolId,,) = amm.createPool(details, "", "", "", "");
    }

    function addDynamicLiquidity(
        bytes32 poolId,
        int24 tickLower,
        int24 tickUpper,
        int128 liquidityAmount,
        uint256 maxAmount0,
        uint256 maxAmount1,
        address token0,
        address token1
    ) external returns (uint256 deposit0, uint256 deposit1) {
        IERC20(token0).transferFrom(msg.sender, address(this), maxAmount0);
        IERC20(token1).transferFrom(msg.sender, address(this), maxAmount1);
        IERC20(token0).approve(address(amm), maxAmount0);
        IERC20(token1).approve(address(amm), maxAmount1);

        LiquidityModificationParams memory params = LiquidityModificationParams({
            liquidityHook: address(0),
            poolId: poolId,
            minLiquidityAmount0: 0,
            minLiquidityAmount1: 0,
            maxLiquidityAmount0: maxAmount0,
            maxLiquidityAmount1: maxAmount1,
            maxHookFee0: 0,
            maxHookFee1: 0,
            poolParams: abi.encode(DynamicLiquidityModificationParams({
                tickLower: tickLower,
                tickUpper: tickUpper,
                liquidityChange: liquidityAmount,
                snapSqrtPriceX96: 0
            }))
        });

        LiquidityHooksExtraData memory hookData;
        (deposit0, deposit1,,) = amm.addLiquidity(params, hookData);

        // Refund unused tokens
        uint256 refund0 = maxAmount0 - deposit0;
        uint256 refund1 = maxAmount1 - deposit1;
        if (refund0 > 0) IERC20(token0).transfer(msg.sender, refund0);
        if (refund1 > 0) IERC20(token1).transfer(msg.sender, refund1);
    }
}
```

## Querying Pool State

Use `getPoolState` to read pool reserves, fee balances, and associated addresses before operations:

```solidity
PoolState memory state = ILimitBreakAMMLiquidity(amm).getPoolState(poolId);
// state.token0, state.token1 -- pool tokens
// state.reserve0, state.reserve1 -- current reserves
// state.feeBalance0, state.feeBalance1 -- unclaimed LP fees
// state.poolHook -- pool hook address
```

## Integration Checklist

1. **Token approval**: Approve AMM for input tokens before any operation
2. **Token ordering**: `token0 < token1` for pool creation
3. **Deadline**: Set reasonable deadlines; `block.timestamp + N` for on-chain
4. **Executor identity**: Your contract is the executor -- hooks see `msg.sender = router`
5. **Refunds**: For output swaps, refund excess input tokens to the user
6. **Native token**: Use `msg.value` for ETH input; AMM wraps automatically
7. **Fee recipient validation**: Exchange fee and fee-on-top recipients can't be `address(0)` if amounts are non-zero
8. **Reentrancy**: Be cautious about callbacks during hook execution
9. **Gas estimation**: Multi-hop swaps with many hooks can be gas-intensive
10. **Hook data**: Pass empty bytes if tokens don't have hooks configured
