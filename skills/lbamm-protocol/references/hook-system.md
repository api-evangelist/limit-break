# LBAMM Hook System

## Overview

LBAMM uses a 3-tier hook system for policy enforcement. Hooks are external contracts invoked by LBAMM core at defined execution points. Hooks do NOT transfer tokens, mutate core accounting, or partially commit executions.

## Tier 1: Token Hooks (`ILimitBreakAMMTokenHook`)

Per-token hooks. Set via `setTokenSettings(token, tokenHook, packedSettings)`. The broadest hook surface -- involved in swaps, liquidity ops, pool creation, flashloans, and handler order validation.

### 10 Hook Flags (bits 0-9 of `packedSettings`)

| Bit | Flag Constant | Description |
|-----|--------------|-------------|
| 0 | `TOKEN_SETTINGS_BEFORE_SWAP_HOOK_FLAG` | Enable `beforeSwap()` callback |
| 1 | `TOKEN_SETTINGS_AFTER_SWAP_HOOK_FLAG` | Enable `afterSwap()` callback |
| 2 | `TOKEN_SETTINGS_ADD_LIQUIDITY_HOOK_FLAG` | Enable `validateAddLiquidity()` callback |
| 3 | `TOKEN_SETTINGS_REMOVE_LIQUIDITY_HOOK_FLAG` | Enable `validateRemoveLiquidity()` callback |
| 4 | `TOKEN_SETTINGS_COLLECT_FEES_HOOK_FLAG` | Enable `validateCollectFees()` callback |
| 5 | `TOKEN_SETTINGS_POOL_CREATION_HOOK_FLAG` | Enable `validatePoolCreation()` callback |
| 6 | `TOKEN_SETTINGS_HOOK_MANAGES_FEES_FLAG` | Hook manages its own fee collection |
| 7 | `TOKEN_SETTINGS_FLASHLOANS_FLAG` | Enable `beforeFlashloan()` callback |
| 8 | `TOKEN_SETTINGS_FLASHLOANS_VALIDATE_FEE_FLAG` | Enable `validateFlashloanFee()` callback |
| 9 | `TOKEN_SETTINGS_HANDLER_ORDER_VALIDATE_FLAG` | Enable `validateHandlerOrder()` callback |

### Flag Negotiation via `hookFlags()`

When a token hook is set, the AMM calls `hookFlags()` to get:
- `requiredFlags`: Flags the token owner MUST enable
- `supportedFlags`: Flags the token owner MAY enable

Rules:
- All required flags must also be supported
- AMM reverts if the token owner enables an unsupported flag
- AMM reverts if the token owner doesn't enable a required flag

### Token Hook Callbacks

```solidity
// Pool creation validation
function validatePoolCreation(bytes32 poolId, address creator, bool hookForToken0, PoolCreationDetails calldata details, bytes calldata hookData) external;

// Swap hooks (return fee amounts)
function beforeSwap(SwapContext calldata context, HookSwapParams calldata swapParams, bytes calldata hookData) external returns (uint256 fee);
function afterSwap(SwapContext calldata context, HookSwapParams calldata swapParams, bytes calldata hookData) external returns (uint256 fee);

// Liquidity hooks (return hook fees)
function validateAddLiquidity(bool hookForToken0, LiquidityContext calldata context, LiquidityModificationParams calldata params, uint256 deposit0, uint256 deposit1, uint256 fees0, uint256 fees1, bytes calldata hookData) external returns (uint256 hookFee0, uint256 hookFee1);
function validateRemoveLiquidity(bool hookForToken0, LiquidityContext calldata context, LiquidityModificationParams calldata params, uint256 withdraw0, uint256 withdraw1, uint256 fees0, uint256 fees1, bytes calldata hookData) external returns (uint256 hookFee0, uint256 hookFee1);
function validateCollectFees(bool hookForToken0, LiquidityContext calldata context, LiquidityCollectFeesParams calldata params, uint256 fees0, uint256 fees1, bytes calldata hookData) external returns (uint256 hookFee0, uint256 hookFee1);

// Handler order validation
function validateHandlerOrder(address maker, bool hookForTokenIn, address tokenIn, address tokenOut, uint256 amountIn, uint256 amountOut, bytes calldata handlerOrderParams, bytes calldata hookData) external;

// Flashloan hooks
function beforeFlashloan(address requester, address loanToken, uint256 loanAmount, address executor, bytes calldata hookData) external returns (address feeToken, uint256 fee);
function validateFlashloanFee(address requester, address loanToken, uint256 loanAmount, address feeToken, uint256 feeAmount, address executor, bytes calldata hookData) external returns (bool allowed);

// Manifest
function tokenHookManifestUri() external view returns (string memory);
```

## Tier 2: Pool Hooks (`ILimitBreakAMMPoolHook`)

