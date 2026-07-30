# LBAMM Test Patterns

## Table of Contents

- [Pattern 1: Dynamic Pool Swap Test](#pattern-1-dynamic-pool-swap-test)
- [Pattern 2: Token Hook Test](#pattern-2-token-hook-test)
- [Pattern 3: Pool Hook Test with Dynamic Fees](#pattern-3-pool-hook-test-with-dynamic-fees)
- [Pattern 4: Multi-Hop Swap with Fee Verification](#pattern-4-multi-hop-swap-with-fee-verification)
- [Pattern 5: Liquidity Hook Test](#pattern-5-liquidity-hook-test)
- [Test Naming Convention](#test-naming-convention)
- [Common Assertions](#common-assertions)
- [Security Test Checklist](#security-test-checklist)

## Pattern 1: Dynamic Pool Swap Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {LBAMMCorePoolBaseTest} from "@limitbreak/lbamm-core/test/LBAMMCorePoolBase.t.sol";
import {DynamicPoolType} from "amm-pool-type-dynamic/src/DynamicPoolType.sol";

contract DynamicPoolSwapTest is LBAMMCorePoolBaseTest {
    DynamicPoolType public dynamicPoolType;
    bytes32 public poolId;

    function setUp() public override {
        super.setUp();

        // Deploy pool type
        changePrank(KEYLESS_DEPLOYER);
        dynamicPoolType = new DynamicPoolType{salt: keccak256("DYNAMIC_POOL_TYPE")}();
        changePrank(address(this));

        // Create pool: WETH/USDC at ~3000 USDC per WETH
        uint160 sqrtPrice = _calculatePriceLimit(3000e6, 1e18);
        PoolCreationDetails memory details = PoolCreationDetails({
            poolType: address(dynamicPoolType),
            fee: 30,   // 30 BPS = 0.30% LP fee
            token0: address(usdc),  // USDC_ADDRESS < WETH_ADDRESS numerically
            token1: address(weth),
            poolHook: address(0),
            poolParams: abi.encode(DynamicPoolCreationDetails({
                tickSpacing: 60,
                sqrtPriceRatioX96: sqrtPrice
            }))
        });

        poolId = _createPool(details, "", "", "", bytes4(0));

        // Add liquidity
        _mintAndApprove(address(usdc), alice, address(amm), 1_000_000e6);
        _mintAndApprove(address(weth), alice, address(amm), 1000e18);

        changePrank(alice);
        _executeAddLiquidity(
            LiquidityModificationParams({
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
            }),
            _emptyLiquidityHooksExtraData(),
            bytes4(0)
        );
    }

    function test_singleSwap_inputSwap_success() public {
        _mintAndApprove(address(weth), bob, address(amm), 1e18);
        changePrank(bob);

        SwapOrder memory order = SwapOrder({
            deadline: block.timestamp + 300,
            recipient: bob,
            amountSpecified: int256(1e18),
            minAmountSpecified: 0,
            limitAmount: 0,
            tokenIn: address(weth),
            tokenOut: address(usdc)
        });

        (uint256 amountIn, uint256 amountOut) = _executeSingleSwap(
            order,
            poolId,
            BPSFeeWithRecipient({recipient: address(0), BPS: 0}),
            FlatFeeWithRecipient({recipient: address(0), amount: 0}),
            _emptySwapHooksExtraData(),
            "",
            bytes4(0)
        );

        assertGt(amountIn, 0, "Should consume input");
        assertGt(amountOut, 0, "Should produce output");
    }

    function test_singleSwap_revert_expiredDeadline() public {
        _mintAndApprove(address(weth), bob, address(amm), 1e18);
        changePrank(bob);

        SwapOrder memory order = SwapOrder({
            deadline: block.timestamp - 1,  // Expired
            recipient: bob,
            amountSpecified: int256(1e18),
            minAmountSpecified: 0,
            limitAmount: 0,
            tokenIn: address(weth),
            tokenOut: address(usdc)
        });

        _executeSingleSwap(
            order, poolId,
            BPSFeeWithRecipient({recipient: address(0), BPS: 0}),
            FlatFeeWithRecipient({recipient: address(0), amount: 0}),
            _emptySwapHooksExtraData(),
            "",
            LBAMM__DeadlineExpired.selector
        );
    }
}
```

## Pattern 2: Token Hook Test

```solidity
contract TokenHookTest is LBAMMCorePoolBaseTest {
    MyTokenHook public hook;
    bytes32 public poolId;

    function setUp() public override {
        super.setUp();

        // Deploy hook
        hook = new MyTokenHook(address(amm));

        // Set token settings with hook
        changePrank(AMM_ADMIN);
        TokenFlagSettings memory flags = TokenFlagSettings({
            beforeSwapHook: true,
            afterSwapHook: true,
            addLiquidityHook: false,
            removeLiquidityHook: false,
            collectFeesHook: false,
            poolCreationHook: false,
            hookManagesFees: false,
            flashLoans: false,
            flashLoansValidateFee: false,
            validateHandlerOrderHook: false
        });
        _setTokenSettings(address(weth), address(hook), flags, bytes4(0));

        // Create pool and add liquidity...
    }

    function test_hook_chargesSwapFees() public {
        // Setup swap...
        (uint256 amountIn, uint256 amountOut) = _executeSingleSwap(...);

        // Verify hook fees were collected
        uint256 hookFees = amm.getHookFeesOwedByToken(address(weth), address(weth));
        assertGt(hookFees, 0, "Hook should have collected fees");
    }

    function test_hook_blocksUnauthorizedSwap() public {
        // Test that hook reverts for unauthorized executor
        _executeSingleSwap(
            ...,
            MyTokenHook__Unauthorized.selector  // Expected revert
        );
    }
}
```

## Pattern 3: Pool Hook Test with Dynamic Fees

```solidity
contract DynamicFeePoolHookTest is LBAMMCorePoolBaseTest {
    DynamicFeeHook public poolHook;

    function setUp() public override {
        super.setUp();

        poolHook = new DynamicFeeHook();

        // Create pool with dynamic fee (55_555) and pool hook
        PoolCreationDetails memory details = PoolCreationDetails({
            poolType: address(dynamicPoolType),
            fee: 55_555,  // Dynamic fee sentinel
            token0: address(usdc),
            token1: address(weth),
            poolHook: address(poolHook),
            poolParams: abi.encode(DynamicPoolCreationDetails({
                tickSpacing: 60,
                sqrtPriceRatioX96: sqrtPrice
            }))
        });

        poolId = _createPool(details, "", "", "", bytes4(0));
        // ... add liquidity
    }

    function test_dynamicFee_appliesCorrectFee() public {
        // Small swap should get base fee
        // Large swap should get higher fee
    }
}
```

## Pattern 4: Multi-Hop Swap with Fee Verification

```solidity
function test_multiSwap_withExchangeFee() public {
    bytes32[] memory poolIds = new bytes32[](2);
    poolIds[0] = poolAB;
    poolIds[1] = poolBC;

    SwapHooksExtraData[] memory hookDatas = new SwapHooksExtraData[](2);
    hookDatas[0] = _emptySwapHooksExtraData();
    hookDatas[1] = _emptySwapHooksExtraData();

    SwapOrder memory order = SwapOrder({
        deadline: block.timestamp + 300,
        recipient: bob,
        amountSpecified: int256(100e18),
        minAmountSpecified: 0,
        limitAmount: 0,
        tokenIn: tokenA,
        tokenOut: tokenC
    });

    uint256 exchangeRecipientBalBefore = IERC20(tokenA).balanceOf(exchangeFeeRecipient);

    (uint256 amountIn, uint256 amountOut) = _executeMultiSwap(
        order,
        poolIds,
        BPSFeeWithRecipient({recipient: exchangeFeeRecipient, BPS: 100}), // 1%
        FlatFeeWithRecipient({recipient: address(0), amount: 0}),
        hookDatas,
        "",
        bytes4(0)
    );

    // Verify exchange fee was collected
    uint256 exchangeRecipientBalAfter = IERC20(tokenA).balanceOf(exchangeFeeRecipient);
    assertGt(exchangeRecipientBalAfter - exchangeRecipientBalBefore, 0, "Exchange fee should be collected");

    // Verify protocol fee on exchange fee
    uint256 protocolFees = amm.getProtocolFees(tokenA);
    assertGt(protocolFees, 0, "Protocol fee should be collected");
}
```

## Pattern 5: Liquidity Hook Test

```solidity
contract PositionHookTest is LBAMMCorePoolBaseTest {
    TimeLockHook public positionHook;

    function test_timelock_blocksEarlyWithdrawal() public {
        // Add liquidity with hook
        changePrank(alice);
        _executeAddLiquidity(
            LiquidityModificationParams({
                liquidityHook: address(positionHook),
                poolId: poolId,
                ...
            }),
            _emptyLiquidityHooksExtraData(),
            bytes4(0)
        );

        // Try to remove immediately -- should revert
        _executeRemoveLiquidity(
            LiquidityModificationParams({
                liquidityHook: address(positionHook),
                poolId: poolId,
                ...
            }),
            _emptyLiquidityHooksExtraData(),
            TimeLockHook__PositionLocked.selector
        );

        // Advance time past lock
        vm.warp(block.timestamp + 1 days + 1);

        // Now removal should succeed
        _executeRemoveLiquidity(
            LiquidityModificationParams({
                liquidityHook: address(positionHook),
                poolId: poolId,
                ...
            }),
            _emptyLiquidityHooksExtraData(),
            bytes4(0)
        );
    }
}
```

## Test Naming Convention

```
test_<operation>_<condition>_<outcome>

Examples:
test_singleSwap_inputSwap_success
test_singleSwap_revert_expiredDeadline
test_addLiquidity_withHookFees_collectsCorrectFees
test_removeLiquidity_beforeLockExpiry_reverts
test_multiSwap_threeHops_correctFeeDistribution
```

## Common Assertions

```solidity
// Balance checks
assertEq(IERC20(token).balanceOf(user), expected, "Balance mismatch");

// Pool state checks
PoolState memory state = amm.getPoolState(poolId);
assertGt(state.reserve0, 0, "Pool should have reserves");

// Fee checks
assertGt(amm.getProtocolFees(token), 0, "Protocol fees should accrue");
assertEq(amm.getHookFeesOwedByToken(token, feeToken), expectedFees, "Hook fees mismatch");

// Revert checks (pass error selector to helper)
_executeSingleSwap(..., ErrorName.selector);
```

## Security Test Checklist

1. Test reentrancy via hook callbacks
2. Test fee overflow/underflow conditions
3. Test expired deadlines
4. Test zero amounts and max amounts
5. Test invalid pool IDs
6. Test unauthorized access (wrong roles)
7. Test hook revert propagation
8. Test multi-hop with all-zero intermediate outputs
9. Test native token edge cases
10. Test partial fill scenarios
