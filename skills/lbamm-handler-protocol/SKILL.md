---
name: lbamm-handler-protocol
description: LBAMM transfer handler architecture -- how transfer handlers work during swap settlement, the ILimitBreakAMMTransferHandler interface, executor validation hooks, callback patterns, and custom handler development. Use this skill whenever the user asks about LBAMM transfer handlers, swap settlement, ammHandleTransfer, custom settlement logic, ITransferHandlerExecutorValidation, how swaps use transfer data, or writing a custom handler contract.
user-invocable: false
---

# LBAMM Transfer Handler Protocol Knowledge Base

Transfer handlers are modular settlement components in LBAMM. During swap finalization, instead of using standard `transferFrom` to acquire input tokens, the AMM delegates token acquisition to a handler contract. This enables gasless swaps (permits), on-chain order books (CLOB), RFQ systems, and any custom settlement logic.

## How Handlers Work

1. User calls `singleSwap()` / `multiSwap()` / `directSwap()` with a non-empty `transferData` parameter (at least 32 bytes)
2. The AMM reads the first 32 bytes as the handler address (left-padded `bytes32`), validates the top 96 bits are zero
3. The remaining bytes (`transferData[32:]`) are passed as `transferExtraData` to the handler
4. AMM computes swap amounts (amountIn, amountOut) via the pool type
5. AMM calls `handler.ammHandleTransfer()` with the swap context
6. The handler transfers **exactly** `amountIn` input tokens to the AMM
7. AMM verifies via strict balance check: `balanceBefore + amountIn == balanceAfter`
8. AMM transfers tokenOut to recipient and settles hook fees
9. If handler returned non-empty `callbackData`, AMM executes it on the handler via low-level `call`

## Transfer Data Encoding

```solidity
// Use bytes.concat, NOT abi.encode
bytes memory transferData = bytes.concat(
    bytes32(uint256(uint160(address(handler)))),  // 32 bytes: handler address
    handlerExtraData                               // Raw handler-specific bytes
);
```

If `transferData.length < 32`, no handler is used (standard safeTransferFrom).

## Core Interfaces

### ILimitBreakAMMTransferHandler

```solidity
interface ILimitBreakAMMTransferHandler {
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

### ITransferHandlerExecutorValidation

Optional hook that handlers can use to validate who can execute/fill orders.

```solidity
interface ITransferHandlerExecutorValidation {
    function validateExecutor(
        bytes32 handlerId,
        address executor,
        SwapOrder calldata swapOrder,
        uint256 amountIn,
        uint256 amountOut,
        BPSFeeWithRecipient calldata exchangeFee,
        FlatFeeWithRecipient calldata feeOnTop,
        bytes calldata hookData
    ) external;  // Revert to block execution
}
```

## Deployed Handlers

| Handler | Purpose | Skill |
|---|---|---|
| **PermitTransferHandler** | Gasless swaps via EIP-712 signed permits with optional cosigner | `lbamm-permit-handler` |
| **CLOBTransferHandler** | On-chain limit order book with maker deposits and FIFO fills | `lbamm-clob` |

## Repository

GitHub: `https://github.com/limitbreakinc/lbamm-hooks-and-handlers`

### Project Setup

1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **LBAMM Core**: `lib/lbamm-core/` must exist. If not: `forge install limitbreakinc/lbamm-core`
3. **LBAMM Hooks and Handlers**: `lib/lbamm-hooks-and-handlers/` must exist. If not: `forge install limitbreakinc/lbamm-hooks-and-handlers`
4. **Remappings**: `remappings.txt` needs both `@limitbreak/lbamm-core/=lib/lbamm-core/` and `@limitbreak/lbamm-hooks-and-handlers/=lib/lbamm-hooks-and-handlers/`. Append if missing -- do not overwrite existing entries.

## Reference Files

- `references/handler-architecture.md` -- Handler invocation flow, callback pattern, transfer data encoding
- `references/custom-handler-patterns.md` -- Example custom handler implementations (RFQ, whitelists, fee-gated)

## Related Skills

- **lbamm-permit-handler** -- Generate integration code for the PermitTransferHandler (gasless swaps)
- **lbamm-clob** -- Generate integration code for the CLOBTransferHandler (limit orders)
- **lbamm-integrator** -- Generate router contracts that use handlers via directSwap
- **lbamm-test** -- Generate Foundry test suites for handler contracts
- **lbamm-protocol** -- Core AMM architecture, pool types, hooks, fees
