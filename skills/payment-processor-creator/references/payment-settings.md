# Creator Payment Settings API

Complete API reference for configuring collection payment settings, pricing constraints, royalties, trusted channels, and payment method whitelists. All functions are called on the **Collection Settings Registry** at `0x9A1D001A842c5e6C74b33F2aeEdEc07F0Cb20BC4`.

---

## Collection Payment Settings

The primary function for configuring a collection's trading parameters:

```solidity
function setCollectionPaymentSettings(
    address tokenAddress,
    CollectionPaymentSettingsParams calldata params,
    bytes32[] calldata dataExtensions,
    bytes[] calldata dataSettings,
    bytes32[] calldata wordExtensions,
    bytes32[] calldata wordSettings,
    address[] calldata paymentProcessorsToSync
) external;
```

The `params` struct controls all collection-level settings. The extensions and settings arrays are for future extensibility (pass empty arrays for standard use). The `paymentProcessorsToSync` array should contain the Payment Processor address to push updates immediately.

### Payment Settings Types

| Type | Value | Behavior |
|---|---|---|
| DefaultPaymentMethodWhitelist | 0 | Uses the protocol's default whitelist: native currency, wrapped native, WETH, USDC (native + bridged) |
| AllowAnyPaymentMethod | 1 | Accepts any ERC20 token or native currency as payment |
| CustomPaymentMethodWhitelist | 2 | Uses a creator-defined whitelist (set `paymentMethodWhitelistId` to the ID) |
| PricingConstraintsCollectionOnly | 3 | Single constrained payment method + collection-level floor/ceiling price enforcement |
| PricingConstraints | 4 | Single constrained payment method + both collection-level and token-level floor/ceiling prices |
| Paused | 5 | All trading is paused for the collection |

---

## Payment Method Whitelist Management

### Create a Whitelist

```solidity
function createPaymentMethodWhitelist(
    string calldata name
) external returns (uint32 whitelistId);
```

Creates a new empty payment method whitelist. The caller becomes the whitelist owner.

### Add Payment Methods

```solidity
function whitelistPaymentMethod(
    uint32 paymentMethodWhitelistId,
    address[] calldata paymentMethods,
    address[] calldata paymentProcessorsToSync
) external;
```

Adds one or more payment methods to a whitelist. Only the whitelist owner can call this.

### Remove Payment Methods

```solidity
function unwhitelistPaymentMethod(
    uint32 paymentMethodWhitelistId,
    address[] calldata paymentMethods,
    address[] calldata paymentProcessorsToSync
) external;
```

### Sync Removals to Payment Processor

```solidity
function syncRemovedPaymentMethodsFromWhitelist(
    uint32 paymentMethodWhitelistId,
    address[] calldata paymentMethods,
    address[] calldata paymentProcessorsToSync
) external;
```

Forces the Payment Processor to recognize removed payment methods. Use this if removals were made without including the PP in `paymentProcessorsToSync`.

### Transfer Whitelist Ownership

```solidity
function reassignOwnershipOfPaymentMethodWhitelist(
    uint32 paymentMethodWhitelistId,
    address newOwner
) external;
```

### Renounce Whitelist Ownership

```solidity
function renounceOwnershipOfPaymentMethodWhitelist(
    uint32 paymentMethodWhitelistId
) external;
```

Permanently locks the whitelist -- no further modifications possible.

---

## Pricing Constraints

### Token-Level Pricing Bounds

```solidity
function setTokenPricingBounds(
    address tokenAddress,
    uint256[] calldata tokenIds,
    RegistryPricingBounds[] calldata pricingBounds,
    address[] calldata paymentProcessorsToSync
) external;
```

Sets individual floor and ceiling prices per token ID. Requires the collection's `paymentSettings` to be set to `PricingConstraints` (value 4).

### Collection-Level Pricing Bounds

Collection-level floor and ceiling prices are set via the `collectionMinimumFloorPrice` and `collectionMaximumCeilingPrice` fields in `CollectionPaymentSettingsParams`. These apply when `paymentSettings` is set to `PricingConstraintsCollectionOnly` (3) or `PricingConstraints` (4).

---

## Royalty Configuration

### Backfill Royalties

For collections that do not implement EIP-2981, set backfill royalties via the `CollectionPaymentSettingsParams`:

- **`royaltyBackfillNumerator`** -- BPS royalty rate (denominator 10000)
- **`royaltyBackfillReceiver`** -- Address that receives the backfill royalties

### FLAG_USE_BACKFILL_AS_ROYALTY_SOURCE

Set bit 2 in `extraData` to skip the on-chain EIP-2981 `royaltyInfo` query and use backfill values directly. This is a gas optimization for collections where the backfill values are known to match the EIP-2981 response.

### Royalty Bounties

Share a portion of royalties with marketplaces to incentivize trading volume:

- **`royaltyBountyNumerator`** -- BPS share of the royalty amount sent to the marketplace (denominator 10000)
- **`exclusiveBountyReceiver`** -- When combined with `FLAG_IS_ROYALTY_BOUNTY_EXCLUSIVE` (bit 0 in `extraData`), restricts the bounty to only this address

---

## Trusted Channels

### Add Trusted Channels

```solidity
function addTrustedChannelForCollection(
    address tokenAddress,
    address[] calldata channels,
    address[] calldata paymentProcessorsToSync
) external;
```

### Remove Trusted Channels

```solidity
function removeTrustedChannelForCollection(
    address tokenAddress,
    address[] calldata channels,
    address[] calldata paymentProcessorsToSync
) external;
```

### FLAG_BLOCK_TRADES_FROM_UNTRUSTED_CHANNELS

Set bit 1 in `extraData` to restrict trading to only trusted channels. When enabled, only addresses added via `addTrustedChannelForCollection` can execute trades as `msg.sender` for this collection.

---

## Permit Processors

### Add Trusted Permit Processors

```solidity
function addTrustedPermitProcessors(
    address[] calldata permitProcessors,
    address[] calldata paymentProcessorsToSync
) external;
```

### Remove Trusted Permit Processors

```solidity
function removeTrustedPermitProcessors(
    address[] calldata permitProcessors,
    address[] calldata paymentProcessorsToSync
) external;
```
