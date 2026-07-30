# EIP-712 Order Signing Reference

Complete reference for signing Payment Processor V3 orders, including domain separator, typehashes, cosignatures, bulk signing, and nonce strategy.

---

## Domain Separator

All Payment Processor orders use the following EIP-712 domain:

| Field | Value |
|---|---|
| name | `"PaymentProcessor"` |
| version | `"3.0.0"` |
| chainId | Current chain ID |
| verifyingContract | PaymentProcessor address (`0x9a1D00000000fC540e2000560054812452eB5366`) |

---

## Order Type Hashes

Each order type has a unique EIP-712 typehash. The typehash is the `keccak256` of the EIP-712 type string.

### SaleApproval (Listings)

```
Typehash: keccak256 of the type string below
```

```
SaleApproval(uint8 protocol,address cosigner,address seller,address marketplace,address fallbackRoyaltyRecipient,address paymentMethod,address tokenAddress,uint256 tokenId,uint256 amount,uint256 itemPrice,uint256 expiration,uint256 marketplaceFeeNumerator,uint256 maxRoyaltyFeeNumerator,uint256 nonce,uint256 masterNonce,uint256 protocolFeeVersion)
```

### ItemOfferApproval

```
Typehash: keccak256 of the type string below
```

```
ItemOfferApproval(uint8 protocol,address cosigner,address buyer,address beneficiary,address marketplace,address fallbackRoyaltyRecipient,address paymentMethod,address tokenAddress,uint256 tokenId,uint256 amount,uint256 itemPrice,uint256 expiration,uint256 marketplaceFeeNumerator,uint256 nonce,uint256 masterNonce)
```

### CollectionOfferApproval

```
Typehash: keccak256 of the type string below
```

```
CollectionOfferApproval(uint8 protocol,address cosigner,address buyer,address beneficiary,address marketplace,address fallbackRoyaltyRecipient,address paymentMethod,address tokenAddress,uint256 amount,uint256 itemPrice,uint256 expiration,uint256 marketplaceFeeNumerator,uint256 nonce,uint256 masterNonce)
```

Note: CollectionOfferApproval does **not** include `tokenId` -- the offer applies to any token in the collection.

### TokenSetOfferApproval

```
Typehash: keccak256 of the type string below
```

```
TokenSetOfferApproval(uint8 protocol,address cosigner,address buyer,address beneficiary,address marketplace,address fallbackRoyaltyRecipient,address paymentMethod,address tokenAddress,uint256 amount,uint256 itemPrice,uint256 expiration,uint256 marketplaceFeeNumerator,uint256 nonce,uint256 masterNonce,bytes32 tokenSetMerkleRoot)
```

Note: Includes `tokenSetMerkleRoot` -- the offer applies only to token IDs whose merkle proof validates against this root.

### PermittedTransferSaleApproval (PermitC Wrapper)

```
Typehash: keccak256 of the type string below
```

```
PermitTransferFromWithAdditionalData(uint256 tokenType,address token,uint256 id,uint256 amount,uint256 nonce,address operator,uint256 expiration,uint256 masterNonce,SaleApproval approval)SaleApproval(uint8 protocol,address cosigner,address seller,address marketplace,address fallbackRoyaltyRecipient,address paymentMethod,address tokenAddress,uint256 tokenId,uint256 amount,uint256 itemPrice,uint256 expiration,uint256 marketplaceFeeNumerator,uint256 maxRoyaltyFeeNumerator,uint256 nonce,uint256 masterNonce,uint256 protocolFeeVersion)
```

Note: This is a `PermitTransferFromWithAdditionalData` wrapper type that embeds the full `SaleApproval` as the `approval` field. The nonce and fill tracking are managed by PermitC. There is also a corresponding `PermitOrderWithAdditionalData` variant for partial-fill orders:

```
PermitOrderWithAdditionalData(uint256 tokenType,address token,uint256 id,uint256 amount,uint256 salt,address operator,uint256 expiration,uint256 masterNonce,SaleApproval approval)SaleApproval(uint8 protocol,address cosigner,address seller,address marketplace,address fallbackRoyaltyRecipient,address paymentMethod,address tokenAddress,uint256 tokenId,uint256 amount,uint256 itemPrice,uint256 expiration,uint256 marketplaceFeeNumerator,uint256 maxRoyaltyFeeNumerator,uint256 nonce,uint256 masterNonce,uint256 protocolFeeVersion)
```

---

## Cosignature

