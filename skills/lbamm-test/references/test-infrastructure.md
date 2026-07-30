# LBAMM Test Infrastructure

## Base Test Classes

### LBAMMCoreBaseTest (`lbamm-core/test/LBAMMCoreBase.t.sol`)

Root test contract providing full AMM deployment infrastructure.

**setUp() provides:**
- `SecureProxy`-wrapped `LimitBreakAMM` deployment
- Three modules: `ModuleAdmin`, `ModuleLiquidity`, `ModuleFeeCollection`
- `RoleSetServer` with AMM roles configured
- `CreatorTokenTransferValidator` (for ERC-721C compatibility)
- Mock tokens at well-known addresses (via `vm.etch`)
- Test accounts: `admin`, `alice`, `bob`, `carol`, `exchangeFeeRecipient`, `feeOnTopRecipient`
- Default protocol fees: `lpFeeBPS=125, exchangeFeeBPS=500, feeOnTopBPS=2500`

**Key test variables:**
```solidity
LimitBreakAMM public amm;        // The AMM contract (proxy)
RoleSetServer public roleServer;
WETH9Mock public weth;             // At WETH_ADDRESS
ERC20Mock public usdc;             // At USDC_ADDRESS
ERC20Mock public currency2;        // At CURRENCY_2_ADDRESS
ERC20Mock public currency3;        // At CURRENCY_3_ADDRESS
ERC20Mock public currency4;        // At CURRENCY_4_ADDRESS
IWrappedNative public wrappedNative;
```

**Helper functions:**
```solidity
// Token management
function _mintAndApprove(address token, address receiver, address spender, uint256 amount) internal;
function _dealDepositApproveNative(address receiver, address spender, uint256 amount) internal;

// Token settings
function _setTokenSettings(address token, address hook, TokenFlagSettings memory settings, bytes4 errorSelector) internal;
function _packSettings(TokenFlagSettings memory settings) internal pure returns (uint32 packedSettings);

// Protocol fees
function _setProtocolFees(uint16 lpFeeBPS, uint16 exchangeFeeBPS, uint16 feeOnTopBPS, bytes4 errorSelector) internal;
function _setFlashLoanFee(uint16 flashLoanFeeBPS, bytes4 errorSelector) internal;
function _collectProtocolFees(address[] memory tokens, bytes4 errorSelector) internal;

// Fee collection
function _collectTokensOwed(address sender, address[] memory tokens, bytes4 errorSelector) internal;
function _collectHookFeesByToken(address sender, address tokenFor, address tokenFee, address recipient, uint256 amount, bytes4 errorSelector) internal;
function _collectHookFeesByHook(address sender, address tokenFee, address recipient, uint256 amount, bytes4 errorSelector) internal;

// Price calculation
function _calculatePriceLimit(uint256 amount1, uint256 amount0) internal pure returns (uint160 sqrtPriceX96);

// Prank management
function changePrank(address msgSender) internal override;
```

**TokenFlagSettings struct:**
```solidity
struct TokenFlagSettings {
    bool beforeSwapHook;
    bool afterSwapHook;
    bool addLiquidityHook;
    bool removeLiquidityHook;
    bool collectFeesHook;
    bool poolCreationHook;
    bool hookManagesFees;
    bool flashLoans;
    bool flashLoansValidateFee;
    bool validateHandlerOrderHook;
}
```

### LBAMMCorePoolBaseTest (`lbamm-core/test/LBAMMCorePoolBase.t.sol`)

Extends `LBAMMCoreBaseTest` with pool operation wrappers and fee verification.

**Execution wrappers:**
```solidity
function _executeSingleSwap(SwapOrder, bytes32 poolId, BPSFeeWithRecipient, FlatFeeWithRecipient, SwapHooksExtraData, bytes transferData, bytes4 errorSelector) internal returns (uint256 amountIn, uint256 amountOut);
function _executeMultiSwap(SwapOrder, bytes32[], BPSFeeWithRecipient, FlatFeeWithRecipient, SwapHooksExtraData[], bytes, bytes4) internal returns (uint256, uint256);
function _executeDirectSwap(SwapOrder, DirectSwapParams, BPSFeeWithRecipient, FlatFeeWithRecipient, SwapHooksExtraData, bytes transferData, bytes4 errorSelector) internal returns (uint256 amountIn, uint256 amountOut);

function _createPool(PoolCreationDetails, bytes token0HookData, bytes token1HookData, bytes poolHookData, bytes4 errorSelector) internal returns (bytes32 poolId);
function _createPoolWithAddLiquidity(PoolCreationDetails, bytes, bytes, bytes, bytes liquidityData, bytes4) internal returns (bytes32, uint256, uint256);

function _executeAddLiquidity(LiquidityModificationParams, LiquidityHooksExtraData, bytes4) internal returns (uint256 deposit0, uint256 deposit1, uint256 fee0, uint256 fee1);
function _executeRemoveLiquidity(LiquidityModificationParams, LiquidityHooksExtraData, bytes4) internal returns (uint256 withdraw0, uint256 withdraw1, uint256 fee0, uint256 fee1);
function _executeCollectFees(LiquidityCollectFeesParams, LiquidityHooksExtraData, bytes4) internal returns (uint256 fees0, uint256 fees1);
```

