# LBAMM Core Data Types & Pool Type Interface

## Core Data Types (`lbamm-core/src/DataTypes.sol`)

### Pool & Storage Types

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

struct PoolCreationDetails {
    address poolType;    // Pool type contract address (must have 6 leading zero bytes)
    uint16 fee;          // Pool fee in BPS (max 10000), or 55_555 for dynamic fees
    address token0;      // First token (must be < token1)
    address token1;      // Second token (must be > token0)
    address poolHook;    // Pool hook contract (zero = no hook)
    bytes poolParams;    // Pool-type-specific encoded params
}

struct TokenSettings {
    uint16 hopFeeBPS;       // Fee for multi-hop swaps
    uint32 packedSettings;  // Bits 0-9: hook flags, Bit 10: require executor pays
    address tokenHook;      // Hook contract address (zero = no hook)
}
```

### Fee Types

```solidity
struct FlatFeeWithRecipient {
    address recipient;
    uint256 amount;      // Fixed fee amount in wei
}

struct BPSFeeWithRecipient {
    address recipient;
    uint256 BPS;         // Fee in basis points
}

struct ProtocolFeeStructure {
    uint16 lpFeeBPS;        // Protocol fee on LP fees
    uint16 exchangeFeeBPS;  // Protocol fee on exchange fees
    uint16 feeOnTopBPS;     // Protocol fee on fee-on-top
}

struct ProtocolFeeOverride {
    bool feeOverrideEnabled;   // True if the fee override is enabled
    uint16 protocolFeeBPS;     // Protocol fee rate to apply in BPS
}
```

### Role Constants

```solidity
bytes32 constant LBAMM_FEE_MANAGER_BASE_ROLE = keccak256("LBAMM_FEE_MANAGER_ROLE");
bytes32 constant LBAMM_FEE_RECEIVER_BASE_ROLE = keccak256("LBAMM_FEE_RECEIVER_ROLE");
bytes32 constant LBAMM_TOKEN_FEE_COLLECTOR_ROLE = keccak256("LBAMM_TOKEN_FEE_COLLECTOR_ROLE");
bytes32 constant LBAMM_TOKEN_SETTING_MANAGER_ROLE = keccak256("LBAMM_TOKEN_SETTING_MANAGER_ROLE");
```

### Swap Types

```solidity
struct SwapOrder {
    uint256 deadline;          // Unix timestamp
    address recipient;         // Output token receiver
    int256 amountSpecified;    // Positive = input swap, negative = output swap
    uint256 minAmountSpecified; // Min amount for partial fills
    uint256 limitAmount;       // Max input (output swap) or min output (input swap)
    address tokenIn;
    address tokenOut;
}

struct DirectSwapParams {
    uint256 swapAmount;    // Exact swap amount
    uint256 maxAmountOut;  // Max output supplied by executor
    uint256 minAmountIn;   // Min input received by executor
}

struct SwapContext {
    address executor;
    address transferHandler;
    address exchangeFeeRecipient;
    uint256 exchangeFeeBPS;
    address feeOnTopRecipient;
    uint256 feeOnTopAmount;
    address recipient;
    address tokenIn;
    address tokenOut;
    uint256 numberOfHops;
}
```

### Hook Parameter Types

```solidity
struct HookSwapParams {
    bool inputSwap;
    uint256 hopIndex;
    bytes32 poolId;
    address tokenIn;
    address tokenOut;
    uint256 amount;
    bool hookForInputToken;  // True if this hook call is for the input token
}

struct HookPoolFeeParams {
    bool inputSwap;
    uint256 hopIndex;
    bytes32 poolId;
    address tokenIn;
    address tokenOut;
    uint256 amount;
}