Per-pool hooks. Set at pool creation via `PoolCreationDetails.poolHook`. Validates pool creation, liquidity ops, and optionally provides dynamic LP fees.

### Pool Hook Callbacks

```solidity
function validatePoolCreation(bytes32 poolId, address creator, PoolCreationDetails calldata details, bytes calldata hookData) external;

// Dynamic fee calculation (only called when pool fee == 55_555)
function getPoolFeeForSwap(SwapContext calldata context, HookPoolFeeParams calldata params, bytes calldata hookData) external returns (uint256 poolFeeBPS);

// Liquidity validation (return hook fees)
function validatePoolAddLiquidity(LiquidityContext calldata context, LiquidityModificationParams calldata params, uint256 deposit0, uint256 deposit1, uint256 fees0, uint256 fees1, bytes calldata hookData) external returns (uint256 hookFee0, uint256 hookFee1);
function validatePoolRemoveLiquidity(LiquidityContext calldata context, LiquidityModificationParams calldata params, uint256 withdraw0, uint256 withdraw1, uint256 fees0, uint256 fees1, bytes calldata hookData) external returns (uint256 hookFee0, uint256 hookFee1);
function validatePoolCollectFees(LiquidityContext calldata context, LiquidityCollectFeesParams calldata params, uint256 fees0, uint256 fees1, bytes calldata hookData) external returns (uint256 hookFee0, uint256 hookFee1);

function poolHookManifestUri() external view returns (string memory);
```

### Dynamic Fee Sentinel

When a pool is created with `fee = 55_555` (`DYNAMIC_POOL_FEE_BPS`), the AMM calls `getPoolFeeForSwap()` on the pool hook for every swap. The hook returns the LP fee in BPS.

Fee bounds:
- Input swap: `poolFeeBPS <= 10_000`
- Output swap: `poolFeeBPS < 10_000` (strictly less than 100%)

## Tier 3: Position Hooks / Liquidity Hooks (`ILimitBreakAMMLiquidityHook`)

Per-position hooks. Set per liquidity operation via `LiquidityModificationParams.liquidityHook`. The hook address becomes part of the position identity (base position ID).

### Position Hook Callbacks

```solidity
function validatePositionAddLiquidity(LiquidityContext calldata context, LiquidityModificationParams calldata params, uint256 deposit0, uint256 deposit1, uint256 fees0, uint256 fees1, bytes calldata hookData) external returns (uint256 hookFee0, uint256 hookFee1);
function validatePositionRemoveLiquidity(LiquidityContext calldata context, LiquidityModificationParams calldata params, uint256 withdraw0, uint256 withdraw1, uint256 fees0, uint256 fees1, bytes calldata hookData) external returns (uint256 hookFee0, uint256 hookFee1);
function validatePositionCollectFees(LiquidityContext calldata context, LiquidityCollectFeesParams calldata params, uint256 fees0, uint256 fees1, bytes calldata hookData) external returns (uint256 hookFee0, uint256 hookFee1);

function liquidityHookManifestUri() external view returns (string memory);
```

## Hook Call Ordering

### Swap Operations
1. Token In hook: `beforeSwap()`
2. Token Out hook: `beforeSwap()`
3. Pool hook: `getPoolFeeForSwap()` (if dynamic fee)
4. Pool type: swap execution
5. Token In hook: `afterSwap()`
6. Token Out hook: `afterSwap()`

### Liquidity Operations (Add/Remove/Collect Fees)
1. Token0 hook: `validateAddLiquidity()` / `validateRemoveLiquidity()` / `validateCollectFees()`
2. Token1 hook: same
3. Position hook: `validatePositionAddLiquidity()` / etc.
4. Pool hook: `validatePoolAddLiquidity()` / etc.

### Pool Creation
1. Token0 hook: `validatePoolCreation(hookForToken0=true)`
2. Token1 hook: `validatePoolCreation(hookForToken0=false)`
3. Pool hook: `validatePoolCreation()`

## Fee Behavior in Hooks

- Swap hooks (`beforeSwap`/`afterSwap`): Return a fee amount. `beforeSwap` fee is on input token (input swap) or output token (output swap). `afterSwap` fee is on the opposite token.
- Liquidity hooks: Return `(hookFee0, hookFee1)`. Fees are accrued in core, not transferred immediately. All hook fees across tiers are aggregated per token and compared against `maxHookFee0`/`maxHookFee1`.
- Hook fee collection is done later via `collectFees()` if `TOKEN_SETTINGS_HOOK_MANAGES_FEES_FLAG` is set.
- Queued fee transfers execute FIFO after the primary operation finalizes.

## Base Position ID Derivation

```
basePositionId = keccak256(abi.encodePacked(provider, liquidityHook, poolId))
```

The position hook address is part of position identity -- different hooks create different positions even for the same provider and pool.
