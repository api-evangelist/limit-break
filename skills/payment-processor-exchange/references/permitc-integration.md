# PermitC Integration Reference

Integration guide for using PermitC with Payment Processor V3 to enable time-bound approvals without `setApprovalForAll`.

---

## Overview

PermitC provides **time-bound, granular token approvals** with built-in expiration. Instead of granting blanket `setApprovalForAll` to the Payment Processor, sellers sign a PermitC permit that authorizes a specific transfer with an expiration timestamp.

Key benefits:

- **No `setApprovalForAll` needed** -- each permit authorizes a specific token transfer
- **Built-in expiration** -- permits automatically become invalid after the expiration timestamp
- **One-click revoke** -- call `lockdown()` on the permit processor to instantly invalidate all outstanding permits

---

## EIP-712 Type for Listings (PermitC)

When creating a PermitC-based listing, the EIP-712 type embeds the `SaleApproval` inside a `PermitTransferFromWithAdditionalData` struct. The full JavaScript EIP-712 type object:

```javascript
const types = {
  PermitTransferFromWithAdditionalData: [
    { name: "tokenType", type: "uint256" },
    { name: "token", type: "address" },
    { name: "id", type: "uint256" },
    { name: "amount", type: "uint256" },
    { name: "nonce", type: "uint256" },
    { name: "operator", type: "address" },
    { name: "expiration", type: "uint256" },
    { name: "masterNonce", type: "uint256" },
    { name: "approval", type: "SaleApproval" }
  ],
  SaleApproval: [
    { name: "protocol", type: "uint8" },
    { name: "cosigner", type: "address" },
    { name: "seller", type: "address" },
    { name: "marketplace", type: "address" },
    { name: "fallbackRoyaltyRecipient", type: "address" },
    { name: "paymentMethod", type: "address" },
    { name: "tokenAddress", type: "address" },
    { name: "tokenId", type: "uint256" },
    { name: "amount", type: "uint256" },
    { name: "itemPrice", type: "uint256" },
    { name: "expiration", type: "uint256" },
    { name: "marketplaceFeeNumerator", type: "uint256" },
    { name: "maxRoyaltyFeeNumerator", type: "uint256" },
    { name: "nonce", type: "uint256" },
    { name: "masterNonce", type: "uint256" },
    { name: "protocolFeeVersion", type: "uint256" }
  ]
};
```

Note: The `SaleApproval` embedded inside the PermitC wrapper uses the full `SaleApproval` type with all fields including `cosigner`, `nonce`, and `masterNonce`. The field name in the wrapper is `approval` (not `additionalData`). The `protocol` field is `uint8` (not `uint256`).

---

## AdvancedOrder Construction

When filling a PermitC-based order, construct the `AdvancedOrder` struct with a populated `permitContext`:

```solidity
AdvancedOrder memory advancedOrder = AdvancedOrder({
    saleDetails: Order({
        protocol: 0,                          // ERC721_FILL_OR_KILL
        maker: seller,
        beneficiary: buyer,
        marketplace: marketplaceAddress,
        fallbackRoyaltyRecipient: address(0),
        paymentMethod: address(0),            // native currency
        tokenAddress: nftContract,
        tokenId: tokenId,
        amount: 1,
        itemPrice: price,
        nonce: 0,                             // not used for permit orders
        expiration: orderExpiration,
        marketplaceFeeNumerator: 250,         // 2.5%
        maxRoyaltyFeeNumerator: 1000,         // 10%
        requestedFillAmount: 0,
        minimumFillAmount: 0,
        protocolFeeVersion: currentFeeVersion
    }),
    signature: SignatureECDSA({
        v: sigV,
        r: sigR,
        s: sigS
    }),
    cosignature: Cosignature({
        signer: address(0),                   // no cosigner
        taker: address(0),
        expiration: 0,
        v: 0,
        r: bytes32(0),
        s: bytes32(0)
    }),
    permitContext: PermitContext({
        permitProcessor: permitCAddress,      // PermitC contract address
        permitNonce: permitNonce              // Nonce from PermitC
    })
});
```

The key difference from standard orders is that `permitContext.permitProcessor` is set to the PermitC contract address (instead of `address(0)`) and `permitContext.permitNonce` is set to the permit nonce.

---

## Signing Flow

1. **Construct the sale approval and permit transfer structs.** The sale approval is embedded as the `approval` field inside the PermitC `PermitTransferFromWithAdditionalData`.

2. **Sign with the PermitC domain separator.** The signature uses the PermitC contract's EIP-712 domain, **not** the Payment Processor's domain separator. This is critical -- using the wrong domain will produce an invalid signature.

3. **Encode via `encodeBuyListingAdvancedCalldata`.** Pass the `AdvancedOrder` with the populated `permitContext` to the Encoder.

4. **Execute the transaction.** Call `buyListingAdvanced` on the Payment Processor with the encoded calldata and the appropriate `msg.value` (for native currency payments).

---

## Partial Fills (ERC1155)

For ERC1155 tokens with partial fill support, use PermitC's order-based permit system:

```solidity
function fillPermittedOrderERC1155(
    bytes calldata signedPermit,
    OrderFillAmounts calldata orderFillAmounts,
    address token,
    uint256 id,
    address owner,
    address to,
    uint256 nonce,
    uint48 expiration,
    bytes32 orderId,
    bytes32 advancedPermitHash
) external returns (uint256 quantityFilled, bool isError);
```

This function is called on the PermitC contract and tracks fill state per-order, allowing multiple partial fills against the same signed permit until the total amount is reached.

---

## Cancellation

### Single Fill-or-Kill Order

Invalidate the permit nonce on the PermitC contract:

```solidity
// On the PermitC contract
function invalidateUnorderedNonce(uint256 nonce) external;
```

### Single Partial Fill Order

Close the permitted order to prevent further fills:

```solidity
// On the PermitC contract
function closePermittedOrder(
    address owner,
    address operator,
    uint256 tokenType,
    address token,
    uint256 id,
    bytes32 orderId
) external;
```

### All Permit Orders

Call `lockdown()` on the permit processor to instantly invalidate all outstanding permits for the caller:

```solidity
// On the Permit Processor
function lockdown() external;
```

This is the nuclear option -- it cancels every outstanding permit-based order in one transaction.
