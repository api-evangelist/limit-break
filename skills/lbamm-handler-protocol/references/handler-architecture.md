# Transfer Handler Architecture

## Handler Invocation Flow

```
User --> singleSwap/multiSwap/directSwap(swapOrder, ..., transferData)
         |
         +-- transferData = bytes.concat(bytes32(uint256(uint160(handler))), extraData)
         |
         +-- AMM reads first 32 bytes as handler address, validates top 96 bits are zero
         |
         +-- AMM computes swap amounts (amountIn, amountOut)
         |
         +-- AMM calls handler.ammHandleTransfer(executor, swapOrder, amountIn, amountOut, ...)
         |     |   with transferData[32:] as transferExtraData
         |     |
         |     +-- Handler transfers EXACTLY amountIn of tokenIn to AMM
         |     +-- Returns callbackData (must include 4-byte selector if non-empty)
         |
         +-- AMM verifies exact balance: balanceBefore + amountIn == balanceAfter
         +-- AMM transfers tokenOut to recipient, settles hook fees
         |
         +-- If callbackData is non-empty:
               AMM executes callbackData on handler via low-level call()
```

## ILimitBreakAMMTransferHandler

Source: `lbamm-core/src/interfaces/ILimitBreakAMMTransferHandler.sol`

```solidity
interface ILimitBreakAMMTransferHandler {
    event TransferHandlerManifestUriUpdated(string uri);

    /// @notice Called during swap finalization to handle input token transfer
    /// @param executor          The swap executor (msg.sender of the swap call)
    /// @param swapOrder         Original swap order parameters
    /// @param amountIn          Input tokens required
    /// @param amountOut         Output tokens that will be sent to recipient
    /// @param exchangeFee       Exchange fee configuration
    /// @param feeOnTop          Additional flat fee configuration
    /// @param transferExtraData Arbitrary calldata from the swap caller
    /// @return callbackData     ABI-encoded callback to execute after swap finalization
    function ammHandleTransfer(
        address executor,
        SwapOrder calldata swapOrder,
        uint256 amountIn,
        uint256 amountOut,
        BPSFeeWithRecipient calldata exchangeFee,
        FlatFeeWithRecipient calldata feeOnTop,
        bytes calldata transferExtraData
    ) external returns (bytes memory callbackData);

    function transferHandlerManifestUri() external view returns (string memory manifestUri);
}
```

## ITransferHandlerExecutorValidation

Source: `lbamm-hooks-and-handlers/src/handlers/interfaces/ITransferHandlerExecutorValidation.sol`

An optional validation hook that transfer handlers reference to validate who can execute/fill orders.

```solidity
interface ITransferHandlerExecutorValidation {
    /// @notice Validates the executor of a swap through a transfer handler
    /// @dev MUST revert to prevent execution
    function validateExecutor(
        bytes32 handlerId,     // Handler-specific identifier (e.g., permit hash, order book key)
        address executor,       // The swap executor
        SwapOrder calldata swapOrder,
        uint256 amountIn,
        uint256 amountOut,
        BPSFeeWithRecipient calldata exchangeFee,
        FlatFeeWithRecipient calldata feeOnTop,
        bytes calldata hookData
    ) external;
}
```

## Transfer Data Encoding

```solidity
// IMPORTANT: Use bytes.concat, NOT abi.encode(address, bytes)
// The AMM reads the first 32 bytes via calldataload, then slices transferData[32:] as raw bytes
bytes memory transferData = bytes.concat(
    bytes32(uint256(uint160(address(transferHandler)))),  // 32 bytes: handler address (left-padded)
    handlerExtraData                                       // Raw handler-specific bytes
);

// If transferData.length < 32, no handler is used (standard safeTransferFrom)
```

## Callback Data Pattern

The `callbackData` returned by `ammHandleTransfer()` is raw calldata (including the 4-byte function selector) that the AMM executes on the handler via low-level `call` after the swap is finalized. This enables:

- **Refunding excess tokens** (e.g., CLOB refunds excess output to executor)
- **Post-swap accounting** (e.g., nonce invalidation, order state cleanup)

If `callbackData` is empty, no callback is made.

```solidity
// Example: encoding a callback
return abi.encodeWithSelector(
    MyHandler.afterSwapCleanup.selector,
    executor,
    someParam
);
```

## Writing a Custom Handler

A custom handler must:

1. Implement `ILimitBreakAMMTransferHandler`
2. Validate `msg.sender == AMM` in `ammHandleTransfer`
3. Transfer **exactly** `amountIn` tokens to the AMM (strict balance check)
4. Return callback data if post-swap logic is needed (empty bytes otherwise)
5. Implement `transferHandlerManifestUri()` (can return empty string)

### Minimal Template

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {ILimitBreakAMMTransferHandler} from "@limitbreak/lbamm-core/src/interfaces/ILimitBreakAMMTransferHandler.sol";
import {SwapOrder, BPSFeeWithRecipient, FlatFeeWithRecipient} from "@limitbreak/lbamm-core/src/DataTypes.sol";

contract MyHandler is ILimitBreakAMMTransferHandler {
    error MyHandler__CallerNotAMM();

    address public immutable AMM;

    constructor(address amm_) { AMM = amm_; }

    function ammHandleTransfer(
        address executor,
        SwapOrder calldata swapOrder,
        uint256 amountIn,
        uint256 amountOut,
        BPSFeeWithRecipient calldata exchangeFee,
        FlatFeeWithRecipient calldata feeOnTop,
        bytes calldata transferExtraData
    ) external returns (bytes memory) {
        if (msg.sender != AMM) revert MyHandler__CallerNotAMM();

        // Decode handler-specific data from transferExtraData
        // ...

        // Transfer EXACTLY amountIn to the AMM
        // IERC20(swapOrder.tokenIn).transferFrom(source, msg.sender, amountIn);

        return "";  // No callback needed
    }

    function transferHandlerManifestUri() external pure returns (string memory) { return ""; }
}
```

## Security Considerations

1. **msg.sender validation**: Always verify `msg.sender == AMM` in `ammHandleTransfer`.
2. **Exact amount**: Transfer exactly `amountIn` -- the AMM checks `balanceBefore + amountIn == balanceAfter` (strict equality, not >=).
3. **Replay protection**: Use nonces or salts to prevent signature replay in signature-based handlers.
4. **Callback safety**: Callbacks execute after swap finalization. Keep them simple, bounded, and reentrant-safe.
5. **Token approvals**: Handlers need token approvals from makers. Use PermitC or direct approvals.
