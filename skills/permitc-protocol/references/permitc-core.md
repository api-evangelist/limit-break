# PermitC Core Reference

## Table of Contents

- [Interface (IPermitC)](#interface-ipermitc)
- [Data Types](#data-types)
- [EIP-712 Typehashes](#eip-712-typehashes)
- [Stored Approvals](#stored-approvals)
- [Permit Transfers (Single-Use)](#permit-transfers-single-use)
- [Additional Data Permits](#additional-data-permits)
- [Order Fills (Partial)](#order-fills-partial)
- [Nonce Management](#nonce-management)
- [Lockdown](#lockdown)
- [Signing Patterns](#signing-patterns)
- [Error Reference](#error-reference)
- [Transfer Validator Inheritance and Auto-Approval](#transfer-validator-inheritance-and-auto-approval)
- [Integration Checklist](#integration-checklist)

---

## Interface (IPermitC)

```solidity
interface IPermitC {
    // ===== Stored Approvals =====
    function approve(uint256 tokenType, address token, uint256 id, address operator, uint200 amount, uint48 expiration) external;

    function updateApprovalBySignature(
        uint256 tokenType, address token, uint256 id, uint256 nonce, uint200 amount,
        address operator, uint48 approvalExpiration, uint48 sigDeadline,
        address owner, bytes calldata signedPermit
    ) external;

    function allowance(address owner, address operator, uint256 tokenType, address token, uint256 id)
        external view returns (uint256 amount, uint256 expiration);

    // ===== Signed Single-Use Transfers =====
    function registerAdditionalDataHash(string memory additionalDataTypeString) external;

    function permitTransferFromERC721(
        address token, uint256 id, uint256 nonce, uint256 expiration,
        address owner, address to, bytes calldata signedPermit
    ) external returns (bool isError);

    function permitTransferFromERC1155(
        address token, uint256 id, uint256 nonce, uint256 permitAmount, uint256 expiration,
        address owner, address to, uint256 transferAmount, bytes calldata signedPermit
    ) external returns (bool isError);

    function permitTransferFromERC20(
        address token, uint256 nonce, uint256 permitAmount, uint256 expiration,
        address owner, address to, uint256 transferAmount, bytes calldata signedPermit
    ) external returns (bool isError);

    // ===== With Additional Data =====
    function permitTransferFromWithAdditionalDataERC721(
        address token, uint256 id, uint256 nonce, uint256 expiration,
        address owner, address to, bytes32 additionalData, bytes32 advancedPermitHash,
        bytes calldata signedPermit
    ) external returns (bool isError);

    function permitTransferFromWithAdditionalDataERC1155(
        address token, uint256 id, uint256 nonce, uint256 permitAmount, uint256 expiration,
        address owner, address to, uint256 transferAmount, bytes32 additionalData,
        bytes32 advancedPermitHash, bytes calldata signedPermit
    ) external returns (bool isError);

    function permitTransferFromWithAdditionalDataERC20(
        address token, uint256 nonce, uint256 permitAmount, uint256 expiration,
        address owner, address to, uint256 transferAmount, bytes32 additionalData,
        bytes32 advancedPermitHash, bytes calldata signedPermit
    ) external returns (bool isError);

    // ===== Order Fills (Partial) =====
    function fillPermittedOrderERC1155(
        bytes calldata signedPermit, OrderFillAmounts calldata orderFillAmounts,
        address token, uint256 id, address owner, address to,
        uint256 nonce, uint48 expiration, bytes32 orderId, bytes32 advancedPermitHash
    ) external returns (uint256 quantityFilled, bool isError);

    function fillPermittedOrderERC20(
        bytes calldata signedPermit, OrderFillAmounts calldata orderFillAmounts,
        address token, address owner, address to,
        uint256 nonce, uint48 expiration, bytes32 orderId, bytes32 advancedPermitHash
    ) external returns (uint256 quantityFilled, bool isError);

    function closePermittedOrder(
        address owner, address operator, uint256 tokenType,
        address token, uint256 id, bytes32 orderId
    ) external;

    function allowance(
        address owner, address operator, uint256 tokenType,
        address token, uint256 id, bytes32 orderId
    ) external view returns (uint256 amount, uint256 expiration);

    // ===== Nonce Management =====
    function invalidateUnorderedNonce(uint256 nonce) external;
    function isValidUnorderedNonce(address owner, uint256 nonce) external view returns (bool isValid);
    function lockdown() external;
    function masterNonce(address owner) external view returns (uint256);

    // ===== Direct Transfers (from stored approvals) =====
    function transferFromERC721(address from, address to, address token, uint256 id) external returns (bool isError);
    function transferFromERC1155(address from, address to, address token, uint256 id, uint256 amount) external returns (bool isError);
    function transferFromERC20(address from, address to, address token, uint256 amount) external returns (bool isError);

    // ===== Domain =====
    function domainSeparatorV4() external view returns (bytes32);
}
```

---

## Data Types

```solidity
/// @dev Storage struct for approvals and order state
struct PackedApproval {
    uint8 state;        // Order state: 0 = open, 1 = filled, 2 = cancelled
    uint200 amount;     // Approved amount
    uint48 expiration;  // Expiration timestamp
}

/// @dev Calldata struct for partial fill amounts
struct OrderFillAmounts {
    uint256 orderStartAmount;     // Total order amount
    uint256 requestedFillAmount;  // Amount to fill in this transaction
    uint256 minimumFillAmount;    // Minimum acceptable fill (reverts if remaining < this)
}
```

### Token Type Constants

```solidity
uint256 constant TOKEN_TYPE_ERC721  = 721;
uint256 constant TOKEN_TYPE_ERC1155 = 1155;
uint256 constant TOKEN_TYPE_ERC20   = 20;
```

### Order State Constants

```solidity
uint8 constant ORDER_STATE_OPEN      = 0;
uint8 constant ORDER_STATE_FILLED    = 1;
uint8 constant ORDER_STATE_CANCELLED = 2;
```

---

## EIP-712 Typehashes

### Stored Approval (UpdateApprovalBySignature)

```solidity
bytes32 constant UPDATE_APPROVAL_TYPEHASH = keccak256(
    "UpdateApprovalBySignature(uint256 tokenType,address token,uint256 id,uint256 amount,uint256 nonce,address operator,uint256 approvalExpiration,uint256 sigDeadline,uint256 masterNonce)"
);
```

### Single-Use Permit (PermitTransferFrom)

```solidity
bytes32 constant SINGLE_USE_PERMIT_TYPEHASH = keccak256(
    "PermitTransferFrom(uint256 tokenType,address token,uint256 id,uint256 amount,uint256 nonce,address operator,uint256 expiration,uint256 masterNonce)"
);
```

### Permit with Additional Data (stub -- appended with protocol-specific type string)

```solidity
string constant SINGLE_USE_PERMIT_TRANSFER_ADVANCED_TYPEHASH_STUB =
    "PermitTransferFromWithAdditionalData(uint256 tokenType,address token,uint256 id,uint256 amount,uint256 nonce,address operator,uint256 expiration,uint256 masterNonce,";

// Example: for Payment Processor, the full type string becomes:
// "PermitTransferFromWithAdditionalData(...,uint256 masterNonce,SaleApproval approval)SaleApproval(...)"
```

### Order Permit with Additional Data (stub)

```solidity
string constant PERMIT_ORDER_ADVANCED_TYPEHASH_STUB =
    "PermitOrderWithAdditionalData(uint256 tokenType,address token,uint256 id,uint256 amount,uint256 salt,address operator,uint256 expiration,uint256 masterNonce,";
```

**Registering additional data type strings:** Protocols that use `permitTransferFromWithAdditionalData` or `fillPermittedOrder` must register their type string by calling `registerAdditionalDataHash(additionalDataTypeString)`. This prevents malicious EIP-712 envelope label substitution. The type string is the portion after the stub, e.g., `"SaleApproval approval)SaleApproval(uint8 protocol,...)"`.

---

## Stored Approvals

### On-Chain Approve

```solidity
// Approve an operator to transfer specific tokens with expiration
permitC.approve(
    TOKEN_TYPE_ERC721,    // tokenType
    nftContract,          // token
    tokenId,              // id (0 for ERC20)
    operatorAddress,      // operator
    1,                    // amount (always 1 for ERC721, up to type(uint200).max for fungibles)
    uint48(block.timestamp + 1 hours)  // expiration
);
```

An `expiration` of `0` means the approval is valid only in the current block (atomic).

### Approval by Signature

Operators can submit a signed message from the owner to create a stored approval on-chain. This avoids requiring the owner to send a transaction.

### Transfer from Stored Approval

```solidity
// Operator calls transferFrom after approval is stored
permitC.transferFromERC721(owner, recipient, nftContract, tokenId);
permitC.transferFromERC20(owner, recipient, tokenContract, amount);
permitC.transferFromERC1155(owner, recipient, tokenContract, tokenId, amount);
```

Returns `true` if the transfer failed (error flag pattern, not revert).

---

## Permit Transfers (Single-Use)

Single-use permits are signed off-chain and consumed in one transaction. The nonce is invalidated on use.

### Basic Permit Transfer

```solidity
// Operator submits the owner's signed permit to transfer tokens
bool isError = permitC.permitTransferFromERC20(
    tokenAddress,        // token
    nonce,               // unordered nonce
    permitAmount,        // amount the owner signed for
    expiration,          // when the permit expires
    owner,               // token owner who signed
    recipient,           // where tokens go
    transferAmount,      // actual transfer amount (<= permitAmount)
    signedPermit         // owner's EIP-712 signature (65 bytes or 64 bytes packed)
);
```

The `operator` in the signed message is `msg.sender` -- PermitC derives it from the caller. This means only the intended operator can use the permit.

---

## Additional Data Permits

The `permitTransferFromWithAdditionalData` variants allow protocols to embed protocol-specific data in the signed message. This is the primary mechanism used by Payment Processor, LBAMM, and TokenMaster.

### How It Works

1. The protocol defines a custom struct (e.g., `SaleApproval`, `AdvancedBuyOrder`)
2. The protocol registers the type string with PermitC via `registerAdditionalDataHash()`
3. The user signs a `PermitTransferFromWithAdditionalData` message that includes both the PermitC fields and the protocol-specific data as `additionalData`
4. The protocol calls `permitTransferFromWithAdditionalData*()` passing `additionalData` (the keccak256 hash of the encoded struct) and `advancedPermitHash` (the keccak256 of the full type string)

### Additional Data Parameters

- `additionalData` (bytes32): The keccak256 hash of the ABI-encoded protocol-specific struct
- `advancedPermitHash` (bytes32): The keccak256 hash of the complete EIP-712 type string (stub + protocol portion)

### Signing Domain

**The signature always uses the PermitC contract's EIP-712 domain**, not the consuming protocol's domain. This is a common source of bugs -- the domain separator must come from `permitC.domainSeparatorV4()`.

---

## Order Fills (Partial)

Order-based permits support multiple partial fills against a single signed permit. PermitC tracks cumulative fill state per order.

### Opening and Filling

```solidity
OrderFillAmounts memory fillAmounts = OrderFillAmounts({
    orderStartAmount: 100,       // Total order size
    requestedFillAmount: 25,     // Fill 25 in this tx
    minimumFillAmount: 10        // Revert if < 10 remaining
});

(uint256 filled, bool isError) = permitC.fillPermittedOrderERC1155(
    signedPermit,
    fillAmounts,
    token,
    tokenId,
    owner,
    recipient,
    nonce,
    expiration,
    orderId,          // Unique order identifier
    advancedPermitHash
);
```

### Closing an Order

```solidity
// Owner or operator can close an open order
permitC.closePermittedOrder(owner, operator, tokenType, token, id, orderId);
```

### Checking Order State

```solidity
(uint256 remainingAmount, uint256 expiration) = permitC.allowance(
    owner, operator, tokenType, token, id, orderId
);
```

---

## Nonce Management

PermitC uses **unordered nonces** (bitmap-based, same pattern as Permit2). Nonces can be used in any order and are tracked in a bitmap for gas efficiency.

### Invalidating a Nonce

```solidity
// Owner invalidates a specific nonce to cancel a pending permit
permitC.invalidateUnorderedNonce(nonce);
```

### Checking Nonce Validity

```solidity
bool valid = permitC.isValidUnorderedNonce(owner, nonce);
```

### Nonce Selection

Nonces are arbitrary `uint256` values. Common patterns:
- Sequential: `0, 1, 2, ...` (simple but must track last used)
- Random: `uint256(keccak256(...))` (no coordination needed, negligible collision risk)
- Timestamp-based: `block.timestamp << 128 | counter` (sortable + unique)

---

## Lockdown

```solidity
// Instantly invalidate ALL outstanding approvals and permits
permitC.lockdown();
```

Lockdown increments the owner's `masterNonce`. Since the `masterNonce` is included in every signed message, all existing signatures become invalid. This is the emergency "revoke everything" mechanism.

```solidity
uint256 currentMasterNonce = permitC.masterNonce(owner);
```

---

## Signing Patterns

### Solidity (Foundry test/script)

```solidity
// Construct the permit hash
bytes32 permitHash = keccak256(
    abi.encode(
        SINGLE_USE_PERMIT_TYPEHASH,
        tokenType,
        token,
        id,
        amount,
        nonce,
        operator,        // who will call permitTransferFrom
        expiration,
        masterNonce
    )
);

// Sign with EIP-712 domain
bytes32 digest = keccak256(
    abi.encodePacked("\x19\x01", permitC.domainSeparatorV4(), permitHash)
);

(uint8 v, bytes32 r, bytes32 s) = vm.sign(ownerPrivateKey, digest);
bytes memory signature = abi.encodePacked(r, s, v);
```

### JavaScript / TypeScript (ethers.js v6)

**Important:** The EIP-712 domain `name` and `version` are set at deployment and may vary between the standalone PermitC and the Transfer Validator (which inherits PermitC). Always verify the domain by querying the contract or checking deployment parameters. When in doubt, reconstruct the domain separator from `domainSeparatorV4()` returned by the contract.

```javascript
// Example domain -- verify name/version against the actual deployment
const domain = {
  name: "PermitC",       // May differ for Transfer Validator
  version: "1",          // May differ for Transfer Validator
  chainId: chainId,
  verifyingContract: permitCAddress  // Use Transfer Validator address for Creator tokens
};

const types = {
  PermitTransferFrom: [
    { name: "tokenType", type: "uint256" },
    { name: "token", type: "address" },
    { name: "id", type: "uint256" },
    { name: "amount", type: "uint256" },
    { name: "nonce", type: "uint256" },
    { name: "operator", type: "address" },
    { name: "expiration", type: "uint256" },
    { name: "masterNonce", type: "uint256" }
  ]
};

const value = {
  tokenType: 20,
  token: tokenAddress,
  id: 0,
  amount: ethers.parseEther("100"),
  nonce: nonce,
  operator: operatorAddress,
  expiration: Math.floor(Date.now() / 1000) + 3600,
  masterNonce: await permitC.masterNonce(signer.address)
};

const signature = await signer.signTypedData(domain, types, value);
```

### Signature Format

PermitC accepts signatures in two formats:
- **65 bytes**: `abi.encodePacked(r, s, v)` -- standard ECDSA
- **64 bytes**: `abi.encodePacked(r, vs)` -- compact EIP-2098 format

PermitC also supports ERC-1271 smart contract signatures for contract wallet owners.

---

## Error Reference

| Error | Cause |
|---|---|
| `PermitC__ApprovalTransferPermitExpiredOrUnset` | Approval has expired or was never set |
| `PermitC__ApprovalTransferExceededPermittedAmount` | Transfer amount exceeds approved amount |
| `PermitC__SignatureTransferExceededPermitExpired` | Permit signature has expired |
| `PermitC__SignatureTransferExceededPermittedAmount` | Transfer amount exceeds signed permit amount |
| `PermitC__SignatureTransferInvalidSignature` | Signature doesn't recover to expected owner |
| `PermitC__SignatureTransferPermitHashNotRegistered` | Additional data type hash not registered |
| `PermitC__NonceAlreadyUsedOrRevoked` | Nonce was already consumed or invalidated |
| `PermitC__OrderIsEitherCancelledOrFilled` | Order is closed (filled or cancelled) |
| `PermitC__UnableToFillMinimumRequestedQuantity` | Remaining order amount < minimum fill |
| `PermitC__CallerMustBeOwnerOrOperator` | Only owner or operator can close an order |
| `PermitC__InvalidTokenType` | Token type must be 20, 721, or 1155 |
| `PermitC__AmountExceedsStorageMaximum` | Amount exceeds `type(uint200).max` |

---

## Transfer Validator Inheritance and Auto-Approval

### The Transfer Validator IS a PermitC Instance

The `CreatorTokenTransferValidator` contract inherits directly from `PermitC`:

```solidity
contract CreatorTokenTransferValidator is
    IEOARegistry,
    ITransferValidator,
    ValidatorBase,
    ERC165,
    Tstorish,
    PermitC {  // <-- PermitC is inherited

    constructor(/* ... */)
        PermitC(name, version, defaultOwner, nativeValueToCheckPauseState)
    { /* ... */ }
}
```

This means the Transfer Validator V5 at `0x721C008fdff27BF06E7E123956E2Fe03B63342e3` exposes the entire `IPermitC` interface. Any PermitC function (`approve`, `permitTransferFrom*`, `fillPermittedOrder*`, `lockdown`, etc.) can be called directly on the Transfer Validator address.

### Auto-Approval for Zero-Friction Launches

Creator Token Standard contracts (ERC-20C, ERC-721C, ERC-1155C) inherit `AutomaticValidatorTransferApproval`, which gives the contract owner a one-call opt-in:

```solidity
// Contract owner enables auto-approval (one-time call)
myToken.setAutomaticApprovalOfTransfersFromValidator(true);
```

When enabled:
- **ERC-721C / ERC-1155C**: `isApprovedForAll(owner, transferValidator)` returns `true` for all token holders automatically
- **ERC-20C**: `allowance(owner, transferValidator)` returns `type(uint256).max` for all token holders automatically

### The Zero-Approval Flow

With auto-approval enabled, users can interact with protocols using only signed PermitC orders -- no approval transactions needed:

1. **Token creator** calls `setAutomaticApprovalOfTransfersFromValidator(true)` (once, at setup)
2. **User** signs a PermitC permit off-chain (e.g., a `PermitTransferFromWithAdditionalData` wrapping a `SaleApproval`)
3. **Protocol** calls `permitTransferFrom*()` on the **Transfer Validator address** (which is the PermitC instance)
4. The Transfer Validator (as PermitC) verifies the signature and calls `transferFrom` on the token
5. The token checks `isApprovedForAll` / `allowance` for the Transfer Validator -- auto-approval returns true
6. Transfer succeeds with zero prior approval transactions from the user

### Important: Which PermitC Address to Use

For Creator Token Standard tokens, the PermitC signing domain must use the **Transfer Validator address** (`0x721C008fdff27BF06E7E123956E2Fe03B63342e3`), not the standalone PermitC deployment. The EIP-712 domain separator is bound to the contract address, so signatures created against the wrong address will be invalid.

```javascript
// Correct: use Transfer Validator as the PermitC address for Creator tokens
const domain = {
  name: "PermitC",
  version: "1",
  chainId: chainId,
  verifyingContract: "0x721C008fdff27BF06E7E123956E2Fe03B63342e3"  // Transfer Validator
};
```

---

## Integration Checklist

When integrating PermitC into a protocol:

1. **Choose the right PermitC address** -- for Creator Token Standard tokens, use the Transfer Validator (`0x721C008fdff27BF06E7E123956E2Fe03B63342e3`) as the PermitC instance; for non-Creator tokens, use the standalone PermitC (`0x000000fCE53B6fC312838A002362a9336F7ce39B`)
2. **Register your type hash** -- call `registerAdditionalDataHash()` with your protocol's additional data type string if using `WithAdditionalData` variants
3. **Use the PermitC domain** -- signatures must use `permitC.domainSeparatorV4()`, not your protocol's domain
4. **Set `operator` to your contract** -- the operator in the signed message must match `msg.sender` when calling PermitC
5. **Handle the `isError` return** -- permit transfer functions return a boolean error flag instead of reverting on transfer failure
6. **Set reasonable expirations** -- don't use `type(uint48).max`; use hours or days appropriate for your use case
7. **Query `masterNonce`** -- always read the current `masterNonce` when constructing signatures; it changes on `lockdown()`
8. **Token type rules** -- ERC-20: `id = 0`; ERC-721: `amount = 1`; ERC-1155: both `id` and `amount` are meaningful
9. **Consider auto-approval** -- if the token is a Creator Standard token, the creator can enable `setAutomaticApprovalOfTransfersFromValidator(true)` to eliminate user approval transactions entirely
