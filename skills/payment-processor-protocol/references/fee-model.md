# Payment Processor V3 Fee Model

Complete reference for protocol fees, royalty enforcement, marketplace fees, and events.

## Table of Contents

- [Protocol Fees](#protocol-fees)
- [Royalty Enforcement](#royalty-enforcement)
- [Royalty Bounties](#royalty-bounties)
- [Marketplace Fees](#marketplace-fees)
- [Events](#events)

---

## Protocol Fees

The protocol charges fees on every trade. These fees are managed by the `FEE_MANAGER` role.

| Parameter | Value | Description |
|---|---|---|
| `minimumProtocolFeeBps` | 25 (0.25%) | Minimum protocol fee per trade |
| `marketplaceFeeProtocolTaxBps` | 1500 (15%) | Tax on marketplace fees collected by the protocol |
| `feeOnTopProtocolTaxBps` | 2500 (25%) | Tax on fee-on-top amounts collected by the protocol |

### Fee Calculation Logic

1. The protocol calculates the exchange fee tax as `marketplaceFee * marketplaceFeeProtocolTaxBps / 10000`.
2. If the exchange fee tax is **greater than or equal to** the minimum protocol fee, the seller's proceeds are unaffected -- the protocol takes its share entirely from the marketplace fee tax.
3. If the exchange fee tax is **less than** the minimum protocol fee, the shortfall is deducted from the seller's proceeds.

### Fee Versioning

Protocol fees support versioning with grace periods. Each order includes a `protocolFeeVersion` field. When protocol fees are updated, a grace period allows existing signed orders to continue using the previous fee version. The fee version is tracked per-collection.

---

## Royalty Enforcement

### EIP-2981 (Primary)

Payment Processor honors the EIP-2981 `royaltyInfo(tokenId, salePrice)` function as the primary source of royalty data. If the collection contract implements EIP-2981, the returned royalty amount and recipient are used.

### Backfill Royalties (Legacy)

For collections that do not implement EIP-2981, creators can configure backfill royalties:

- **`royaltyBackfillNumerator`** -- BPS royalty rate (denominator 10000)
- **`royaltyBackfillReceiver`** -- Address that receives backfill royalties

### FLAG_USE_BACKFILL_AS_ROYALTY_SOURCE

When this flag is set in `extraData`, the protocol skips the on-chain EIP-2981 `royaltyInfo` query entirely and uses the backfill values directly. This saves gas for collections where the backfill values match the EIP-2981 response.

---

## Royalty Bounties

Creators can share a portion of royalties with marketplaces to incentivize trading volume.

- **`royaltyBountyNumerator`** -- BPS share of the royalty amount that goes to the marketplace (denominator 10000). For example, a value of 2000 means 20% of royalties go to the marketplace.
- **`exclusiveBountyReceiver`** -- When set along with `FLAG_IS_ROYALTY_BOUNTY_EXCLUSIVE`, only this address receives the royalty bounty. All other marketplaces get zero bounty.

---

## Marketplace Fees

### Maker Marketplace Fee

The maker's marketplace receives a fee based on `marketplaceFeeNumerator` (BPS, denominator 10000) specified in the signed order. This fee is deducted from the sale price.

### Taker Fee-On-Top

The taker can add a `FeeOnTop` to the trade, which is an additional payment on top of the item price. The fee-on-top is paid to the taker's marketplace or platform. The `amount` may not exceed the item sale price.

---

## Events

### Trade Events (emitted on Payment Processor)

```solidity
event BuyListingERC721(
    address indexed buyer,
    address indexed seller,
    address indexed tokenAddress,
    address beneficiary,
    address paymentCoin,
    uint256 tokenId,
    uint256 salePrice
);

event BuyListingERC1155(
    address indexed buyer,
    address indexed seller,
    address indexed tokenAddress,
    address beneficiary,
    address paymentCoin,
    uint256 tokenId,
    uint256 amount,
    uint256 salePrice
);

event AcceptOfferERC721(
    address indexed seller,
    address indexed buyer,
    address indexed tokenAddress,
    address beneficiary,
    address paymentCoin,
    uint256 tokenId,
    uint256 salePrice
);

event AcceptOfferERC1155(
    address indexed seller,
    address indexed buyer,
    address indexed tokenAddress,
    address beneficiary,
    address paymentCoin,
    uint256 tokenId,
    uint256 amount,
    uint256 salePrice
);
```

### Partial Fill Events (emitted on Payment Processor)

```solidity
event OrderDigestOpened(
    bytes32 indexed orderDigest,
    address indexed account,
    uint256 orderStartAmount
);

event OrderDigestItemsFilled(
    bytes32 indexed orderDigest,
    address indexed account,
    uint256 amountFilled
);

event OrderDigestItemsRestored(
    bytes32 indexed orderDigest,
    address indexed account,
    uint256 amountRestoredToOrder
);
```

### Cancellation Events (emitted on Payment Processor)

```solidity
event MasterNonceInvalidated(
    address indexed account,
    uint256 nonce
);

event NonceInvalidated(
    uint256 indexed nonce,
    address indexed account,
    bool wasCancellation
);

event NonceRestored(
    uint256 indexed nonce,
    address indexed account
);

event DestroyedCosigner(
    address indexed cosigner
);

event PermittedOrderNonceInvalidated(
    uint256 indexed permitNonce,
    uint256 indexed orderNonce,
    address indexed account,
    bool wasCancellation
);

event OrderDigestInvalidated(
    bytes32 indexed orderDigest,
    address indexed account,
    bool wasCancellation
);
```

### Settings Events (emitted on Payment Processor)

```solidity
event UpdatedCollectionPaymentSettings(
    address indexed tokenAddress,
    PaymentSettings paymentSettings,
    uint32 indexed paymentMethodWhitelistId,
    address indexed constrainedPricingPaymentMethod,
    uint16 royaltyBackfillNumerator,
    address royaltyBackfillReceiver,
    uint16 royaltyBountyNumerator,
    address exclusiveBountyReceiver,
    bool blockTradesFromUntrustedChannels,
    bool useRoyaltyBackfillAsRoyaltySource
);

event TrustedChannelAddedForCollection(
    address indexed tokenAddress,
    address indexed channel
);

event TrustedChannelRemovedForCollection(
    address indexed tokenAddress,
    address indexed channel
);

event TrustedPermitProcessorAdded(
    address indexed permitProcessor
);

event TrustedPermitProcessorRemoved(
    address indexed permitProcessor
);
```

### Payment Method Events

```solidity
event CreatedPaymentMethodWhitelist(
    uint32 indexed paymentMethodWhitelistId,
    address indexed whitelistOwner,
    string whitelistName
);

event PaymentMethodAddedToWhitelist(
    uint32 indexed whitelistId,
    address indexed paymentMethod
);

event PaymentMethodRemovedFromWhitelist(
    uint32 indexed whitelistId,
    address indexed paymentMethod
);

event ReassignedPaymentMethodWhitelistOwnership(
    uint32 indexed id,
    address indexed newOwner
);
```

### Pricing Events (emitted on Payment Processor)

```solidity
event UpdatedCollectionLevelPricingBoundaries(
    address indexed tokenAddress,
    uint256 floorPrice,
    uint256 ceilingPrice
);

event UpdatedTokenLevelPricingBoundaries(
    address indexed tokenAddress,
    uint256 indexed tokenId,
    uint256 floorPrice,
    uint256 ceilingPrice
);
```

### Dual Event Sources: UpdatedCollectionPaymentSettings

The `UpdatedCollectionPaymentSettings` event is emitted from **both** PaymentProcessor and CollectionSettingsRegistry, but with **different signatures**.

**PP version** (emitted when settings are synced to PP):

```solidity
event UpdatedCollectionPaymentSettings(
    address indexed tokenAddress,
    PaymentSettings paymentSettings,
    uint32 indexed paymentMethodWhitelistId,
    address indexed constrainedPricingPaymentMethod,
    uint16 royaltyBackfillNumerator,
    address royaltyBackfillReceiver,
    uint16 royaltyBountyNumerator,
    address exclusiveBountyReceiver,
    bool blockTradesFromUntrustedChannels,
    bool useRoyaltyBackfillAsRoyaltySource
);
```

**Registry version** (emitted when creators configure settings via the registry):

```solidity
event UpdatedCollectionPaymentSettings(
    address indexed tokenAddress,
    CollectionPaymentSettingsParams params
);
```

Indexers must listen on **both** contract addresses and decode according to the emitting contract's ABI. The topic0 hashes differ because the parameter lists differ.

---

### Settings Sync Mechanism

Settings are lazily loaded from the Registry to Payment Processor. When registry functions include a `paymentProcessorsToSync` array, updates are pushed automatically. The following functions on PaymentProcessor handle sync:

**Push sync (called by the registry):**

| Function | Description |
|---|---|
| `registrySyncSettings(address tokenAddress)` | Pulls full collection settings from registry into PP storage |
| `registryUpdateWhitelistPaymentMethods(uint32 id, address[] methods, bool added)` | Adds or removes payment methods from a whitelist in PP |
| `registryUpdateTrustedChannels(address tokenAddress, address[] channels, bool added)` | Adds or removes trusted channels for a collection in PP |
| `registryUpdateTrustedPermitProcessors(address[] processors, bool added)` | Adds or removes trusted permit processors in PP |
| `registryUpdateTokenPricingBounds(address tokenAddress, uint256[] tokenIds, RegistryPricingBounds[] bounds)` | Updates token-level pricing bounds in PP |

**Lazy check functions (callable by registry or PP itself):**

| Function | Description |
|---|---|
| `checkSyncCollectionSettings(address tokenAddress)` | If collection is initialized in registry, syncs settings to PP; otherwise clears PP settings |
| `checkSyncTokenPricingBounds(address tokenAddress, uint256 tokenId)` | Syncs a single token's pricing bounds from registry to PP |
| `checkCollectionTrustedChannels(address tokenAddress, address channel)` | Checks if a channel is trusted in the registry and syncs the result to PP |
| `checkTrustedPermitProcessors(address permitProcessor)` | Checks if a permit processor is trusted in the registry and syncs the result to PP |
| `checkWhitelistedPaymentMethod(uint32 whitelistId, address paymentMethod)` | Checks if a payment method is whitelisted in the registry and syncs the result to PP |

All sync and check functions require the caller to be the collection settings registry or PP itself (`_requireCallerIsCollectionSettingsRegistryOrSelf`).

---

### Protocol Fee Events (emitted on Payment Processor)

```solidity
event ProtocolFeesUpdated(
    address indexed newProtocolFeeRecipient,
    uint16 minimumProtocolFeeBps,
    uint16 marketplaceFeeProtocolTaxBps,
    uint16 feeOnTopProtocolTaxBps,
    uint48 gracePeriodExpiration
);
```