**Empty data helpers:**
```solidity
function _emptySwapHooksExtraData() internal pure returns (SwapHooksExtraData memory);
function _emptyLiquidityHooksExtraData() internal pure returns (LiquidityHooksExtraData memory);
```

**Fee calculation framework (virtual functions for override by hook tests):**
```solidity
function _calculateHookFeeBeforeSwapSwapByInput(...) internal view virtual returns (uint256) { return 0; }
function _calculateHookFeeAfterSwapSwapByInput(...) internal view virtual returns (uint256) { return 0; }
function _calculateHookFeeBeforeSwapSwapByOutput(...) internal view virtual returns (uint256) { return 0; }
function _calculateHookFeeAfterSwapSwapByOutput(...) internal view virtual returns (uint256) { return 0; }
```

## TestConstants (`lbamm-core/test/TestConstants.t.sol`)

```solidity
address constant KEYLESS_DEPLOYER = 0x4e59b44847b379578588920cA78FbF26c0B4956C;
bytes32 constant AMM_ROLE_SERVER_SET_SALT = keccak256("AMM_ROLES");
bytes32 constant AMM_SALT = keccak256("AMM_SALT");
bytes32 constant AMM_PROXY_SALT = keccak256("AMM_PROXY_SALT");
bytes32 constant MODULE_FEE_COLLECTION_SALT = keccak256("MODULE_FEE_COLLECTION_SALT");
bytes32 constant MODULE_ADMIN_SALT = keccak256("MODULE_ADMIN_SALT");
bytes32 constant MODULE_LIQUIDITY_SALT = keccak256("MODULE_LIQUIDITY_SALT");

address constant WRAPPED_NATIVE_ADDRESS = 0x6000030000842044000077551D00cfc6b4005900;
address constant WETH_ADDRESS = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
address constant USDC_ADDRESS = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
address constant CURRENCY_2_ADDRESS = 0xDDdDddDdDdddDDddDDddDDDDdDdDDdDDdDDDDDDd;
address constant CURRENCY_3_ADDRESS = 0x1111111111111111111111111111111111111111;
address constant CURRENCY_4_ADDRESS = 0x3333333333333333333333333333333333333333;
address constant ROLE_SERVER = 0x00000000d7b37203F54e165Fb204B57c30d15835;
address constant AMM_ADMIN = 0x591Aa9dfF01B8144DC17Cb416001D9aC84b951cd;
address constant AMM_FEE_RECEIVER = 0x591Aa9dfF01B8144DC17Cb416001D9aC84b951cd;

uint256 constant Q96 = 2 ** 96;
uint8 constant Q96_RESOLUTION = 96;
uint160 constant MIN_SQRT_RATIO = 4_295_128_739;
uint160 constant MAX_SQRT_RATIO = 1_461_446_703_485_210_103_287_273_052_203_988_822_378_723_970_342;
int24 constant MAX_TICK_SPACING = 16_384;
bytes4 constant PANIC_SELECTOR = 0x00000001;
```

## Mock Contracts

### ERC20Mock
```solidity
contract ERC20Mock {
    function mint(address to, uint256 amount) external;
    // Standard ERC20 functions
}
```

### MockPoolHook
Minimal `ILimitBreakAMMPoolHook` -- returns 5000 BPS fee, all validations are no-ops.

### MockLiquidityHookAudit
File: `MockLiquidityHookAuditAMM10.sol`. Contract name: `MockLiquidityHookAudit`.
`ILimitBreakAMMLiquidityHook` with state counters for tracking callback invocations per call context (token-level, pool-level, position-level).

### WETH9Mock
File: `WETHMock.sol`. Contract name: `WETH9Mock` (not `WETHMock`).
WETH9-compatible mock at `WETH_ADDRESS`.

### ReceiverMock
For testing token receipt callbacks.

## Deployment Pattern

The test suite uses deterministic CREATE2 deployment:
1. `vm.etch` places mock bytecode at well-known addresses
2. `vm.store` initializes storage slots (e.g., for role server)
3. Modules deployed with `new Module{salt: ...}(...)` from `KEYLESS_DEPLOYER`
4. AMM implementation + proxy deployed with CREATE2 salts
5. Pool types deployed separately (test-specific)
