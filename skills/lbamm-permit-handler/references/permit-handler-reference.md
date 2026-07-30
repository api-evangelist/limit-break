# Permit Transfer Handler Reference

## Table of Contents

- [Overview](#overview)
- [Permit Types and Structs](#permit-types-and-structs)
- [Transfer Data Encoding](#transfer-data-encoding)
- [EIP-712 Signing](#eip-712-signing)
- [Cosigner System](#cosigner-system)
- [Fill-or-Kill Validation](#fill-or-kill-validation)
- [Partial Fill Validation](#partial-fill-validation)
- [Hook Validation](#hook-validation)
- [End-to-End Foundry Example](#end-to-end-foundry-example)
- [Error Reference](#error-reference)
- [Security Considerations](#security-considerations)

---

## Overview

The PermitTransferHandler enables gasless swap execution. A maker signs a PermitC permit offline, and an executor fills it on-chain by passing the permit data through the AMM swap call. The handler:

1. Decodes the permit type from the first byte of `transferExtraData`
2. Validates the permit (fill constraints, cosignature, hook)
3. Calls PermitC to execute the token transfer
4. Transfers exactly `amountIn` tokens to the AMM

**EIP-712 Domain:** `name: "PermitTransferHandler", version: "1"`

---

## Permit Types and Structs

### FillOrKillPermitTransfer

Atomic, all-or-nothing execution. The entire swap amount must be filled.

```solidity
struct FillOrKillPermitTransfer {
    address permitProcessor;        // PermitC contract address
    address from;                   // Permit signer (token owner)
    uint256 nonce;                  // Unique nonce (consumed by PermitC on use)
    uint256 permitAmount;           // Exact amount to transfer
    uint256 expiration;             // Permit expiry timestamp
    bytes signature;                // EIP-712 signature from `from` (PermitC domain)
    address cosigner;               // Optional cosigner (address(0) = no cosigner)
    uint256 cosignatureExpiration;  // Cosignature expiry timestamp
    bytes cosignature;              // Cosignature (PermitTransferHandler domain)
    address hook;                   // Optional ITransferHandlerExecutorValidation hook
    bytes hookData;                 // Data passed to the validation hook
}
```

### PartialFillPermitTransfer

Supports incremental fills across multiple executions.

```solidity
struct PartialFillPermitTransfer {
    address permitProcessor;        // PermitC contract address
    address from;                   // Permit signer
    uint256 salt;                   // Unique salt (PermitC tracks cumulative fill state)
    int256 permitAmountSpecified;   // Positive = input swap, negative = output swap
    uint256 permitLimitAmount;      // For input swaps: total input. For output swaps: max input
    uint256 expiration;             // Permit expiry timestamp
    bytes signature;                // EIP-712 signature from `from` (PermitC domain)
    address cosigner;               // Optional cosigner
    uint256 cosignatureExpiration;  // Cosignature expiry
    uint256 cosignatureNonce;       // 0 = reusable, >0 = consumed on-chain via bitmap
    bytes cosignature;              // Cosignature (PermitTransferHandler domain)
    address hook;                   // Optional validation hook
    bytes hookData;                 // Data for the validation hook
}
```

### Permit Type Constants

```solidity
bytes1 constant FILL_OR_KILL_PERMIT = 0x00;
bytes1 constant PARTIAL_FILL_PERMIT = 0x01;
```

---

## Transfer Data Encoding

```solidity
// Step 1: Build handler-specific extra data
// First byte is the permit type discriminator, then ABI-encoded struct
bytes memory handlerExtraData = bytes.concat(
    bytes1(0x00),                    // FILL_OR_KILL_PERMIT (or 0x01 for partial)
    abi.encode(permitData)           // ABI-encoded struct
);

// Step 2: Prepend the handler address (32 bytes, left-padded)
bytes memory transferData = bytes.concat(
    bytes32(uint256(uint160(address(permitTransferHandler)))),
    handlerExtraData
);

// Pass transferData to singleSwap/multiSwap/directSwap
```

The AMM reads bytes 0-31 as the handler address, then passes bytes 32+ as `transferExtraData`. The handler reads byte 0 of `transferExtraData` as the permit type, then ABI-decodes bytes 1+ as the struct.

---

## EIP-712 Signing

### Two-Domain Model

The PermitTransferHandler uses **two separate EIP-712 domains**:

| Signature | Signer | Domain Contract | Domain Name |
|---|---|---|---|
| Permit signature | Maker | PermitC contract | Varies (see `permitc-protocol`) |
| Cosignature | Cosigner | PermitTransferHandler | `"PermitTransferHandler"` |

### Swap Typehash (Additional Data)

The maker's permit signature includes swap parameters as additional data:

```solidity
bytes32 constant SWAP_TYPEHASH = keccak256(
    "Swap(bool partialFill,address recipient,int256 amountSpecified,"
    "uint256 limitAmount,address tokenOut,address exchangeFeeRecipient,"
    "uint16 exchangeFeeBPS,address cosigner,address hook)"
);
```

### Additional Data Hash Construction

**Fill-or-kill:**
```solidity
additionalDataHash = hash(
    SWAP_TYPEHASH,
    false,                              // partialFill = false
    swapOrder.recipient,
    swapOrder.amountSpecified,
    swapOrder.limitAmount,
    swapOrder.tokenOut,
    exchangeFee.recipient,
    exchangeFee.BPS,
    permitData.cosigner,
    permitData.hook
);
```

**Partial fill:**
```solidity
additionalDataHash = hash(
    SWAP_TYPEHASH,
    true,                               // partialFill = true
    swapOrder.recipient,
    permitData.permitAmountSpecified,    // NOT swapOrder.amountSpecified
    permitData.permitLimitAmount,        // NOT swapOrder.limitAmount
    swapOrder.tokenOut,
    exchangeFee.recipient,
    exchangeFee.BPS,
    permitData.cosigner,
    permitData.hook
);
```

Key difference: partial fill uses the permit's own amount/limit values instead of the swap order's, because each fill may have different swap parameters.

### PermitC Type Strings

**Fill-or-kill** uses `PermitTransferFromWithAdditionalData`:
```
"PermitTransferFromWithAdditionalData(uint256 tokenType,address token,uint256 id,
uint256 amount,uint256 nonce,address operator,uint256 expiration,uint256 masterNonce,
Swap swapData)Swap(bool partialFill,address recipient,int256 amountSpecified,
uint256 limitAmount,address tokenOut,address exchangeFeeRecipient,uint16 exchangeFeeBPS,
address cosigner,address hook)"
```

**Partial fill** uses `PermitOrderWithAdditionalData`:
```
"PermitOrderWithAdditionalData(uint256 tokenType,address token,uint256 id,
uint256 amount,uint256 salt,address operator,uint256 expiration,uint256 masterNonce,
Swap swapData)Swap(bool partialFill,address recipient,int256 amountSpecified,
uint256 limitAmount,address tokenOut,address exchangeFeeRecipient,uint16 exchangeFeeBPS,
address cosigner,address hook)"
```

Note the difference: `nonce` for fill-or-kill vs `salt` for partial fill.

### What Is NOT Signed

`feeOnTop` (flat fee) is NOT part of the signed data. Only `exchangeFee` (recipient + BPS) is signed. This means an executor can set `feeOnTop` without invalidating the maker's signature.

---

## Cosigner System

### Purpose

The cosigner adds a second authorization layer. Common uses:
- **MEV protection**: Cosigner validates execution conditions before co-signing
- **Order book servers**: Cosigner controls which executors can fill which orders
- **Time-limited authorization**: Cosignature has its own expiration

### Cosignature Typehash

```solidity
bytes32 constant COSIGNATURE_TYPEHASH = keccak256(
    "Cosignature(bytes permitSignature,uint256 cosignatureExpiration,"
    "uint256 cosignatureNonce,address executor)"
);
```

The cosignature signs over: the **hash of the permit signature** (not the raw permit data), expiration, nonce, and the specific executor address. This binds the cosignature to a specific executor.

### Cosignature Nonces

- `cosignatureNonce = 0` (`REUSABLE_COSIGNATURE_NONCE`): Reusable until the order fills or expires. Fill-or-kill always uses 0 since the permit itself can only fill once.
- `cosignatureNonce > 0`: Consumed on-chain via bitmap. Once used, the same cosignature cannot be reused. Enables the cosigner to issue single-use authorizations for partial fills.

### Cosigner Destruction

If a cosigner key is compromised, it can be permanently destroyed:

```solidity
// Signed with COSIGNER_SELF_DESTRUCT_TYPEHASH using a universal EIP-712 domain
function destroyCosigner(address cosigner, bytes calldata signature) external;
```

Once `destroyedCosigners[cosigner] = true`, the cosigner can never be used again.

---

## Fill-or-Kill Validation

Before executing, the handler verifies exact fill:
- **Output swap** (`amountSpecified < 0`): `amountOut` must equal `|amountSpecified|`
- **Input swap** (`amountSpecified >= 0`): `amountIn` must equal `amountSpecified`

If the check fails, reverts with `PermitTransferHandler__FillOrKillPermitOrderNotFilled`.

After validation, calls `IPermitC.permitTransferFromWithAdditionalDataERC20()` on the PermitC contract.

---

## Partial Fill Validation

### Mode Matching

The sign of `permitAmountSpecified` must match `swapOrder.amountSpecified`:
- Both positive = input-based swap
- Both negative = output-based swap
- Mismatch = reverts with `PermitTransferHandler__PermitSwapInputOutputModeMismatch`

### Overpayment Prevention

For partial fills, the handler enforces proportional pricing:

**Output swap** (negative amountSpecified):
```
maxAmountIn = permitLimitAmount * amountOut / |permitAmountSpecified|
```
Reverts if `amountIn > maxAmountIn`.

**Input swap** (positive amountSpecified):
```
maxAmountIn = permitAmountSpecified * amountOut / permitLimitAmount
```
Reverts if `amountIn > maxAmountIn`.

After validation, calls `IPermitC.fillPermittedOrderERC20()` with the fill amounts.

---

## Hook Validation

If `permitData.hook != address(0)`, the handler calls:

```solidity
ITransferHandlerExecutorValidation(hook).validateExecutor(
    additionalDataHash,  // Used as the handlerId parameter
    executor,
    swapOrder,
    amountIn,
    amountOut,
    exchangeFee,
    feeOnTop,
    hookData
);
```

The hook can revert to block execution.

---

## End-to-End Foundry Example

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import "forge-std/Script.sol";
import {ILimitBreakAMM} from "@limitbreak/lbamm-core/src/interfaces/core/ILimitBreakAMM.sol";
import {SwapOrder} from "@limitbreak/lbamm-core/src/DataTypes.sol";
import {FillOrKillPermitTransfer} from
    "@limitbreak/lbamm-hooks-and-handlers/src/handlers/permit/DataTypes.sol";

contract PermitSwapScript is Script {
    function run() external {
        address amm = vm.envAddress("AMM_ADDRESS");
        address permitHandler = vm.envAddress("PERMIT_HANDLER_ADDRESS");
        address permitC = vm.envAddress("PERMIT_C_ADDRESS");
        uint256 makerKey = vm.envUint("MAKER_PRIVATE_KEY");
        address maker = vm.addr(makerKey);

        // 1. Build the permit data
        FillOrKillPermitTransfer memory permitData = FillOrKillPermitTransfer({
            permitProcessor: permitC,
            from: maker,
            nonce: 0,                    // Unique nonce
            permitAmount: 1 ether,       // Amount to swap
            expiration: block.timestamp + 1 hours,
            signature: "",               // Will be filled after signing
            cosigner: address(0),        // No cosigner
            cosignatureExpiration: 0,
            cosignature: "",
            hook: address(0),            // No hook
            hookData: ""
        });

        // 2. Build the swap order
        SwapOrder memory swapOrder = SwapOrder({
            deadline: block.timestamp + 1 hours,
            recipient: address(0xBEEF),  // Swap output recipient
            amountSpecified: int256(permitData.permitAmount),
            minAmountSpecified: 0,       // Accept any partial fill
            limitAmount: 0,              // Minimum output
            tokenIn: vm.envAddress("TOKEN_IN"),
            tokenOut: vm.envAddress("TOKEN_OUT")
        });

        // 3. Sign the permit (PermitC domain)
        // ... construct additionalDataHash from SWAP_TYPEHASH + swap params
        // ... sign with maker's private key against PermitC domain separator
        // ... set permitData.signature = abi.encodePacked(r, s, v)

        // 4. Encode transfer data
        bytes memory transferData = bytes.concat(
            bytes32(uint256(uint160(permitHandler))),
            bytes1(0x00),                // FILL_OR_KILL_PERMIT
            abi.encode(permitData)
        );

        // 5. Execute the swap
        vm.broadcast();
        ILimitBreakAMM(amm).singleSwap(
            swapOrder,
            vm.envBytes32("POOL_ID"),
            BPSFeeWithRecipient({recipient: address(0), BPS: 0}),
            FlatFeeWithRecipient({recipient: address(0), amount: 0}),
            SwapHooksExtraData({tokenInHook: "", tokenOutHook: "", poolHook: "", poolType: ""}),
            transferData
        );
    }
}
```

---

## Error Reference

| Error | Cause |
|---|---|
| `PermitTransferHandler__CallbackMustBeFromAMM` | `msg.sender` is not the AMM |
| `PermitTransferHandler__InvalidDataLength` | `transferExtraData` is empty |
| `PermitTransferHandler__InvalidPermitType` | First byte is not `0x00` or `0x01` |
| `PermitTransferHandler__FillOrKillPermitOrderNotFilled` | Fill-or-kill amount doesn't match exactly |
| `PermitTransferHandler__PermitSwapInputOutputModeMismatch` | `permitAmountSpecified` sign doesn't match swap mode |
| `PermitTransferHandler__PartialFillExceedsMaximumInputForOutput` | Proportional pricing violation |
| `PermitTransferHandler__PermitTransferFailed` | PermitC transfer returned error |
| `PermitTransferHandler__CosignatureExpired` | Cosignature past expiration |
| `PermitTransferHandler__CosignerDestroyed` | Cosigner was permanently destroyed |
| `PermitTransferHandler__CosignatureNonceAlreadyConsumed` | Single-use cosignature nonce reused |

---

## Security Considerations

1. **Two EIP-712 domains**: Maker's permit signature is domain-bound to **PermitC**. Cosigner's cosignature is domain-bound to **PermitTransferHandler**. Never mix these up.
2. **Cosigner key security**: Protect cosigner keys. Use `destroyCosigner()` immediately if compromised.
3. **Fill-or-kill is atomic**: `amountIn == amountSpecified` (input) or `amountOut == |amountSpecified|` (output) is strictly enforced.
4. **Partial fill proportionality**: Prevents overpayment via `FullMath.mulDiv` ratio checks.
5. **feeOnTop is unsigned**: Only `exchangeFee` (recipient + BPS) is signed. An executor can modify `feeOnTop` without invalidating the permit.
6. **Hook receives additionalDataHash as handlerId**: The first parameter to `validateExecutor` is the EIP-712 additional data hash, not a simple identifier.
7. **Cosignature binds to executor**: The cosignature includes the executor address, preventing front-running by different executors.