struct SwapHooksExtraData {
    bytes tokenInHook;
    bytes tokenOutHook;
    bytes poolHook;
    bytes poolType;
}
```

### Liquidity Types

```solidity
struct LiquidityModificationParams {
    address liquidityHook;     // Position hook (zero = no hook)
    bytes32 poolId;
    uint256 minLiquidityAmount0;
    uint256 minLiquidityAmount1;
    uint256 maxLiquidityAmount0;
    uint256 maxLiquidityAmount1;
    uint256 maxHookFee0;       // Max hook fee cap for token0
    uint256 maxHookFee1;       // Max hook fee cap for token1
    bytes poolParams;          // Pool-type-specific params
}

struct LiquidityCollectFeesParams {
    address liquidityHook;
    bytes32 poolId;
    uint256 maxHookFee0;
    uint256 maxHookFee1;
    bytes poolParams;
}

struct LiquidityContext {
    address provider;
    address token0;
    address token1;
    bytes32 positionId;
}

struct LiquidityHooksExtraData {
    bytes token0Hook;
    bytes token1Hook;
    bytes liquidityHook;
    bytes poolHook;
}
```

### Flashloan Types

```solidity
struct FlashloanRequest {
    address loanToken;
    uint256 loanAmount;
    address executor;
    bytes executorData;
    bytes tokenHookData;
    bytes feeTokenHookData;
}
```

## ILimitBreakAMMPoolType Interface

Pool types are modular contracts that provide swap math and liquidity accounting. They do NOT hold reserves.

```solidity
interface ILimitBreakAMMPoolType {
    function createPool(PoolCreationDetails calldata details) external returns (bytes32 poolId);
    function computePoolId(PoolCreationDetails calldata details) external view returns (bytes32 poolId);

    function collectFees(bytes32 poolId, address provider, bytes32 ammBasePositionId, bytes calldata poolParams)
        external returns (bytes32 positionId, uint256 fees0, uint256 fees1);

    function addLiquidity(bytes32 poolId, address provider, bytes32 ammBasePositionId, bytes calldata poolParams)
        external returns (bytes32 positionId, uint256 deposit0, uint256 deposit1, uint256 fees0, uint256 fees1);

    function removeLiquidity(bytes32 poolId, address provider, bytes32 ammBasePositionId, bytes calldata poolParams)
        external returns (bytes32 positionId, uint256 withdraw0, uint256 withdraw1, uint256 fees0, uint256 fees1);

    function swapByInput(
        SwapContext calldata context, bytes32 poolId, bool zeroForOne,
        uint256 amountIn, uint256 poolFeeBPS, uint256 protocolFeeBPS, bytes calldata swapExtraData
    ) external returns (uint256 actualAmountIn, uint256 amountOut, uint256 feeOfAmountIn, uint256 protocolFees);

    function swapByOutput(
        SwapContext calldata context, bytes32 poolId, bool zeroForOne,
        uint256 amountOut, uint256 poolFeeBPS, uint256 protocolFeeBPS, bytes calldata swapExtraData
    ) external returns (uint256 actualAmountOut, uint256 amountIn, uint256 feeOfAmountIn, uint256 protocolFees);

    function getCurrentPriceX96(address amm, bytes32 poolId) external view returns (uint160 sqrtPriceX96);

    function poolTypeManifestUri() external view returns (string memory manifestUri);
}
```

## Pool ID Encoding

The pool ID is a packed `bytes32` combining the pool type address, a creation details hash, and pool-specific packed data. The bit layout:

```
poolId (bytes32):
  Bits    0-15: Pool fee in BPS (uint16, POOL_ID_FEE_SHIFT = 0)
  Bits  16-143: Creation details hash (may include pool-specific params)
  Bits 144-255: Pool type address (must have 6 leading zero bytes to fit in 112 bits)
```

Construction uses `EfficientHash` of all parameters, masked by `POOL_HASH_MASK`, then OR'd with the bit-shifted pool type address and fee.

```solidity
POOL_HASH_MASK = 0x0000000000000000000000000000FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF0000;
```

Extraction:
```solidity
// Pool type: right-shift by 144
address poolType = address(uint160(uint256(poolId) >> POOL_ID_TYPE_ADDRESS_SHIFT));
// Fee: cast low 16 bits
uint16 fee = uint16(uint256(poolId));
```
