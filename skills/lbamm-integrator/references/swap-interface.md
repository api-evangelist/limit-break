# LBAMM Swap Interface Reference

## singleSwap

Executes a swap in a single pool.

```solidity
function singleSwap(
    SwapOrder calldata swapOrder,
    bytes32 poolId,
    BPSFeeWithRecipient calldata exchangeFee,
    FlatFeeWithRecipient calldata feeOnTop,
    SwapHooksExtraData calldata swapHooksExtraData,
    bytes calldata transferData
) external payable returns (uint256 amountIn, uint256 amountOut);
```

### SwapOrder Construction

```solidity
SwapOrder memory order = SwapOrder({
    deadline: block.timestamp + 300,  // 5 minute deadline
    recipient: msg.sender,            // Who receives output
    amountSpecified: int256(1e18),    // Positive = swap by input
                                      // Negative = swap by output
    minAmountSpecified: 0,            // Min for partial fills (0 = accept any partial fill amount)
    limitAmount: 0,                   // Min output (input swap) / max input (output swap)
    tokenIn: WETH,
    tokenOut: USDC
});
```

### Input Swap vs Output Swap

| | Input Swap | Output Swap |
|---|---|---|
| `amountSpecified` | Positive (swap by input) | Negative (swap by output) |
| `limitAmount` | Minimum output | Maximum input |
| Fee deduction | From input before swap | Added to required input after swap |

### Fee Structs

```solidity
// Exchange fee (BPS-based, on tokenIn)
BPSFeeWithRecipient memory exchangeFee = BPSFeeWithRecipient({
    recipient: feeCollector,
    BPS: 50  // 0.5%
});

// Fee-on-top (flat amount, on tokenIn)
FlatFeeWithRecipient memory feeOnTop = FlatFeeWithRecipient({
    recipient: feeCollector,
    amount: 1e15  // 0.001 tokens
});

// No fees:
BPSFeeWithRecipient memory noExchangeFee = BPSFeeWithRecipient({recipient: address(0), BPS: 0});
FlatFeeWithRecipient memory noFeeOnTop = FlatFeeWithRecipient({recipient: address(0), amount: 0});
```

### Hook Extra Data

```solidity
SwapHooksExtraData memory hookData = SwapHooksExtraData({
    tokenInHook: "",   // Data for tokenIn's hook
    tokenOutHook: "",  // Data for tokenOut's hook
    poolHook: "",      // Data for pool hook
    poolType: ""       // Data for pool type
});
```

### Transfer Data

For standard ERC20 swaps (no handler): `transferData = ""`

For transfer handler swaps:
```solidity
// IMPORTANT: Use bytes.concat, NOT abi.encode(address, bytes)
// The AMM reads first 32 bytes as address, then passes transferData[32:] to handler
bytes memory transferData = bytes.concat(
    bytes32(uint256(uint160(address(handler)))),  // 32 bytes: handler address (left-padded)
    handlerExtraData                               // Raw handler-specific data
);
```

## multiSwap

Executes a multi-hop swap across multiple pools.

```solidity
function multiSwap(
    SwapOrder calldata swapOrder,
    bytes32[] calldata poolIds,           // Ordered list of pool IDs
    BPSFeeWithRecipient calldata exchangeFee,
    FlatFeeWithRecipient calldata feeOnTop,
    SwapHooksExtraData[] calldata swapHooksExtraDatas,  // One per hop
    bytes calldata transferData
) external payable returns (uint256 amountIn, uint256 amountOut);
```

### Multi-Hop Rules
- `poolIds.length == swapHooksExtraDatas.length`
- Intermediate tokens are inferred from pool pairs
- Exchange fee + fee-on-top apply once (not per hop)
- LP fees apply per hop

### Example: A → B → C Route
```solidity
bytes32[] memory poolIds = new bytes32[](2);
poolIds[0] = poolAB;  // A/B pool
poolIds[1] = poolBC;  // B/C pool

SwapOrder memory order = SwapOrder({
    deadline: block.timestamp + 300,
    recipient: msg.sender,
    amountSpecified: int256(1e18),
    minAmountSpecified: 0,
    limitAmount: 0,
    tokenIn: tokenA,
    tokenOut: tokenC
});
```

## directSwap

Peer-to-peer swap at a fixed rate without using pool liquidity.

```solidity
function directSwap(
    SwapOrder calldata swapOrder,
    DirectSwapParams calldata directSwapParams,
    BPSFeeWithRecipient calldata exchangeFee,
    FlatFeeWithRecipient calldata feeOnTop,
    SwapHooksExtraData calldata swapHooksExtraData,
    bytes calldata transferData
) external payable returns (uint256 amountIn, uint256 amountOut);
```

```solidity
DirectSwapParams memory directParams = DirectSwapParams({
    swapAmount: 1e18,       // Exact swap amount
    maxAmountOut: 3000e6,   // Max output supplied by executor
    minAmountIn: 1e18       // Min input received by executor
});
```

### Direct Swap Notes
- The executor provides output tokens to the recipient
- The executor receives input tokens
- No pool state is affected
- Pool hooks are NOT called (only token hooks)
- Commonly used with permit/CLOB transfer handlers

## Native Token (ETH) Handling

- Send ETH as `msg.value` when `tokenIn` is the wrapped native token
- The AMM will wrap ETH automatically
- For any swap where `tokenOut` is the wrapped native token, the AMM unwraps and sends ETH

```solidity
// Swap ETH for USDC:
amm.singleSwap{value: 1 ether}(
    SwapOrder({
        tokenIn: WRAPPED_NATIVE,
        tokenOut: USDC,
        amountSpecified: int256(1 ether),
        ...
    }),
    poolId, noFee, noFee, emptyHookData, ""
);
```
