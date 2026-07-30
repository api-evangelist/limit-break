# Token Hook Interface Reference

## ILimitBreakAMMTokenHook

Source: `lbamm-core/src/interfaces/hooks/ILimitBreakAMMTokenHook.sol`

```solidity
interface ILimitBreakAMMTokenHook {
    event TokenHookManifestUriUpdated(string uri);

    /// @notice Returns required and supported flags for hook negotiation
    /// @return requiredFlags  Flags that MUST be enabled by token owner
    /// @return supportedFlags Flags that MAY be enabled (must include all required)
    function hookFlags() external view returns (uint32 requiredFlags, uint32 supportedFlags);

    /// @notice Validates pool creation. Revert to block.
    function validatePoolCreation(
        bytes32 poolId,
        address creator,
        bool hookForToken0,
        PoolCreationDetails calldata details,
        bytes calldata hookData
    ) external;

    /// @notice Called before swap. Returns fee amount.
    /// @dev Fee is on input token (input swap) or output token (output swap)
    function beforeSwap(
        SwapContext calldata context,
        HookSwapParams calldata swapParams,
        bytes calldata hookData
    ) external returns (uint256 fee);

    /// @notice Called after swap. Returns fee amount.
    /// @dev Fee is on output token (input swap) or input token (output swap)
    function afterSwap(
        SwapContext calldata context,
        HookSwapParams calldata swapParams,
        bytes calldata hookData
    ) external returns (uint256 fee);

    /// @notice Validates handler order creation. Revert to block.
    function validateHandlerOrder(
        address maker,
        bool hookForTokenIn,
        address tokenIn,
        address tokenOut,
        uint256 amountIn,
        uint256 amountOut,
        bytes calldata handlerOrderParams,
        bytes calldata hookData
    ) external;

    /// @notice Validates fee collection. Returns hook fees.
    function validateCollectFees(
        bool hookForToken0,
        LiquidityContext calldata context,
        LiquidityCollectFeesParams calldata liquidityParams,
        uint256 fees0,
        uint256 fees1,
        bytes calldata hookData
    ) external returns (uint256 hookFee0, uint256 hookFee1);

    /// @notice Validates liquidity add. Returns hook fees.
    function validateAddLiquidity(
        bool hookForToken0,
        LiquidityContext calldata context,
        LiquidityModificationParams calldata liquidityParams,
        uint256 deposit0,
        uint256 deposit1,
        uint256 fees0,
        uint256 fees1,
        bytes calldata hookData
    ) external returns (uint256 hookFee0, uint256 hookFee1);

    /// @notice Validates liquidity removal. Returns hook fees.
    function validateRemoveLiquidity(
        bool hookForToken0,
        LiquidityContext calldata context,
        LiquidityModificationParams calldata liquidityParams,
        uint256 withdraw0,
        uint256 withdraw1,
        uint256 fees0,
        uint256 fees1,
        bytes calldata hookData
    ) external returns (uint256 hookFee0, uint256 hookFee1);

    /// @notice Called before flashloan. Returns fee token and fee amount.
    function beforeFlashloan(
        address requester,
        address loanToken,
        uint256 loanAmount,
        address executor,
        bytes calldata hookData
    ) external returns (address feeToken, uint256 fee);

    /// @notice Validates this token as flashloan fee token. Return false or revert to block.
    function validateFlashloanFee(
        address requester,
        address loanToken,
        uint256 loanAmount,
        address feeToken,
        uint256 feeAmount,
        address executor,
        bytes calldata hookData
    ) external returns (bool allowed);

    /// @notice Returns the manifest URI for app integrations
    function tokenHookManifestUri() external view returns (string memory manifestUri);
}
```

## 10 Hook Flag Constants (`lbamm-core/src/Constants.sol`)

