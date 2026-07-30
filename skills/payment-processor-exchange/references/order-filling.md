# Order Filling and Cancellation Reference

Complete reference for executing trades, encoding calldata, cancelling orders, and querying state on Payment Processor V3.

---

## PaymentProcessor Trade Functions

All trade functions on PaymentProcessor accept a single `bytes calldata data` parameter, which is pre-encoded via the PaymentProcessorEncoder. The executor (`msg.sender`) is always the taker.

### Beneficiary Field

The `beneficiary` field in the `Order` struct determines who receives the NFT:

- **Listings (SaleApproval):** `beneficiary` is **not** part of the signed typehash. It is set at fill time and must be the **buyer's address** (or a delegate who should receive the NFT). Setting it to the seller's address will cause the NFT to transfer back to the seller.
- **Offers (ItemOfferApproval, CollectionOfferApproval, TokenSetOfferApproval):** `beneficiary` **is** part of the signed typehash. The buyer signs the address that should receive the NFT at the time the offer is created.

| Function | Payable | Reentrancy Guard | Description |
|---|---|---|---|
| `buyListing(bytes calldata data)` | Yes | Yes | Buy a single listing |
| `buyListingAdvanced(bytes calldata data)` | Yes | Yes | Buy a single listing with cosignature, permit, and/or fee-on-top |
| `bulkBuyListings(bytes calldata data)` | Yes | Yes | Buy multiple listings in one transaction |
| `bulkBuyListingsAdvanced(bytes calldata data)` | Yes | Yes | Buy multiple listings with advanced features |
| `acceptOffer(bytes calldata data)` | Yes | Yes | Accept a single offer (item, collection, or token set) |
| `acceptOfferAdvanced(bytes calldata data)` | Yes | Yes | Accept an offer with advanced features |
| `bulkAcceptOffers(bytes calldata data)` | Yes | Yes | Accept multiple offers in one transaction |
| `bulkAcceptOffersAdvanced(bytes calldata data)` | Yes | Yes | Accept multiple offers with advanced features |
| `sweepCollection(bytes calldata data)` | Yes | Yes | Sweep multiple listings from one collection |
| `sweepCollectionAdvanced(bytes calldata data)` | Yes | Yes | Sweep with advanced features |

---

## PaymentProcessorEncoder Functions

The Encoder contract at `0x9A1D00C3a699f491037745393a0592AC6b62421D` provides encoding functions for each trade type. Call these off-chain or on-chain to produce the `bytes calldata data` parameter.

### Standard Encoding Functions

```solidity
function encodeBuyListingCalldata(
    address paymentProcessorAddress,
    Order memory saleDetails,
    SignatureECDSA memory signature,
    Cosignature memory cosignature,
    FeeOnTop memory feeOnTop
) external view returns (bytes memory);

function encodeAcceptOfferCalldata(
    address paymentProcessorAddress,
    uint256 offerType,
    Order memory saleDetails,
    SignatureECDSA memory signature,
    TokenSetProof memory tokenSetProof,
    Cosignature memory cosignature,
    FeeOnTop memory feeOnTop
) external view returns (bytes memory);

function encodeBulkBuyListingsCalldata(
    address paymentProcessorAddress,
    Order[] calldata saleDetailsArray,
    SignatureECDSA[] calldata signatures,
    Cosignature[] calldata cosignatures,
    FeeOnTop[] calldata feesOnTop
) external view returns (bytes memory);

function encodeBulkAcceptOffersCalldata(
    address paymentProcessorAddress,
    uint256[] memory offerTypeArray,
    Order[] memory saleDetailsArray,
    SignatureECDSA[] memory signatures,
    TokenSetProof[] memory tokenSetProofsArray,
    Cosignature[] memory cosignaturesArray,
    FeeOnTop[] memory feesOnTopArray
) external view returns (bytes memory);

function encodeSweepCollectionCalldata(
    address paymentProcessorAddress,
    FeeOnTop memory feeOnTop,
    SweepOrder memory sweepOrder,
    SweepItem[] calldata items,
    SignatureECDSA[] calldata signatures,
    Cosignature[] calldata cosignatures
) external view returns (bytes memory);
```

### Advanced Encoding Functions

```solidity
function encodeBuyListingAdvancedCalldata(
    address paymentProcessorAddress,
    AdvancedOrder memory advancedListing,
    BulkOrderProof memory bulkOrderProof,
    FeeOnTop memory feeOnTop
) external view returns (bytes memory);

function encodeAcceptOfferAdvancedCalldata(
    address paymentProcessorAddress,
    AdvancedBidOrder memory advancedBid,
    BulkOrderProof memory bulkOrderProof,
    FeeOnTop memory feeOnTop,
    TokenSetProof memory tokenSetProof
) external view returns (bytes memory);

function encodeBulkBuyListingsAdvancedCalldata(
    address paymentProcessorAddress,
    AdvancedOrder[] memory advancedListingsArray,
    BulkOrderProof[] memory bulkOrderProofs,
    FeeOnTop[] memory feesOnTop
) external view returns (bytes memory);

function encodeBulkAcceptOffersAdvancedCalldata(
    address paymentProcessorAddress,
    AdvancedBidOrder[] memory advancedBidsArray,
    BulkOrderProof[] memory bulkOrderProofs,
    FeeOnTop[] memory feesOnTop,
    TokenSetProof[] memory tokenSetProofs
) external view returns (bytes memory);

function encodeSweepCollectionAdvancedCalldata(
    address paymentProcessorAddress,
    AdvancedSweep memory advancedSweep
) external view returns (bytes memory);
```

