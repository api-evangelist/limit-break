# Wrapped Native API Reference

Complete function signatures and postconditions for all Wrapped Native operations.

## Table of Contents

1. [Depositing / Wrapping](#depositing--wrapping)
2. [Withdrawing / Unwrapping](#withdrawing--unwrapping)
3. [ERC-20 Approvals and Transfers](#erc-20-approvals-and-transfers)
4. [Permits](#permits)
5. [Permit Cancellation](#permit-cancellation)
6. [View Functions](#view-functions)

---

## Depositing / Wrapping

### receive() / fallback()

Native value sent to the contract without calldata triggers `receive()`. Native value sent with calldata triggers `fallback()`. Both credit `msg.sender` with WNATIVE.

### deposit()

```solidity
function deposit() public payable;
```

Credits `msg.sender` with WNATIVE equal to `msg.value`.

**Postconditions:**
1. Contract's native balance increased by `msg.value`
2. `msg.sender`'s WNATIVE balance increased by `msg.value`
3. `Deposit` event emitted with `msg.sender`

### depositTo(address to)

```solidity
function depositTo(address to) public payable;
```

Credits the specified `to` account with WNATIVE while taking `msg.sender`'s native funds. Ideal for protocol refunds -- more gas efficient than `deposit` + `transfer` and limits re-entrancy risk.

**Postconditions:**
1. Contract's native balance increased by `msg.value`
2. `to`'s WNATIVE balance increased by `msg.value`
3. `Deposit` event emitted -- **the `to` address is logged, not `msg.sender`**

---

## Withdrawing / Unwrapping

### withdraw(uint256 amount)

```solidity
function withdraw(uint256 amount) public;
```

Debits `msg.sender`'s WNATIVE and sends native tokens back.

**Throws:** Insufficient balance, native transfer failure.

**Postconditions:**
1. `msg.sender`'s WNATIVE balance decreased by `amount`
2. `msg.sender`'s native balance increased by `amount`
3. `Withdrawal` event emitted with `msg.sender`

### withdrawToAccount(address to, uint256 amount)

```solidity
function withdrawToAccount(address to, uint256 amount) public;
```

Debits `msg.sender`'s WNATIVE but sends native tokens to `to`.

**Throws:** Insufficient balance, native transfer failure to `to`.

**Postconditions:**
1. `msg.sender`'s WNATIVE balance decreased by `amount`
2. `to`'s native balance increased by `amount`
3. `Withdrawal` event emitted -- **`msg.sender` is logged, not `to`**

### withdrawSplit(address[] calldata toAddresses, uint256[] calldata amounts)

```solidity
function withdrawSplit(address[] calldata toAddresses, uint256[] calldata amounts) external;
```

Batch withdrawal splitting `msg.sender`'s WNATIVE across multiple recipients.

**Throws:** Insufficient balance, array length mismatch, native transfer failure.

**Postconditions:**
1. `msg.sender`'s WNATIVE balance decreased by sum of `amounts`
2. Each receiver's native balance increased by corresponding amount
3. `Withdrawal` event emitted per receiver -- **`msg.sender` is logged, not receiver**

---

## ERC-20 Approvals and Transfers

All three functions are **payable**. When `msg.value > 0`, a deposit occurs before the ERC-20 action.

### approve(address spender, uint256 amount)

```solidity
function approve(address spender, uint256 amount) public payable returns (bool);
```

Standard ERC-20 approval with optional deposit. Setting `amount` to `type(uint256).max` grants unlimited approval. Powerful UX improvement -- protocols like Seaport and Payment Processor no longer need separate deposit and approval calls.

**Postconditions:**
1. `spender` approved for `amount` of `msg.sender`'s WNATIVE
2. `Approval` event emitted
3. If `msg.value > 0`: `msg.sender`'s WNATIVE balance increased, `Deposit` event emitted

### transfer(address to, uint256 amount)

```solidity
function transfer(address to, uint256 amount) public payable returns (bool);
```

Standard ERC-20 transfer with optional deposit.

**Throws:** Insufficient balance.

**Postconditions:**
1. `amount` transferred from `msg.sender` to `to`
2. `Transfer` event emitted
3. If `msg.value > 0`: `msg.sender`'s WNATIVE balance increased first, `Deposit` event emitted

### transferFrom(address from, address to, uint256 amount)

```solidity
function transferFrom(address from, address to, uint256 amount) public payable returns (bool);
```

Standard ERC-20 transferFrom with optional deposit.

**IMPORTANT: When `msg.value > 0`, the deposit credits the `from` address, NOT `msg.sender`.** Integrating spender/operator protocols must be aware that deposits during transfers will not credit their own account.

**Throws:** Insufficient balance, insufficient allowance.

**Postconditions:**
1. `amount` transferred from `from` to `to`
2. `Transfer` event emitted
3. If `msg.value > 0`: `from`'s WNATIVE balance increased, `Deposit` event emitted with `from`

---

## Permits

### permitTransfer

```solidity
function permitTransfer(
    address from,
    address to,
    uint256 transferAmount,
    uint256 permitAmount,
    uint256 nonce,
    uint256 expiration,
    bytes calldata signedPermit
) external payable;
```

Gasless ERC-20 transfer using EIP-712 signature. Permits are single-use. The `from` account signs a permit authorizing `msg.sender` (operator) to transfer up to `permitAmount`. The actual `transferAmount` can be less than or equal to `permitAmount`.

**IMPORTANT: Same deposit behavior as `transferFrom` -- deposits credit `from`, not `msg.sender`.**

**Throws:** `from` is zero address, `msg.sender` != signed operator, parameter mismatch with signature, expired permit, `transferAmount` > `permitAmount`, nonce already used/revoked, master nonce revoked, invalid signature, insufficient balance.

**Postconditions:**
1. `nonce` invalidated for `from`
2. `PermitNonceInvalidated` event emitted
3. `transferAmount` transferred from `from` to `to`
4. `Transfer` event emitted with `from`
5. If `msg.value > 0`: `from`'s WNATIVE balance increased, `Deposit` event emitted

### doPermittedWithdraw

```solidity
function doPermittedWithdraw(
    address from,
    address to,
    uint256 amount,
    uint256 nonce,
    uint256 expiration,
    address convenienceFeeReceiver,
    uint256 convenienceFeeBps,
    bytes calldata signedPermit
) external;
```

Gasless withdrawal on behalf of a user. Designed for gas sponsorship services where a user has WNATIVE but insufficient native funds. The operator pays gas and optionally charges a convenience fee acknowledged in the user's signed permit.

See `permit-system.md` for fee calculation details.

**Throws:** `from` is zero address, `msg.sender` != signed operator, parameter mismatch with signature, expired permit, nonce already used/revoked, master nonce revoked, invalid signature, insufficient balance.

**Postconditions:**
1. `nonce` invalidated for `from`
2. `PermitNonceInvalidated` event emitted
3. `from`'s WNATIVE balance decreased by `amount`
4. `to`'s native balance increased by `amount` minus fees
5. `convenienceFeeReceiver`'s WNATIVE balance increased by convenience fee
6. Infrastructure tax account's WNATIVE balance increased by infrastructure fee
7. `Withdrawal` event emitted -- **`from` is logged, not `to` or `msg.sender`**

---

## Permit Cancellation

### revokeMyOutstandingPermits()

```solidity
function revokeMyOutstandingPermits() external;
```

Increments `msg.sender`'s master nonce by 1, invalidating ALL outstanding transfer and withdrawal permits.

**Postconditions:**
1. `msg.sender`'s master nonce incremented
2. `MasterNonceInvalidated` event emitted

### revokeMyNonce(uint256 nonce)

```solidity
function revokeMyNonce(uint256 nonce) external;
```

Permanently invalidates a single permit nonce.

**Throws:** Nonce already used or revoked.

**Postconditions:**
1. `nonce` invalidated for `msg.sender`
2. `PermitNonceInvalidated` event emitted

---

## View Functions

```solidity
function totalSupply() external view returns (uint256);
function balanceOf(address account) external view returns (uint256);
function allowance(address owner, address spender) external view returns (uint256);
function name() external view returns (string memory);     // "Wrapped Native"
function symbol() external view returns (string memory);   // "WNATIVE"
function decimals() external view returns (uint8);         // 18
function domainSeparatorV4() external view returns (bytes32);
function isNonceUsed(address account, uint256 nonce) external view returns (bool);
function masterNonces(address account) external view returns (uint256);
```

Note: `totalSupply`, `isNonceUsed`, `masterNonces`, `domainSeparatorV4`, `name`, `symbol`, and `decimals` are routed through the `fallback()` function via assembly for gas efficiency.
