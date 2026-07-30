# Payment Processor V3 Data Structures

All structs, enums, and constants used by the Payment Processor protocol.

## Table of Contents

- [Order](#order)
- [SignatureECDSA](#signatureecdsa)
- [Cosignature](#cosignature)
- [FeeOnTop](#feeontop)
- [TokenSetProof](#tokensetproof)
- [BulkOrderProof](#bulkorderproof)
- [PermitContext](#permitcontext)
- [SweepOrder](#sweeporder)
- [SweepItem](#sweepitem)
- [AdvancedOrder](#advancedorder)
- [AdvancedBidOrder](#advancedbidorder)
- [AdvancedSweep](#advancedsweep)
- [AdvancedSweepItem](#advancedsweepitem)
- [CollectionPaymentSettingsParams](#collectionpaymentsettingsparams)
- [RegistryPricingBounds](#registrypricingbounds)
- [Enums](#enums)
- [ProtocolFees](#protocolfees)
- [CollectionPaymentSettings (PP Internal)](#collectionpaymentsettings-pp-internal)
- [CollectionRegistryPaymentSettings](#collectionregistrypaymentsettings)
- [BulkAcceptOffersParams](#bulkacceptoffersparams)
- [Settings Registry Flags](#settings-registry-flags)

---

## Order

The primary order struct representing a listing or offer. This is the core data structure that gets signed via EIP-712.

```solidity
struct Order {
    uint256 protocol;               // 0=ERC721_FILL_OR_KILL, 1=ERC1155_FILL_OR_KILL, 2=ERC1155_FILL_PARTIAL
    address maker;                  // Signer who created the order
    address beneficiary;            // Who receives the NFT. For listings: set to buyer at fill time (not signed). For offers: signed by buyer.
    address marketplace;            // Fee recipient for marketplace
    address fallbackRoyaltyRecipient; // Royalties when ERC2981 unsupported
    address paymentMethod;          // ERC20 address, or address(0) for native currency
    address tokenAddress;           // NFT collection address
    uint256 tokenId;
    uint256 amount;                 // Must be 1 for ERC721, >=1 for ERC1155
    uint256 itemPrice;              // Total price in base units (capped at type(uint240).max)
    uint256 nonce;                  // Set to 0 for cosigned orders
    uint256 expiration;             // Unix timestamp
    uint256 marketplaceFeeNumerator; // BPS, denominator 10000
    uint256 maxRoyaltyFeeNumerator;  // Max royalty the maker will pay (BPS)
    uint256 requestedFillAmount;     // For partial fills
    uint256 minimumFillAmount;       // For partial fills
    uint256 protocolFeeVersion;      // Protocol fee version to use
}
```

---

## SignatureECDSA

Standard ECDSA signature components.

```solidity
struct SignatureECDSA {
    uint256 v;
    bytes32 r;
    bytes32 s;
}
```

---

## Cosignature

A secondary signature from a cosigner that authorizes a specific taker to fill an order. Used for order-book security and taker restriction.

```solidity
struct Cosignature {
    address signer;    // Must be EOA or address(0)
    address taker;     // Taker address or address(0) for any taker
    uint256 expiration; // Unix timestamp in seconds, or 0 for no expiry
    uint256 v;
    bytes32 r;
    bytes32 s;
}
```

When `signer` is `address(0)`, cosigning is disabled for that order.

---

## FeeOnTop

An additional fee added by the taker's marketplace, paid on top of the item price.

```solidity
struct FeeOnTop {
    address recipient;  // address(0) when empty
    uint256 amount;     // Absolute amount in wei, may not exceed item sale price
}
```

---

## TokenSetProof

Merkle proof for token set offers. Allows an offer to target a specific subset of token IDs in a collection.

```solidity
struct TokenSetProof {
    bytes32[] proof;  // Merkle proof, or empty array
}
```

Leaf hash computation:

```solidity
bytes32 leaf = keccak256(abi.encode(collectionAddress, tokenId));
```

---

## BulkOrderProof

Merkle proof for bulk order signing. References a specific order within a bulk-signed merkle tree.

```solidity
struct BulkOrderProof {
    uint256 orderIndex;
    bytes32[] proof;
}
```

---

## PermitContext

Context for PermitC-based orders. Specifies the permit processor contract and the permit nonce.

```solidity
struct PermitContext {
    address permitProcessor;  // PermitC contract address
    uint256 permitNonce;
}
```

---

## SweepOrder

Header struct for sweep operations. All items in a sweep must share these fields.

```solidity
struct SweepOrder {
    uint256 protocol;       // Order protocol (ERC721 or ERC1155)
    address tokenAddress;   // NFT collection address
    address paymentMethod;  // Payment token or address(0)
    address beneficiary;    // Who receives all the NFTs
}
```

## SweepItem

Individual item within a sweep operation. Each item represents a distinct listing to purchase.

```solidity
struct SweepItem {
    address maker;                    // Listing creator
    address marketplace;              // Fee recipient for marketplace
    address fallbackRoyaltyRecipient; // Royalties when ERC2981 unsupported
    uint256 tokenId;
    uint256 amount;                   // Must be 1 for ERC721
    uint256 itemPrice;                // Price in base units
    uint256 nonce;
    uint256 expiration;               // Unix timestamp
    uint256 marketplaceFeeNumerator;  // BPS
    uint256 maxRoyaltyFeeNumerator;   // BPS
    uint256 protocolFeeVersion;
}
```

---

## AdvancedOrder

Wraps an Order with its signature, optional cosignature, and optional permit context for advanced trade flows.

```solidity
struct AdvancedOrder {
    Order saleDetails;
    SignatureECDSA signature;
    Cosignature cosignature;
    PermitContext permitContext;
}
```

## AdvancedBidOrder

Wraps an advanced offer order with its offer type and an optional seller permit signature.

```solidity
struct AdvancedBidOrder {
    uint256 offerType;                   // 0=collection, 1=item, 2=tokenSet
    AdvancedOrder advancedOrder;
    SignatureECDSA sellerPermitSignature;
}
```

## AdvancedSweep

A sweep operation with fee-on-top support and advanced item data.

```solidity
struct AdvancedSweep {
    FeeOnTop feeOnTop;
    SweepOrder sweepOrder;
    AdvancedSweepItem[] items;
}
```

## AdvancedSweepItem

Individual item in an advanced sweep, with full signature, cosignature, permit, and bulk proof support.

```solidity
struct AdvancedSweepItem {
    SweepItem sweepItem;
    SignatureECDSA signature;
    Cosignature cosignature;
    PermitContext permitContext;
    BulkOrderProof bulkOrderProof;
}
```

---

## CollectionPaymentSettingsParams

Parameters for configuring a collection's payment and trading settings in the Settings Registry.

```solidity
struct CollectionPaymentSettingsParams {
    uint8 paymentSettings;                  // PaymentSettings enum value
    uint32 paymentMethodWhitelistId;        // Whitelist ID (for CustomPaymentMethodWhitelist)
    address constrainedPricingPaymentMethod; // Payment method for pricing constraints
    uint16 royaltyBackfillNumerator;        // BPS royalty backfill rate
    address royaltyBackfillReceiver;        // Backfill royalty recipient
    uint16 royaltyBountyNumerator;          // BPS share of royalties to marketplace
    address exclusiveBountyReceiver;        // Exclusive bounty recipient (optional)
    uint16 extraData;                       // Bitfield for flags (see below)
    uint120 collectionMinimumFloorPrice;    // Minimum allowed sale price
    uint120 collectionMaximumCeilingPrice;  // Maximum allowed sale price
}
```

## RegistryPricingBounds

Per-token pricing bounds for individual token IDs within a collection.

```solidity
struct RegistryPricingBounds {
    bool isSet;           // Whether bounds are active
    uint120 floorPrice;   // Minimum allowed price
    uint120 ceilingPrice; // Maximum allowed price
}
```

---

## Enums

### PaymentSettings

Controls which payment methods are allowed for a collection:

| Value | Name | Description |
|---|---|---|
| 0 | DefaultPaymentMethodWhitelist | Uses the protocol's default whitelist (native, WETH, USDC) |
| 1 | AllowAnyPaymentMethod | Accepts any ERC20 or native currency |
| 2 | CustomPaymentMethodWhitelist | Uses a creator-defined whitelist |
| 3 | PricingConstraintsCollectionOnly | Single constrained payment method + collection-level floor/ceiling prices |
| 4 | PricingConstraints | Single constrained payment method + collection-level and token-level floor/ceiling prices |
| 5 | Paused | All trading is paused for this collection |

### Protocol Constants

Order protocol identifiers:

| Value | Name | Description |
|---|---|---|
| 0 | ORDER_PROTOCOLS_ERC721_FILL_OR_KILL | ERC721 order, must fill completely or revert |
| 1 | ORDER_PROTOCOLS_ERC1155_FILL_OR_KILL | ERC1155 order, must fill completely or revert |
| 2 | ORDER_PROTOCOLS_ERC1155_FILL_PARTIAL | ERC1155 order, supports partial fills |

### Offer Types

Used in `AdvancedBidOrder.offerType`:

| Value | Name | Description |
|---|---|---|
| 0 | OFFER_TYPE_COLLECTION_OFFER | Offer for any token in the collection |
| 1 | OFFER_TYPE_ITEM_OFFER | Offer for a specific token ID |
| 2 | OFFER_TYPE_TOKEN_SET_OFFER | Offer for a specific set of token IDs (merkle proof) |

---

## ProtocolFees

Defines the protocol fee parameters for a given fee version. Stored in PP Diamond storage and loaded into `TradeContext` during execution.

```solidity
struct ProtocolFees {
    address protocolFeeReceiver;      // Address that receives protocol fees
    uint16 minimumProtocolFeeBps;     // Minimum fee in BPS applied to a trade
    uint16 marketplaceFeeProtocolTaxBps; // BPS tax applied to marketplace fees
    uint16 feeOnTopProtocolTaxBps;    // BPS tax applied to fee-on-top amounts
    uint48 versionExpiration;         // Timestamp when this fee version expires
}
```

---

## CollectionPaymentSettings (PP Internal)

The struct stored in Payment Processor's Diamond storage. Different from the registry version -- this is the post-sync representation used at trade time.

```solidity
struct CollectionPaymentSettings {
    bool initialized;                   // True once settings are loaded/synced
    PaymentSettings paymentSettings;    // PaymentSettings enum value
    uint32 paymentMethodWhitelistId;    // Whitelist ID for CustomPaymentMethodWhitelist
    address royaltyBackfillReceiver;    // Backfill royalty recipient
    uint16 royaltyBackfillNumerator;    // BPS royalty backfill rate
    uint16 royaltyBountyNumerator;      // BPS share of royalties to marketplace
    uint8 flags;                        // Packed flags (bounty exclusive, block untrusted, use backfill)
}
```

---

## CollectionRegistryPaymentSettings

The struct stored in the Collection Settings Registry's internal storage. This is the registry's own representation, not the same as the PP-internal `CollectionPaymentSettings`.

```solidity
struct CollectionRegistryPaymentSettings {
    bool initialized;                   // True if settings have been initialized
    uint8 paymentSettingsType;          // PaymentSettings enum value (as uint8)
    uint32 paymentMethodWhitelistId;    // Whitelist ID
    address royaltyBackfillReceiver;    // Backfill royalty recipient
    uint16 royaltyBackfillNumerator;    // BPS royalty backfill rate
    uint16 royaltyBountyNumerator;      // BPS share of royalties to marketplace
    uint16 extraData;                   // Bitfield for flags
}
```

---

## BulkAcceptOffersParams

Parameters for a single offer within a `bulkAcceptOffers` transaction. An array of these is encoded to accept multiple offers atomically.

```solidity
struct BulkAcceptOffersParams {
    uint256 offerType;          // 0=collection, 1=item, 2=tokenSet
    Order saleDetails;          // Order execution details
    SignatureECDSA signature;   // Maker signature authorizing the order
    Cosignature cosignature;    // Additional cosignature, as applicable
}
```

---

## Settings Registry Flags

The `extraData` field in `CollectionPaymentSettingsParams` is a `uint16` bitmask containing the following flags:

| Bit | Flag | Value | Description |
|---|---|---|---|
| 0 | FLAG_IS_ROYALTY_BOUNTY_EXCLUSIVE | `1 << 0` | When set, only `exclusiveBountyReceiver` receives the royalty bounty |
| 1 | FLAG_BLOCK_TRADES_FROM_UNTRUSTED_CHANNELS | `1 << 1` | When set, only trusted channels can execute trades for this collection |
| 2 | FLAG_USE_BACKFILL_AS_ROYALTY_SOURCE | `1 << 2` | When set, skips the EIP-2981 `royaltyInfo` query and uses backfill values directly (gas optimization) |