### Cancellation Encoding Functions

```solidity
function encodeRevokeSingleNonceCalldata(
    address paymentProcessorAddress,
    uint256 nonce
) external view returns (bytes memory);

function encodeRevokeOrderDigestCalldata(
    address paymentProcessorAddress,
    bytes32 digest
) external view returns (bytes memory);

function encodeDestroyCosignerCalldata(
    address paymentProcessorAddress,
    address cosigner,
    SignatureECDSA memory signature
) external view returns (bytes memory);
```

---

## Sweep Requirements

All items in a sweep operation must share the following properties:

- **Same collection** (`tokenAddress`)
- **Same payment method** (`paymentMethod`)
- **Same beneficiary** (who receives all the NFTs)

### Gas Optimization

Group sweep items by the following fields to reduce gas costs. The protocol batches payments when consecutive items share these properties:

- Marketplace address (combines marketplace fees)
- Seller address (combines seller payments)
- Royalty recipient (combines royalty payments)

---

## Cancellation

### Standard Orders (Non-Cosigned)

| Method | Use Case | Scope |
|---|---|---|
| `revokeSingleNonce(uint256 nonce)` | Cancel a single fill-or-kill order | One order |
| `revokeOrderDigest(bytes32 digest)` | Cancel a partial-fill order by its digest | One order |
| `revokeMasterNonce()` | Cancel all outstanding orders for the caller | All orders |

> **Note:** `revokeMasterNonce()` is called directly on PaymentProcessor with no parameters -- no encoder is needed. It increments the caller's master nonce by 1, invalidating all previously signed orders.

### Cosigned Orders

Cosigned orders (`nonce = 0`) are cancelled **off-chain** by simply stopping the issuance of new cosignatures. Since the cosignature has a short expiration, the order becomes unfillable when the cosignature expires.

### Permit-Based Orders (PermitC)

| Method | Use Case | Contract |
|---|---|---|
| `invalidateUnorderedNonce(uint256 nonce)` | Cancel a single fill-or-kill permit order | PermitC |
| `closePermittedOrder(...)` | Cancel a single partial-fill permit order | PermitC |
| `lockdown()` | Cancel all permit-based orders for the caller | PermitC (Permit Processor) |

---

## Read-Only Functions

Query functions available on the Payment Processor contract:

| Function | Returns | Description |
|---|---|---|
| `getDomainSeparator()` | `bytes32` | EIP-712 domain separator |
| `wrappedNativeCoinAddress()` | `address` | Address of the wrapped native coin for the network |
| `masterNonces(address account)` | `uint256` | Current master nonce for an account |
| `isNonceUsed(address account, uint256 nonce)` | `bool` | Whether a specific nonce has been used or revoked |
| `remainingFillableQuantity(address account, bytes32 orderDigest)` | `PartiallyFillableOrderStatus memory` | State and remaining fillable quantity for a partial-fill order (returns struct with `state` enum and `remainingFillableQuantity` uint248) |
| `collectionPaymentSettings(address tokenAddress)` | `CollectionPaymentSettings memory` | Current payment settings for a collection (returns struct with `initialized`, `paymentSettings`, `paymentMethodWhitelistId`, `royaltyBackfillReceiver`, `royaltyBackfillNumerator`, `royaltyBountyNumerator`, `flags`) |
| `collectionBountySettings(address tokenAddress)` | `(uint16, address)` | Bounty numerator and exclusive receiver |
| `collectionRoyaltyBackfillSettings(address tokenAddress)` | `(uint16, address)` | Backfill numerator and receiver |
| `collectionConstrainedPaymentMethod(address tokenAddress)` | `address` | Payment method used for pricing constraints |
| `isPaymentMethodWhitelisted(uint32 whitelistId, address method)` | `bool` | Whether a payment method is on a whitelist |
| `isDefaultPaymentMethod(address method)` | `bool` | Whether a payment method is on the default whitelist |
| `getDefaultPaymentMethods()` | `address[]` | All default payment methods |
| `getFloorPrice(address tokenAddress, uint256 tokenId)` | `uint256` | Floor price for a specific token |
| `getCeilingPrice(address tokenAddress, uint256 tokenId)` | `uint256` | Ceiling price for a specific token |
| `getProtocolFeeVersion()` | `uint256` | Current protocol fee version |
| `getProtocolFees()` | `ProtocolFees memory` | Current protocol fees (returns struct with `protocolFeeReceiver` address, `minimumProtocolFeeBps` uint16, `marketplaceFeeProtocolTaxBps` uint16, `feeOnTopProtocolTaxBps` uint16, `versionExpiration` uint48) |
| `getProtocolFees(uint256 version)` | `ProtocolFees memory` | Protocol fees for a specific historical version |
