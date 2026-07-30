# LBAMM Events & Errors

## Events (`ILimitBreakAMMEvents`)

```solidity
event PoolCreated(address indexed poolType, address indexed token0, address indexed token1, bytes32 poolId, address poolHook);
event TokenSettingsUpdated(address indexed token, address indexed tokenHook, uint32 packedSettings);
event ProtocolFeeTaken(address indexed token, uint256 amount);
event Swap(bytes32 indexed poolId, address indexed recipient, bool zeroForOne, uint256 amountIn, uint256 amountOut, uint256 lpFeeAmount);
event DirectSwap(address indexed executor, address indexed recipient, address indexed tokenIn, address tokenOut, uint256 amountIn, uint256 amountOut);
event TokensClaimed(address indexed recipient, address indexed token, uint256 amount);
event Flashloan(address indexed requester, address indexed executor, address indexed loanedToken, uint256 loanAmount, address feeToken, uint256 feeAmount);
event LiquidityAdded(bytes32 indexed poolId, address indexed provider, uint256 amount0, uint256 amount1, uint256 fees0, uint256 fees1);
event LiquidityRemoved(bytes32 indexed poolId, address indexed provider, uint256 amount0, uint256 amount1, uint256 fees0, uint256 fees1);
event FeesCollected(bytes32 indexed poolId, address indexed provider, uint256 fees0, uint256 fees1);
event ProtocolFeesSet(uint16 lpFeeBPS, uint16 exchangeFeeBPS, uint16 feeOnTopBPS);
event FlashloanFeeSet(uint16 flashLoanBPS);
event ProtocolFeesCollected(address indexed protocolFeeReceiver, address indexed token, uint256 protocolFeesCollected);
event TokenFeeSet(address indexed token, uint16 hopFeeBPS);
event ExchangeProtocolFeeOverrideSet(address recipient, bool feeOverrideEnabled, uint16 protocolFeeBPS);
event FeeOnTopProtocolFeeOverrideSet(address recipient, bool feeOverrideEnabled, uint16 protocolFeeBPS);
event LPProtocolFeeOverrideSet(bytes32 poolId, bool feeOverrideEnabled, uint16 protocolFeeBPS);
```

## Errors (`lbamm-core/src/Errors.sol`)

### Pool Lifecycle
```solidity
error LBAMM__PoolAlreadyExists();
error LBAMM__PoolDoesNotExist();
error LBAMM__InvalidPoolId();
error LBAMM__InvalidPoolType();
error LBAMM__InvalidPoolFeeBPS();
error LBAMM__InvalidPoolFeeHook();
error LBAMM__PoolFeeMustBeLessThan100Percent();
error LBAMM__CannotPairIdenticalTokens();
error LBAMM__PoolCreationWithNoCodeAtToken();
error LBAMM__PoolCreationWithLiquidityDidNotAddLiquidity();
error LBAMM__LiquidityDataDoesNotCallAddLiquidity();
error LBAMM__PoolHookDataNotSupported();
```

### Swap Errors
```solidity
error LBAMM__DeadlineExpired();
error LBAMM__RecipientCannotBeAddressZero();
error LBAMM__LimitAmountExceeded();
error LBAMM__LimitAmountNotMet();
error LBAMM__NoPoolsProvidedForMultiswap();
error LBAMM__ActualAmountCannotExceedInitialAmount();
error LBAMM__CannotPartialFillAfterFirstHop();
error LBAMM__PartialFillLessThanFees();
error LBAMM__PartialFillLessThanMinimumSpecified();
```

### Fee Errors
```solidity
error LBAMM__FeeAmountExceedsInputAmount();
error LBAMM__FeeAmountExceedsMaxFee();
error LBAMM__FeeRecipientCannotBeAddressZero();
error LBAMM__InsufficientInputForFees();
error LBAMM__InsufficientOutputForFees();
error LBAMM__InsufficientProtocolFee();
error LBAMM__InvalidFlashloanBPS();
error LBAMM__ExchangeFeeTransferFailed();
error LBAMM__FeeOnTopTransferFailed();
error LBAMM__ProtocolFeeTransferFailed();
error LBAMM__TransferHookFeeTransferFailed();
```

### Liquidity Errors
```solidity
error LBAMM__ExcessiveHookFees();
error LBAMM__ExcessiveLiquidityChange();
error LBAMM__InsufficientLiquidityChange();
```

### Flashloan Errors
```solidity
error LBAMM__FlashloansDisabled();
error LBAMM__FlashloanExecutionFailed();
error LBAMM__FlashloanFeeTokenCannotBeAddressZero();
error LBAMM__FeeTokenNotAllowedForFlashloan();
error LBAMM__CannotCollectFeesDuringFlashloan();
```

### Transfer & Value Errors
```solidity
error LBAMM__TokenInTransferFailed();
error LBAMM__TokenOutTransferFailed();
error LBAMM__TokenOwedTransferFailed();
error LBAMM__RefundFailed();
error LBAMM__InsufficientValue();
error LBAMM__ValueNotUsed();
error LBAMM__InputNotWrappedNative();
error LBAMM__InvalidTransferHandler();
```

### General Errors
```solidity
error LBAMM__ArrayLengthMismatch();
error LBAMM__BytesLengthExceedsMax();
error LBAMM__CallerMustBeSelf();
error LBAMM__HookFlagsMissingRequiredFlags();
error LBAMM__UnsupportedHookFlags();
error LBAMM__InvalidExtraDataArrayLength();
error LBAMM__Overflow();
error LBAMM__Underflow();
```
