# Wrapped Native Permit System

EIP-712 typed data signatures enable gasless transfers and withdrawals. Permits can be signed by EOAs (ECDSA) or smart contracts supporting EIP-1271.

## Table of Contents

1. [Domain Separator](#domain-separator)
2. [Transfer Permits](#transfer-permits)
3. [Withdrawal Permits](#withdrawal-permits)
4. [Nonce Management](#nonce-management)
5. [Convenience Fee Model](#convenience-fee-model)
6. [Signature Format](#signature-format)

---

## Domain Separator

All permits use EIP-712 structured data with the following domain:

```solidity
bytes32 private constant DOMAIN_SEPARATOR =
    keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)");
```

| Field | Value |
|---|---|
| name | `"Wrapped Native"` |
| version | `"1"` |
| chainId | Chain-specific (e.g., 1 for Ethereum mainnet) |
| verifyingContract | `0x6000030000842044000077551D00cfc6b4005900` |

---

## Transfer Permits

### Type Hash

```solidity
bytes32 constant PERMIT_TRANSFER_TYPEHASH =
    keccak256("PermitTransfer(address operator,uint256 amount,uint256 nonce,uint256 expiration,uint256 masterNonce)");
```

### Signed Fields

| Field | Type | Description |
|---|---|---|
| operator | address | The address authorized to execute the transfer (`msg.sender` when calling `permitTransfer`) |
| amount | uint256 | Maximum amount that can be transferred (the `permitAmount` parameter) |
| nonce | uint256 | Single-use nonce for this permit |
| expiration | uint256 | Unix timestamp after which the permit is no longer valid |
| masterNonce | uint256 | The signer's current master nonce at time of signing |

### Usage

The signer (`from`) creates and signs this typed data. The operator calls `permitTransfer` with the signed permit bytes. The actual `transferAmount` can be less than or equal to the signed `amount`.

---

## Withdrawal Permits

### Type Hash

```solidity
bytes32 constant PERMIT_WITHDRAWAL_TYPEHASH =
    keccak256("PermitWithdrawal(address operator,uint256 amount,uint256 nonce,uint256 expiration,uint256 masterNonce,address to,address convenienceFeeReceiver,uint256 convenienceFeeBps)");
```

### Signed Fields

| Field | Type | Description |
|---|---|---|
| operator | address | The address authorized to execute the withdrawal (`msg.sender`) |
| amount | uint256 | Amount of WNATIVE to withdraw |
| nonce | uint256 | Single-use nonce for this permit |
| expiration | uint256 | Unix timestamp after which the permit is no longer valid |
| masterNonce | uint256 | The signer's current master nonce at time of signing |
| to | address | Recipient of the unwrapped native tokens |
| convenienceFeeReceiver | address | Address that receives the convenience fee (can be zero for no fee) |
| convenienceFeeBps | uint256 | Convenience fee in basis points |

### Usage

The signer (`from`) creates and signs this typed data, acknowledging the convenience fee. The operator calls `doPermittedWithdraw` with matching parameters and the signed permit bytes.

---

## Nonce Management

Wrapped Native uses a **bitmap-based nonce system** for gas-efficient nonce tracking:

- Each `uint256` storage slot tracks 256 individual nonces (one bit each)
- Nonce `N` is stored at slot `N / 256`, bit position `N % 256`
- This means checking or invalidating a nonce requires only a single storage read + write

### Master Nonce

Each account has a master nonce that acts as a global invalidation counter:

- The master nonce is included in every permit signature
- Calling `revokeMyOutstandingPermits()` increments the master nonce by 1
- Any permit signed with the old master nonce becomes invalid
- This is the "nuclear option" for cancelling all outstanding permits at once

### Individual Nonce Revocation

- `revokeMyNonce(uint256 nonce)` permanently marks a single nonce as used
- This allows cancelling a specific permit without affecting others
- Reverts if the nonce was already used or revoked

### Checking Nonce Status

```solidity
function isNonceUsed(address account, uint256 nonce) external view returns (bool);
function masterNonces(address account) external view returns (uint256);
```

---

## Convenience Fee Model

Permitted withdrawals (`doPermittedWithdraw`) support a convenience fee for gas sponsorship services. The fee structure includes an infrastructure tax:

### Fee Calculation

When `convenienceFeeBps > 9` (above the infrastructure tax threshold):
- **Infrastructure fee** = 10% of the convenience fee (i.e., `convenienceFeeBps / 10` of the amount)
- **Convenience fee receiver** gets the remaining 90%
- **User receives** = `amount` - total fees

When `0 < convenienceFeeBps <= 9`:
- **Infrastructure fee** = `amount / 10000` (fixed minimum)
- **Convenience fee receiver** gets the specified BPS minus infrastructure share
- **User receives** = `amount` - total fees

When `convenienceFeeReceiver` is `address(0)`:
- Convenience fee defaults to zero regardless of `convenienceFeeBps`
- Infrastructure tax of 1 BPS (`amount / 10000`) is **still charged** -- it always applies to permitted withdrawals
- User receives `amount - (amount / 10000)`

### Fee Destination

- Convenience fees and infrastructure taxes remain **wrapped** (WNATIVE balance credits)
- Only the user's withdrawal amount is unwrapped to native tokens

### Example

A gas sponsorship app charges 0.25% (25 BPS) convenience fee on a 1 ETH withdrawal:
- Total fee: 0.0025 ETH
- Infrastructure tax (10% of fee): 0.00025 ETH (stays as WNATIVE)
- App receives: 0.00225 ETH (stays as WNATIVE)
- User receives: 0.9975 ETH (native)

---

## Signature Format

Wrapped Native accepts three signature formats:

1. **Standard ECDSA (65 bytes)**: `r` (32 bytes) + `s` (32 bytes) + `v` (1 byte)
2. **Compact ECDSA (64 bytes)**: EIP-2098 compact format with `v` packed into `s`
3. **EIP-1271 contract signatures**: For smart contract wallets -- the contract's `isValidSignature(bytes32, bytes)` is called to verify

The signature verification first attempts ECDSA recovery. If the recovered address doesn't match the signer, it falls back to EIP-1271 verification on the signer address.
