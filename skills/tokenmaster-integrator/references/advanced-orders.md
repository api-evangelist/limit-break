# TokenMaster Advanced Orders

## EIP-712 Signing Reference

### Domain Separator
```
name:              "TokenMasterRouter"
version:           "1"
chainId:           <chain id>
verifyingContract:  <Router address>
```

### EIP-712 Types

**BuyTokenMasterToken:**
```
BuyTokenMasterToken(
    bytes32 creatorBuyIdentifier,
    address tokenMasterToken,
    address tokenMasterOracle,
    address baseToken,
    uint256 baseValue,
    uint256 maxPerWallet,
    uint256 maxTotal,
    uint256 expiration,
    address hook,
    address cosigner
)
```

**SellTokenMasterToken:** Same pattern with `creatorSellIdentifier`.

**SpendTokenMasterToken:** Same pattern with `creatorSpendIdentifier`.

### Signing Requirements
- Signed by an authorized signer set via `router.setOrderSigner(token, signerAddress, true)`
- `maxPerWallet` / `maxTotal`: tokens for buy/sell, multipliers for spend
- `tokenMasterOracle = address(0)` to skip oracle adjustment

---

## Cosigning

### Cosignature Type
```
Cosignature(
    uint8 v,
    bytes32 r,
    bytes32 s,
    uint256 expiration,
    address executor
)
```

### Important Notes
- The EIP-712 type above defines the **signed data** (expiration + executor). The `v`, `r`, `s` fields are the cosigner's signature over this data. The `executor` field restricts who can submit the transaction (`msg.sender` must match).
- The Solidity `Cosignature` struct has a `signer` field (the cosigner's address, for verification) -- this is NOT the same as `executor` in the signed data.
- Cosigner is NOT permissioned on Router -- it just needs to match the `signer` field in the Cosignature struct

### Use Cases
- Restrict order execution to a specific app/frontend
- Require completed off-chain actions before allowing on-chain execution
- Time-limited execution windows via cosignature expiration

---

## PermitC for Buys

### PermitC Transfer Type
```
PermitTransferFromWithAdditionalData(
    uint256 tokenType,
    address token,
    uint256 id,
    uint256 amount,
    uint256 nonce,
    address operator,
    uint256 expiration,
    uint256 masterNonce,
    AdvancedBuyOrder advancedBuyOrder
)
```

### Embedded AdvancedBuyOrder Type
```
AdvancedBuyOrder(
    address tokenMasterToken,
    uint256 tokensToBuy,
    uint256 pairedValueIn,
    bytes32 creatorBuyIdentifier,
    address hook,
    uint8 buyOrderSignatureV,
    bytes32 buyOrderSignatureR,
    bytes32 buyOrderSignatureS
)
```

### Important
- Domain separator comes from the **PermitC contract**, NOT the Router
- Used to combine token approval and buy in a single gasless signature

---

## Oracle Integration

### Interface
```solidity
ITokenMasterOracle.adjustValue(
    uint256 transactionType,  // Buy=0, Sell=1, Spend=2
    address executor,
    address tokenMasterToken,
    address baseToken,
    uint256 baseValue,
    bytes calldata oracleExtraData
) external view returns (uint256 tokenValue);
```

### Important Notes
- View function only (cannot modify state)
- Returns the adjusted token value

### Use Cases
- Price pegging to external assets (e.g., USD stablecoin peg)
- Dynamic discounts based on user behavior or time
- External price feed integration (e.g., Chainlink)
