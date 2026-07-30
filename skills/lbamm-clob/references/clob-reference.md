# CLOB Transfer Handler Reference

## Table of Contents

- [Overview](#overview)
- [Data Structures](#data-structures)
- [Key Generation](#key-generation)
- [Maker Flow](#maker-flow)
- [Taker Flow](#taker-flow)
- [Price Calculation](#price-calculation)
- [Order Book Traversal](#order-book-traversal)
- [CLOBQuotor](#clobquotor)
- [Hook System](#hook-system)
- [Events](#events)
- [Error Reference](#error-reference)
- [End-to-End Examples](#end-to-end-examples)
- [Security Considerations](#security-considerations)

---

## Overview

The CLOBTransferHandler implements an on-chain Central Limit Order Book. Makers deposit tokens and create limit orders at specific prices. Takers fill orders by executing swaps through the AMM with the CLOB as the transfer handler. Key properties:

- **Doubly-linked price list**: Prices are maintained in ascending order for efficient traversal
- **FIFO order buckets**: Orders at the same price are filled first-in-first-out
- **Input-based only**: Only input-based swaps (`amountSpecified >= 0`) are supported
- **Recipient must be handler**: `swapOrder.recipient` must be `address(this)` -- the handler collects output tokens to distribute to makers
- **Refund to executor**: Excess output tokens are refunded to the taker via a post-swap callback

---

## Data Structures

### OrderBookKey

Uniquely identifies an order book:

```solidity
struct OrderBookKey {
    address tokenIn;           // Input token for orders
    address tokenOut;          // Output token for orders
    address hook;              // Optional ICLOBHook for validation
    uint16 minimumOrderBase;   // Base for minimum order size
    uint8 minimumOrderScale;   // Scale for minimum order size
    // Minimum order = minimumOrderBase * 10^minimumOrderScale
}
```

### OrderBook

```solidity
struct OrderBook {
    uint160 currentPrice;                              // Best (lowest) price in the book
    mapping(uint160 => uint160) nextPriceAbove;        // Price linked list (ascending)
    mapping(uint160 => uint160) nextPriceBelow;        // Price linked list (descending)
    mapping(uint160 => OrderBucket) priceOrderBucket;  // Orders at each price level
}
```

### OrderBucket (orders at a specific price)

```solidity
struct OrderBucket {
    bytes32 currentOrderId;                        // Head of the order queue (storage slot pointer)
    uint256 inputAmountRemaining;                  // Remaining input on the head order
    mapping(bytes32 => bytes32) nextOrder;          // Order linked list (forward)
    mapping(bytes32 => bytes32) previousOrder;      // Order linked list (reverse)
    mapping(uint256 => Order) orders;               // Order data by nonce
}
```

### Order

```solidity
struct Order {
    address maker;          // Order creator
    uint256 orderNonce;     // Unique nonce (assigned by handler, auto-incrementing)
    uint256 inputAmount;    // Total input token amount (max: uint128.max, set to 0 when filled/closed)
}
```

### FillParams (taker-provided via transferExtraData)

```solidity
struct FillParams {
    bytes32 groupKey;           // Packed bytes32 of (hook, minimumOrderBase, minimumOrderScale)
    uint256 maxOutputSlippage;  // Max excess output tokens tolerated (reverts if exceeded)
    bytes hookData;             // Data for the ICLOBHook.validateExecutor
}
```

### HooksExtraData (used in maker operations)

```solidity
struct HooksExtraData {
    bytes tokenInHook;     // Data for tokenIn's validateHandlerOrder hook
    bytes tokenOutHook;    // Data for tokenOut's validateHandlerOrder hook
    bytes clobHook;        // Data for ICLOBHook.validateMaker
}
```

### FillCache (internal)

```solidity
struct FillCache {
    address tokenIn;
    address tokenOut;
    uint256 amountIn;
    uint256 amountOut;
}
```

---

## Key Generation

### Group Key

Packs the hook address, minimum order base, and minimum order scale into a single `bytes32`:

```solidity
bytes32 groupKey = clobHandler.generateGroupKey(hook, minimumOrderBase, minimumOrderScale);

// Internal packing: hook shifted left 96 bits | base shifted left 8 bits | scale
// key = bytes32(uint256(uint160(hook)) << 96) | bytes32(uint256(minimumOrderBase) << 8) | bytes32(uint256(minimumOrderScale))
```

Utility functions to decode:
- `getGroupKeyHook(groupKey)` -- returns the hook address
- `getGroupKeyMinimumOrder(groupKey)` -- returns `minimumOrderBase * 10^minimumOrderScale`
- `getGroupKeyMinimumOrderBase(groupKey)` -- returns the base value
- `getGroupKeyMinimumOrderScale(groupKey)` -- returns the scale value

### Order Book Key

Hashes tokenIn, tokenOut, and groupKey together using `EfficientHash`:

```solidity
bytes32 orderBookKey = clobHandler.generateOrderBookKey(tokenIn, tokenOut, groupKey);

// This is NOT keccak256(abi.encode(OrderBookKey)) -- the struct is decomposed into three bytes32 values
// orderBookKey = EfficientHash.efficientHash(
//     bytes32(uint256(uint160(tokenIn))),
//     bytes32(uint256(uint160(tokenOut))),
//     groupKey
// );
```

### Order Book Initialization

Order books are lazily initialized on first use. You can also explicitly initialize:

```solidity
clobHandler.initializeOrderBookKey(tokenIn, tokenOut, hook, minimumOrderBase, minimumOrderScale);
```

This stores the `OrderBookKey` struct in `orderBookKeys[key]` for public lookup and emits `OrderBookInitialized`.

---

## Maker Flow

### 1. Deposit Tokens

```solidity
// Maker must approve the CLOB handler first
IERC20(token).approve(address(clobHandler), amount);

// Deposit tokens to the CLOB
clobHandler.depositToken(token, amount);
// Tracked in: makerTokenBalance[token][msg.sender]
```

Reverts if `amount == 0` or transfer fails. Validates that the handler's balance increased by exactly `amount`.

### 2. Open Order

```solidity
uint256 orderNonce = clobHandler.openOrder(
    tokenIn,            // Input token address
    tokenOut,           // Output token address
    sqrtPriceX96,       // Order price in sqrtPriceX96 format
    orderAmount,        // Input token amount (max: uint128.max)
    groupKey,           // Packed group key from generateGroupKey()
    hintSqrtPriceX96,   // Hint for linked-list insertion (gas optimization)
    HooksExtraData({
        tokenInHook: "",    // Hook data for tokenIn's validateHandlerOrder
        tokenOutHook: "",   // Hook data for tokenOut's validateHandlerOrder
        clobHook: ""        // Hook data for ICLOBHook.validateMaker
    })
);
```

**Key behaviors:**
- If the maker has insufficient deposited balance, `openOrder` auto-collects via `transferFrom` for the deficit (maker must have approved the CLOB handler)
- Order amount must be >= group minimum (`minimumOrderBase * 10^minimumOrderScale`)
- Order amount capped at `uint128.max`
- Price must be within `MIN_SQRT_RATIO` and `MAX_SQRT_RATIO`
- `tokenIn` and `tokenOut` cannot be the same address
- The `hintSqrtPriceX96` helps the handler find the insertion point in the price linked list. Use the current price or a nearby price for best gas efficiency. Using 0 works but costs more gas.
- Token hooks' `validateHandlerOrder()` are called for BOTH tokenIn AND tokenOut independently (if the token has `TOKEN_SETTINGS_HANDLER_ORDER_VALIDATE_FLAG` set)
- If the group has an `ICLOBHook`, `validateMaker()` is called

### 3. Close Order

```solidity
clobHandler.closeOrder(
    tokenIn,        // Input token address
    tokenOut,       // Output token address
    sqrtPriceX96,   // Price level of the order
    orderNonce,     // The order's nonce (returned from openOrder)
    groupKey        // Packed group key
);
```

Removes the order from the book. The unfilled input amount is credited back to `makerTokenBalance[tokenIn][maker]`. Only the original maker can close their order.

### 4. Withdraw Tokens

```solidity
// Withdraw deposited or filled tokens
clobHandler.withdrawToken(token, amount);
```

Transfers tokens from the CLOB handler to `msg.sender` via ERC20 `safeTransfer`. Does not attempt to unwrap WNATIVE.

### Complete Maker Lifecycle

```
1. approve(clobHandler, amount)  -- one-time token approval
2. depositToken(token, amount)   -- or let openOrder auto-collect
3. openOrder(...)                -- place limit order, get orderNonce back
4. [wait for fills]              -- makerTokenBalance[tokenOut] increases as orders fill
5. closeOrder(...)               -- optional: cancel unfilled portion
6. withdrawToken(tokenIn, ...)   -- withdraw unfilled input tokens
7. withdrawToken(tokenOut, ...)  -- withdraw filled output tokens
```

---

## Taker Flow

Takers fill CLOB orders by executing AMM swaps with the CLOB as the transfer handler.

### 1. Encode FillParams

```solidity
FillParams memory params = FillParams({
    groupKey: groupKey,            // Must match the order book's group key
    maxOutputSlippage: 0,          // Max excess output (0 = no slippage tolerance)
    hookData: ""                   // Data for ICLOBHook.validateExecutor
});

bytes memory transferExtraData = abi.encode(params);
```

### 2. Build Transfer Data

```solidity
// IMPORTANT: Use bytes.concat, NOT abi.encode(address, bytes)
bytes memory transferData = bytes.concat(
    bytes32(uint256(uint160(address(clobHandler)))),
    transferExtraData
);
```

### 3. Build Swap Order

```solidity
SwapOrder memory swapOrder = SwapOrder({
    deadline: block.timestamp + 1 hours,
    recipient: address(clobHandler),     // MUST be the CLOB handler address
    amountSpecified: int256(amountIn),   // MUST be >= 0 (input-based only)
    minAmountSpecified: 0,               // Accept any partial fill
    limitAmount: minOutput,              // Minimum output from the AMM
    tokenIn: tokenIn,
    tokenOut: tokenOut
});
```

### 4. Execute Swap

```solidity
ILimitBreakAMM(amm).singleSwap(
    swapOrder,
    poolId,
    BPSFeeWithRecipient({recipient: address(0), BPS: 0}),
    FlatFeeWithRecipient({recipient: address(0), amount: 0}),
    SwapHooksExtraData({tokenInHook: "", tokenOutHook: "", poolHook: "", poolType: ""}),
    transferData
);
```

### What Happens During Fill

1. AMM calls `clobHandler.ammHandleTransfer()` with the swap amounts
2. CLOB validates: `msg.sender == AMM`, `recipient == address(this)`, `amountSpecified >= 0`
3. If the group has an `ICLOBHook`, `validateExecutor()` is called
4. `CLOBHelper.fillOrder()` traverses the order book from best price, filling orders FIFO
5. For each filled order, the maker's `makerTokenBalance[tokenOut]` is credited
6. If excess output remains and `fillOutputRemaining > maxOutputSlippage`, reverts
7. Handler transfers exactly `amountIn` of `tokenIn` to the AMM
8. Returns callback data encoding `afterSwapRefund(executor, tokenOut, excessAmount)`
9. AMM executes the callback -- excess output tokens go to the **executor** (taker)

---

## Price Calculation

Orders use `sqrtPriceX96` format (same as Uniswap V3):

```
sqrtPriceX96 = sqrt(amountOut / amountIn) * 2^96
```

### Converting Between Price and sqrtPriceX96

```solidity
// Q96 constant
uint256 constant Q96 = 2 ** 96;

// Price to sqrtPriceX96 (off-chain)
// sqrtPriceX96 = sqrt(price) * 2^96

// sqrtPriceX96 to price (off-chain)
// price = (sqrtPriceX96 / 2^96)^2
```

### Output Calculation (On-chain)

```solidity
// amountOut = amountIn * (sqrtPriceX96 / 2^96)^2
// Done in two steps with rounding up:
amountOut = FullMath.mulDivRoundingUp(amountIn, sqrtPriceX96, Q96);
amountOut = FullMath.mulDivRoundingUp(amountOut, sqrtPriceX96, Q96);
```

### Price Bounds

```solidity
uint160 constant MIN_SQRT_RATIO = 4_295_128_739;
uint160 constant MAX_SQRT_RATIO = 1_461_446_703_485_210_103_287_273_052_203_988_822_378_723_970_342;
```

---

## Order Book Traversal

The order book uses a doubly-linked list of price levels, with FIFO order queues at each level.

### Fill Order Logic

1. Start at `currentPrice` (the lowest/best price)
2. Consume the head order's remaining input
3. If the order is fully consumed, mark it filled (`inputAmount = 0`) and traverse to the next order
4. **Intra-bucket traversal**: If there's a next order at the same price, advance to it
5. **Inter-bucket traversal**: If the bucket is empty, move to `nextPriceAbove` and remove the empty price level from the linked list
6. Continue until all input is consumed or the book is exhausted

### Order IDs

Order IDs are the raw storage slot pointers of the `Order` struct. This is an internal implementation detail -- external code uses `orderNonce` to reference orders.

---

## CLOBQuotor

The CLOBQuotor reads order book state via static delegate calls through the CLOB handler's storage.

```solidity
CLOBQuotor quotor = CLOBQuotor(quotorAddress);

// Get the current best price for an order book
uint160 currentPrice = quotor.quoteGetCurrentPrice(orderBookKey);

// Get remaining input at a specific price level
uint256 remaining = quotor.quoteGetInputAmountRemaining(orderBookKey, sqrtPriceX96);
```

The quotor uses `StaticDelegateCall` to access the handler's storage without modifying state.

---

## Hook System

### ICLOBHook

The CLOB hook interface extends `ITransferHandlerExecutorValidation` and adds maker validation:

```solidity
interface ICLOBHook is ITransferHandlerExecutorValidation {
    /// @notice Validates the maker of an order. MUST revert to prevent order creation.
    function validateMaker(
        bytes32 orderBookKey,
        address depositor,
        uint160 sqrtPriceX96,
        uint256 orderAmount,
        bytes calldata hookData     // From HooksExtraData.clobHook
    ) external;

    // Inherited from ITransferHandlerExecutorValidation:
    // function validateExecutor(bytes32 handlerId, address executor, ...) external;
}
```

### Token Hooks

When a maker opens an order, token hooks' `validateHandlerOrder()` are called for BOTH tokenIn and tokenOut independently, if the respective token has `TOKEN_SETTINGS_HANDLER_ORDER_VALIDATE_FLAG` set in the AMM's token settings:

```solidity
ILimitBreakAMMTokenHook(tokenHook).validateHandlerOrder(
    maker,           // msg.sender (the maker)
    isTokenIn,       // true for tokenIn, false for tokenOut
    tokenIn,
    tokenOut,
    orderAmount,     // Input token amount
    amountOut,       // Calculated output via calculateFixedInput
    handlerOrderParams,  // abi.encode(orderBookKey, sqrtPriceX96)
    hookData         // From HooksExtraData.tokenInHook or .tokenOutHook
);
```

---

## Events

```solidity
event TokenDeposited(address indexed token, address indexed depositor, uint256 amount);
event TokenWithdrawn(address indexed token, address indexed depositor, uint256 amount);
event OrderBookInitialized(bytes32 indexed orderBookKey, address tokenIn, address tokenOut, address hook, uint16 minimumOrderBase, uint8 minimumOrderScale);
event OrderOpened(address indexed maker, bytes32 indexed orderBookKey, uint256 orderAmount, uint160 sqrtPriceX96, uint256 orderNonce);
event OrderClosed(address indexed maker, bytes32 indexed orderBookKey, uint256 unfilledInputAmount, uint256 orderNonce);
event OrderBookFill(bytes32 indexed orderBookKey, uint256 endingOrderNonce, uint256 endingOrderInputRemaining);
```

---

## Error Reference

| Error | Cause |
|---|---|
| `CLOBTransferHandler__CallbackMustBeFromAMM` | `msg.sender` is not the AMM |
| `CLOBTransferHandler__CannotPairIdenticalTokens` | `tokenIn == tokenOut` in openOrder |
| `CLOBTransferHandler__FillOutputExceedsMaxSlippage` | Excess output > `maxOutputSlippage` |
| `CLOBTransferHandler__GroupMinimumCannotBeZero` | `minimumOrderBase` is 0 |
| `CLOBTransferHandler__HandlerMustBeRecipient` | `swapOrder.recipient` is not the CLOB handler |
| `CLOBTransferHandler__InsufficientInputToFill` | Order book exhausted before input consumed |
| `CLOBTransferHandler__InsufficientMakerBalance` | Maker balance too low and transferFrom failed |
| `CLOBTransferHandler__InsufficientOutputToFill` | Output from AMM insufficient to cover CLOB fills |
| `CLOBTransferHandler__InvalidDataLength` | `transferExtraData` is empty |
| `CLOBTransferHandler__InvalidMaker` | Caller is not the order's maker (closeOrder) |
| `CLOBTransferHandler__InvalidNativeTransfer` | Native ETH sent from non-WNATIVE address |
| `CLOBTransferHandler__InvalidPrice` | Current price is 0 or max (empty book) |
| `CLOBTransferHandler__InvalidSqrtPriceX96` | Price outside MIN/MAX bounds |
| `CLOBTransferHandler__InvalidTransferAmount` | Balance check failed after transfer (fee-on-transfer tokens) |
| `CLOBTransferHandler__MinimumOrderScaleExceedsMaximum` | Scale > 72 |
| `CLOBTransferHandler__OrderAmountExceedsMax` | Amount > `uint128.max` |
| `CLOBTransferHandler__OrderAmountLessThanGroupMinimum` | Amount < `minimumOrderBase * 10^minimumOrderScale` |
| `CLOBTransferHandler__OrderInvalidFilledOrClosed` | Order already filled or closed |
| `CLOBTransferHandler__OutputBasedNotAllowed` | `amountSpecified < 0` in swap |
| `CLOBTransferHandler__TransferFailed` | ERC20 transfer failed |
| `CLOBTransferHandler__ZeroDepositAmount` | Deposit amount is 0 |
| `CLOBTransferHandler__ZeroOrderAmount` | Order amount is 0 |
| `CLOBTransferHandler__ZeroWithdrawAmount` | Withdraw amount is 0 |

---

## End-to-End Examples

### Maker: Deposit and Open Order

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import "forge-std/Script.sol";
import {CLOBTransferHandler} from
    "@limitbreak/lbamm-hooks-and-handlers/src/handlers/clob/CLOBTransferHandler.sol";
import {HooksExtraData} from
    "@limitbreak/lbamm-hooks-and-handlers/src/handlers/clob/DataTypes.sol";
import {IERC20} from "forge-std/interfaces/IERC20.sol";

contract MakerOpenOrderScript is Script {
    function run() external {
        address clobHandler = vm.envAddress("CLOB_HANDLER_ADDRESS");
        address tokenIn = vm.envAddress("TOKEN_IN");
        address tokenOut = vm.envAddress("TOKEN_OUT");
        uint256 orderAmount = 1000e18; // 1000 tokens
        uint160 sqrtPriceX96 = 79228162514264337593543950336; // price = 1.0

        // 1. Generate group key (no hook, min order 1e18)
        bytes32 groupKey = CLOBTransferHandler(clobHandler).generateGroupKey(
            address(0),  // no hook
            1,           // minimumOrderBase
            18           // minimumOrderScale (1 * 10^18)
        );

        // 2. Approve and deposit (or let openOrder auto-collect)
        vm.startBroadcast();
        IERC20(tokenIn).approve(clobHandler, orderAmount);

        // 3. Open order
        uint256 orderNonce = CLOBTransferHandler(clobHandler).openOrder(
            tokenIn,
            tokenOut,
            sqrtPriceX96,
            orderAmount,
            groupKey,
            0,  // hint price (0 = no hint, costs more gas)
            HooksExtraData("", "", "")
        );

        vm.stopBroadcast();

        console.log("Order opened with nonce:", orderNonce);
    }
}
```

### Maker: Close Order and Withdraw

```solidity
contract MakerCloseOrderScript is Script {
    function run() external {
        address clobHandler = vm.envAddress("CLOB_HANDLER_ADDRESS");
        address tokenIn = vm.envAddress("TOKEN_IN");
        address tokenOut = vm.envAddress("TOKEN_OUT");
        uint160 sqrtPriceX96 = 79228162514264337593543950336;
        uint256 orderNonce = vm.envUint("ORDER_NONCE");

        bytes32 groupKey = CLOBTransferHandler(clobHandler).generateGroupKey(
            address(0), 1, 18
        );

        vm.startBroadcast();

        // Close the order (unfilled input credited to makerTokenBalance)
        CLOBTransferHandler(clobHandler).closeOrder(
            tokenIn, tokenOut, sqrtPriceX96, orderNonce, groupKey
        );

        // Withdraw unfilled input tokens
        uint256 inputBalance = CLOBTransferHandler(clobHandler)
            .makerTokenBalance(tokenIn, msg.sender);
        if (inputBalance > 0) {
            CLOBTransferHandler(clobHandler).withdrawToken(tokenIn, inputBalance);
        }

        // Withdraw any filled output tokens
        uint256 outputBalance = CLOBTransferHandler(clobHandler)
            .makerTokenBalance(tokenOut, msg.sender);
        if (outputBalance > 0) {
            CLOBTransferHandler(clobHandler).withdrawToken(tokenOut, outputBalance);
        }

        vm.stopBroadcast();
    }
}
```

### Taker: Fill CLOB Orders via AMM Swap

```solidity
import {ILimitBreakAMM} from "@limitbreak/lbamm-core/src/interfaces/core/ILimitBreakAMM.sol";
import {SwapOrder} from "@limitbreak/lbamm-core/src/DataTypes.sol";
import {FillParams} from
    "@limitbreak/lbamm-hooks-and-handlers/src/handlers/clob/DataTypes.sol";

contract TakerFillScript is Script {
    function run() external {
        address amm = vm.envAddress("AMM_ADDRESS");
        address clobHandler = vm.envAddress("CLOB_HANDLER_ADDRESS");
        address tokenIn = vm.envAddress("TOKEN_IN");
        address tokenOut = vm.envAddress("TOKEN_OUT");

        bytes32 groupKey = CLOBTransferHandler(clobHandler).generateGroupKey(
            address(0), 1, 18
        );

        // 1. Encode FillParams as transferExtraData
        bytes memory transferExtraData = abi.encode(FillParams({
            groupKey: groupKey,
            maxOutputSlippage: 0,   // No tolerance for excess output
            hookData: ""
        }));

        // 2. Build transfer data (bytes.concat, NOT abi.encode)
        bytes memory transferData = bytes.concat(
            bytes32(uint256(uint160(clobHandler))),
            transferExtraData
        );

        // 3. Build swap order
        SwapOrder memory swapOrder = SwapOrder({
            deadline: block.timestamp + 1 hours,
            recipient: clobHandler,          // MUST be the CLOB handler
            amountSpecified: int256(100e18), // Input-based only (>= 0)
            minAmountSpecified: 0,           // Accept any partial fill
            limitAmount: 0,                  // Minimum output from AMM
            tokenIn: tokenIn,
            tokenOut: tokenOut
        });

        // 4. Execute
        vm.startBroadcast();
        IERC20(tokenIn).approve(amm, uint256(swapOrder.amountSpecified));
        ILimitBreakAMM(amm).singleSwap(
            swapOrder,
            vm.envBytes32("POOL_ID"),
            BPSFeeWithRecipient({recipient: address(0), BPS: 0}),
            FlatFeeWithRecipient({recipient: address(0), amount: 0}),
            SwapHooksExtraData({tokenInHook: "", tokenOutHook: "", poolHook: "", poolType: ""}),
            transferData
        );
        vm.stopBroadcast();

        // Excess output (if any) is automatically refunded to executor via afterSwapRefund callback
    }
}
```

---

## Security Considerations

1. **Input-based only**: The CLOB only supports `amountSpecified >= 0`. Output-based swaps revert with `OutputBasedNotAllowed`.
2. **Recipient must be handler**: `swapOrder.recipient` must be `address(this)` -- the handler collects output tokens from the AMM to distribute to makers.
3. **Minimum orders**: `minimumOrderBase * 10^minimumOrderScale` prevents dust orders that waste gas during traversal. Maximum scale is 72.
4. **Order amount cap**: Orders are limited to `uint128.max` in input amount.
5. **Refund goes to executor**: After fills, excess output tokens are refunded to the **executor** (taker) via the callback, not to makers. Makers receive their share credited to `makerTokenBalance` during the fill.
6. **FIFO fairness**: Orders at the same price level are filled in order of creation (FIFO).
7. **Reentrancy protection**: Uses transient-storage-based reentrancy guard (`TstorishReentrancyGuard`).
8. **Fee-on-transfer tokens**: Deposits validate `balanceAfter - balanceBefore == amount`, so fee-on-transfer tokens will revert.
9. **WNATIVE handling**: `afterSwapRefund` attempts to unwrap WNATIVE to native ETH before falling back to ERC20 transfer.
10. **Hook validation**: `ICLOBHook` provides both maker validation (`validateMaker`) and executor validation (`validateExecutor`). Token hooks provide `validateHandlerOrder` for both tokenIn and tokenOut.