A cosignature is a secondary authorization that restricts who can fill an order and when.

### Format

```
Cosignature(uint8 v,bytes32 r,bytes32 s,uint256 expiration,address taker)
```

### Requirements

- The cosigner **must be an EOA** (no smart contract cosigners).
- Use **short expiration** windows (5-10 minutes recommended) for security.
- Set `taker` to `address(0)` to allow any taker, or a specific address to restrict filling.
- When an order has `nonce = 0`, it is a cosigned order -- the cosigner controls authorization.

### Cosigner Compromise

If a cosigner key is compromised, call `destroyCosigner` on the Payment Processor to permanently invalidate all orders using that cosigner on the current chain. This must be done **on every chain** where the cosigner was used.

---

## Bulk Order Signing

Bulk signing allows signing 2 to 1024 orders with a single ECDSA signature using a merkle tree.

### How It Works

1. Choose a tree height from 1 to 10 (supporting 2^height orders).
2. Hash each individual order using its standard typehash.
3. Pad the array to exactly 2^height entries using empty (zeroed) struct hashes.
4. Build a merkle tree from the order hashes.
5. Sign the merkle root using the appropriate bulk typehash.
6. When filling, provide the `BulkOrderProof` with the `orderIndex` and `proof` array.

### Bulk Typehashes

There are 40 bulk typehashes total: 10 tree heights for each of the 4 order types. The typehash name follows the pattern `Bulk{OrderType}_{Height}` where the type string nests the base type inside `SaleApproval[2^height]` (or the corresponding offer type array).

**BulkSaleApproval (heights 1-10):**

| Height | Orders | Typehash |
|---|---|---|
| 1 | 2 | `keccak256("BulkSaleApproval(SaleApproval[2] orders)SaleApproval(...)")` |
| 2 | 4 | `keccak256("BulkSaleApproval(SaleApproval[4] orders)SaleApproval(...)")` |
| 3 | 8 | `keccak256("BulkSaleApproval(SaleApproval[8] orders)SaleApproval(...)")` |
| 4 | 16 | `keccak256("BulkSaleApproval(SaleApproval[16] orders)SaleApproval(...)")` |
| 5 | 32 | `keccak256("BulkSaleApproval(SaleApproval[32] orders)SaleApproval(...)")` |
| 6 | 64 | `keccak256("BulkSaleApproval(SaleApproval[64] orders)SaleApproval(...)")` |
| 7 | 128 | `keccak256("BulkSaleApproval(SaleApproval[128] orders)SaleApproval(...)")` |
| 8 | 256 | `keccak256("BulkSaleApproval(SaleApproval[256] orders)SaleApproval(...)")` |
| 9 | 512 | `keccak256("BulkSaleApproval(SaleApproval[512] orders)SaleApproval(...)")` |
| 10 | 1024 | `keccak256("BulkSaleApproval(SaleApproval[1024] orders)SaleApproval(...)")` |

**BulkItemOfferApproval (heights 1-10):**

Same pattern using `BulkItemOfferApproval(ItemOfferApproval[2^height] orders)ItemOfferApproval(...)`.

**BulkCollectionOfferApproval (heights 1-10):**

Same pattern using `BulkCollectionOfferApproval(CollectionOfferApproval[2^height] orders)CollectionOfferApproval(...)`.

**BulkTokenSetOfferApproval (heights 1-10):**

Same pattern using `BulkTokenSetOfferApproval(TokenSetOfferApproval[2^height] orders)TokenSetOfferApproval(...)`.

Each bulk typehash is computed as `keccak256` of the full type string, which includes the nested base type definition.

---

## Nonce Strategy

### Orderbook Differentiation

Each order book should use a unique nonce prefix to avoid collisions:

```solidity
uint256 orderbookDifferentiator = uint256(keccak256("YOUR_ORDER_BOOK")) << 224;
uint256 orderNonce = orderbookDifferentiator | incrementingCounter;
```

The upper 32 bits identify the order book, and the lower 224 bits are an incrementing counter.

### Bitmap Storage

Nonces are tracked in a bitmap structure for gas efficiency:

- **256 nonces per storage slot** -- each bit represents one nonce
- **First nonce in a new bucket:** ~20,000 gas (cold storage write)
- **Additional nonces in the same bucket within the same transaction:** ~100 gas each (warm storage)

To minimize gas costs, assign nonces sequentially so multiple orders share the same bitmap slot. When revoking nonces, batch revocations within the same 256-nonce bucket when possible.
