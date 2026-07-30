# LBAMM Core API

Entry points for the `ILimitBreakAMM` composite interface. The AMM contract inherits from all sub-interfaces listed below.

## Swap Functions (`ILimitBreakAMMSwap`)

```solidity
function singleSwap(
    SwapOrder calldata swapOrder,
    bytes32 poolId,
    BPSFeeWithRecipient calldata exchangeFee,
    FlatFeeWithRecipient calldata feeOnTop,
    SwapHooksExtraData calldata swapHooksExtraData,
    bytes calldata transferData
) external payable returns (uint256 amountIn, uint256 amountOut);

function multiSwap(
    SwapOrder calldata swapOrder,
    bytes32[] calldata poolIds,
    BPSFeeWithRecipient calldata exchangeFee,
    FlatFeeWithRecipient calldata feeOnTop,
    SwapHooksExtraData[] calldata swapHooksExtraDatas,
    bytes calldata transferData
) external payable returns (uint256 amountIn, uint256 amountOut);

function directSwap(
    SwapOrder calldata swapOrder,
    DirectSwapParams calldata directSwapParams,
    BPSFeeWithRecipient calldata exchangeFee,
    FlatFeeWithRecipient calldata feeOnTop,
    SwapHooksExtraData calldata swapHooksExtraData,
    bytes calldata transferData
) external payable returns (uint256 amountIn, uint256 amountOut);
```

## Pool State & Liquidity (`ILimitBreakAMMLiquidity`)

```solidity
function getPoolState(bytes32 poolId) external view returns (PoolState memory state);

function createPool(
    PoolCreationDetails memory details,
    bytes calldata token0HookData,
    bytes calldata token1HookData,
    bytes calldata poolHookData,
    bytes calldata liquidityData  // Optional: calldata to addLiquidity after creation
) external payable returns (bytes32 poolId, uint256 deposit0, uint256 deposit1);

function addLiquidity(
    LiquidityModificationParams calldata liquidityParams,
    LiquidityHooksExtraData calldata liquidityHooksExtraData
) external payable returns (uint256 deposit0, uint256 deposit1, uint256 fees0, uint256 fees1);

function removeLiquidity(
    LiquidityModificationParams calldata liquidityParams,
    LiquidityHooksExtraData calldata liquidityHooksExtraData
) external returns (uint256 withdraw0, uint256 withdraw1, uint256 fees0, uint256 fees1);

function collectFees(
    LiquidityCollectFeesParams calldata liquidityParams,
    LiquidityHooksExtraData calldata liquidityHooksExtraData
) external returns (uint256 fees0, uint256 fees1);
```

## Token Settings (`ILimitBreakAMMTokenSettings`)

```solidity
function setTokenSettings(address token, address tokenHook, uint32 packedSettings) external;
function getTokenSettings(address token) external view returns (TokenSettings memory);
```

## Fee Functions (`ILimitBreakAMMFees`)

```solidity
// Collect tokens owed to the caller (hook fees, failed transfers, etc.)
function collectTokensOwed(address[] calldata tokensOwed) external;

// Hook collects its own accumulated fees
function collectHookFeesByHook(address tokenFor, address tokenFee, address recipient, uint256 amount) external;

// Token admin collects hook fees for their token
function collectHookFeesByToken(address tokenFor, address tokenFee, address recipient, uint256 amount) external;

// View: tokens owed to a user
function getTokensOwed(address user, address token) external view returns (uint256 tokensOwedAmount);

// View: hook fees owed to a hook contract
function getHookFeesOwedByHook(address hook, address tokenFor, address tokenFee) external view returns (uint256);

// View: hook fees owed for a token
function getHookFeesOwedByToken(address tokenFor, address tokenFee) external view returns (uint256);
```

## Flashloan Functions (`ILimitBreakAMMFlashloan`)

```solidity
// Returns the flash loan fee rate in basis points
function getFlashloanFeeBPS() external view returns (uint16 flashLoanBPS);

// Executes a flash loan. Fee BPS > MAX_BPS disables flash loans.
function flashLoan(FlashloanRequest calldata flashloanRequest) external;
```

## Protocol Management (`ILimitBreakAMMProtocol`)

```solidity
// View: protocol fee structure for a given exchange/feeOnTop recipient and pool
function getProtocolFeeStructure(
    address exchangeFeeRecipient, address feeOnTopRecipient, bytes32 poolId
) external view returns (ProtocolFeeStructure memory);

// View: accumulated protocol fees for a token
function getProtocolFees(address token) external view returns (uint256 fees);

// Set global protocol fee structure (requires FEE_MANAGER role)
function setProtocolFees(ProtocolFeeStructure memory protocolFeeStructure) external;

// Set protocol fee override for an exchange fee recipient
function setExchangeProtocolFeeOverride(address recipient, bool feeOverrideEnabled, uint256 protocolFeeBPS) external;

// Set protocol fee override for a fee-on-top recipient
function setFeeOnTopProtocolFeeOverride(address recipient, bool feeOverrideEnabled, uint256 protocolFeeBPS) external;

// Set protocol fee override for a specific pool's LP fees
function setLPProtocolFeeOverride(bytes32 poolId, bool feeOverrideEnabled, uint256 protocolFeeBPS) external;

// Set flash loan fee rate (requires FEE_MANAGER role; > MAX_BPS disables flash loans)
function setFlashloanFee(uint256 flashLoanBPS) external;

// Set per-token hop fees (requires FEE_MANAGER role)
function setTokenFees(address[] calldata tokens, uint16[] calldata hopFeesBPS) external;

// Collect accumulated protocol fees -- unrestricted caller, fees always go to FEE_RECEIVER role
function collectProtocolFees(address[] calldata tokens) external;

// Internal: execute queued hook fee transfers after a swap
function executeQueuedHookFeesByHookTransfers() external;

// Check AMM execution state via reentrancy flags
function checkAMMExecutionState(uint256 flags) external view returns (bool);
```