```solidity
uint16 constant TOKEN_SETTINGS_BEFORE_SWAP_HOOK_FLAG           = 1 << 0;  // Bit 0
uint16 constant TOKEN_SETTINGS_AFTER_SWAP_HOOK_FLAG             = 1 << 1;  // Bit 1
uint16 constant TOKEN_SETTINGS_ADD_LIQUIDITY_HOOK_FLAG          = 1 << 2;  // Bit 2
uint16 constant TOKEN_SETTINGS_REMOVE_LIQUIDITY_HOOK_FLAG       = 1 << 3;  // Bit 3
uint16 constant TOKEN_SETTINGS_COLLECT_FEES_HOOK_FLAG           = 1 << 4;  // Bit 4
uint16 constant TOKEN_SETTINGS_POOL_CREATION_HOOK_FLAG          = 1 << 5;  // Bit 5
uint16 constant TOKEN_SETTINGS_HOOK_MANAGES_FEES_FLAG           = 1 << 6;  // Bit 6
uint16 constant TOKEN_SETTINGS_FLASHLOANS_FLAG                  = 1 << 7;  // Bit 7
uint16 constant TOKEN_SETTINGS_FLASHLOANS_VALIDATE_FEE_FLAG     = 1 << 8;  // Bit 8
uint16 constant TOKEN_SETTINGS_HANDLER_ORDER_VALIDATE_FLAG      = 1 << 9;  // Bit 9

uint16 constant TOKEN_SETTINGS_HOOK_FLAGS_MASK = 0x03FF; // All 10 flags
```

## Hook Flag Negotiation Rules

When `setTokenSettings(token, tokenHook, packedSettings)` is called:
1. AMM calls `tokenHook.hookFlags()` → `(requiredFlags, supportedFlags)`
2. `requiredFlags` must be a subset of `supportedFlags` (all required must be supported)
3. `packedSettings & TOKEN_SETTINGS_HOOK_FLAGS_MASK` must include all `requiredFlags`
4. `packedSettings & TOKEN_SETTINGS_HOOK_FLAGS_MASK` must be a subset of `supportedFlags`
5. If any check fails, the AMM reverts

## Fee Behavior Per Callback

| Callback | Fee Token | When Assessed |
|---|---|---|
| `beforeSwap` (input swap) | tokenIn | Per hop, before pool swap |
| `beforeSwap` (output swap) | tokenOut | Per hop, before pool swap |
| `afterSwap` (input swap) | tokenOut | Per hop, after pool swap |
| `afterSwap` (output swap) | tokenIn | Per hop, after pool swap |
| `validateAddLiquidity` | token0/token1 | On deposit |
| `validateRemoveLiquidity` | token0/token1 | On withdrawal |
| `validateCollectFees` | token0/token1 | On fee claim |
| `beforeFlashloan` | feeToken (returned) | Before flashloan |

## `HookSwapParams.amount` Semantics

The `amount` field in `HookSwapParams` varies by callback and swap direction:

| Callback | Input Swap (`inputSwap=true`) | Output Swap (`inputSwap=false`) |
|---|---|---|
| `beforeSwap` | `amountIn` (input amount) | `amountOut` (output amount) |
| `afterSwap` | `amountOut` (output amount) | `amountIn` (input amount) |

In `beforeSwap`, `amount` is the "specified" side of the swap. In `afterSwap`, `amount` is the "unspecified" (computed) side. Both the tokenIn and tokenOut hooks receive the same `amount` value for a given phase.

## `hookManagesFees` Flag (Bit 6)

Controls how hook fees are keyed in storage and who can collect them:

| | Flag SET | Flag NOT SET |
|---|---|---|
| Storage key | `hash(hookAddress, hash(tokenFor, tokenFee))` | `hash(TOKEN_MANAGED_HOOK_FEE, hash(tokenFor, tokenFee))` |
| Collection function | `collectHookFeesByHook()` | `collectHookFeesByToken()` |
| Who can collect | The hook contract itself (`msg.sender == hook`) | Token owner, admin, or `LBAMM_TOKEN_FEE_COLLECTOR_ROLE` holder |

In both cases fees are accrued in the `tokensOwed` mapping, never transferred directly. The hook or token admin must call the appropriate collection function to claim fees. If `collectHookFeesByHook()` is called during a swap or liquidity operation (re-entrancy guard active), the transfer is queued and executed after finalization.

## Important: `hookForToken0` / `hookForInputToken`

- In liquidity callbacks: `hookForToken0` tells you if this is the token0 or token1 callback for the same pool operation. The hook is called once for each token in the pool.
- In swap callbacks: `swapParams.hookForInputToken` tells you if the hook is for the input or output token of the swap.
