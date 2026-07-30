# TokenMaster Data Structures

## Core Structs

### DeploymentParameters
```
tokenFactory              address
tokenSalt                 bytes32
tokenAddress              address
blockTransactionsFromUntrustedChannels  bool
restrictPairingToLists    bool
poolParams                PoolDeploymentParameters
maxInfrastructureFeeBPS   uint16
```

### PoolDeploymentParameters
```
name                      string
symbol                    string
tokenDecimals             uint8
initialOwner              address
pairedToken               address    // address(0) = native token
initialPairedTokenToDeposit  uint256
encodedInitializationArgs bytes
defaultTransferValidator  address
useRouterForPairedTransfers  bool
partnerFeeRecipient       address
partnerFeeBPS             uint256
```

### BuyOrder
```
tokenMasterToken          address
tokensToBuy               uint256
pairedValueIn             uint256
```

### SellOrder
```
tokenMasterToken          address
tokensToSell              uint256
minimumOut                uint256
```

### SpendOrder
```
tokenMasterToken          address
multiplier                uint256
maxAmountToSpend          uint256
```

### SignedOrder
```
creatorIdentifier         bytes32
tokenMasterOracle         address    // address(0) to skip oracle
baseToken                 address
baseValue                 uint256
maxPerWallet              uint256
maxTotal                  uint256
expiration                uint256
hook                      address
signature                 SignatureECDSA
cosignature               Cosignature
hookExtraData             bytes
oracleExtraData           bytes
```

### PermitTransfer
```
permitProcessor           address
nonce                     uint256
permitAmount              uint256
expiration                uint256
signedPermit              bytes
```

### SignatureECDSA
```
v                         uint256
r                         bytes32
s                         bytes32
```

### Cosignature
```
signer                    address
expiration                uint256
v                         uint256
r                         bytes32
s                         bytes32
```

## Token Settings Flags

| Flag | Bit | Description |
|------|-----|-------------|
| FLAG_DEPLOYED_BY_TOKENMASTER | bit 0 | Token was deployed by TokenMaster |
| FLAG_BLOCK_TRANSACTIONS_FROM_UNTRUSTED_CHANNELS | bit 1 | Block transactions from untrusted channels |
| FLAG_RESTRICT_PAIRING_TO_LISTS | bit 2 | Restrict pairing to approved lists |

## Pause Flags (StandardPool)

| Flag | Bit | Description |
|------|-----|-------------|
| PAUSE_FLAG_BUYS | bit 0 | Pause buy operations |
| PAUSE_FLAG_SELLS | bit 1 | Pause sell operations |
| PAUSE_FLAG_SPENDS | bit 2 | Pause spend operations |

## Roles

| Role | Selector |
|------|----------|
| DEFAULT_ADMIN | `0x0000000000000000000000000000000000000000000000000000000000000000` |
| ORDER_MANAGER | `0x3c6581d000000000000000000000000000000000000000000000000000000000` |
| MINTER_ROLE | `0x9f2df0fed2c77648de5860a4cc508cd0818c85b8b8a1ab4ceeef8d981c8956a6` |
| BURNER_ROLE | `0x3c11d16cbaffd01df69ce1c404f6340ee057498f5f00246190ea54220576a848` |

## ERC-165 Interface IDs

| ID | Description |
|----|-------------|
| `0xb080171e` | General TokenMaster pool |
| `0x1f4456b6` | Standard pool |
| `0x049ac6b3` | Emissions |
| `0xcffb014c` | Promotional pool |
| `0xa18b736c` | Mint/burn |
| `0x511e5ebf` | Stable pool |

## ITokenMasterERC20C Interface

Interface implemented by all token pool contracts deployed through TokenMasterRouter.

```solidity
function PAIRED_TOKEN() external view returns (address);
function buyTokens(address buyer, uint256 pairedTokenIn, uint256 pooledTokenToBuy) external payable returns (uint256 totalCost, uint256 refundByRouterAmount);
function sellTokens(address seller, uint256 pooledTokenToSell, uint256 pairedTokenMinimumOut) external returns (address pairedToken, uint256 pairedValueToSeller, uint256 transferByRouterAmount);
function spendTokens(address spender, uint256 pooledTokenToSpend) external;
function withdrawCreatorShare(address withdrawTo, uint256 withdrawAmount, address infrastructureFeeRecipient, address partnerFeeRecipient) external returns (address pairedToken, uint256 transferByRouterAmountCreator, uint256 transferByRouterAmountInfrastructure, uint256 transferByRouterAmountPartner);
function transferCreatorShareToMarket(uint256 transferAmount, address infrastructureFeeRecipient, address partnerFeeRecipient) external returns (address pairedToken, uint256 transferByRouterAmountInfrastructure, uint256 transferByRouterAmountPartner);
function withdrawFees(address infrastructureFeeRecipient, address partnerFeeRecipient) external returns (address pairedToken, uint256 transferByRouterAmountInfrastructure, uint256 transferByRouterAmountPartner);
function withdrawUnrelatedToken(address tokenAddress, address withdrawTo, uint256 withdrawAmount) external;
function resetPairedTokenApproval() external;
function pairedTokenShares() external view returns (uint256 marketShare, uint256 creatorShare, uint256 infrastructureShare, uint256 partnerShare);
```

These functions are called by the Router, not directly by users. `buyTokens`, `sellTokens`, `spendTokens`, `withdrawCreatorShare`, `transferCreatorShareToMarket`, `withdrawFees`, and `resetPairedTokenApproval` all require `msg.sender == ROUTER`.

## DEPLOYMENT_TYPEHASH

EIP-712 typehash for signed deployments:

```
keccak256("DeploymentParameters(address tokenFactory,bytes32 tokenSalt,address tokenAddress,bool blockTransactionsFromUntrustedChannels,bool restrictPairingToLists)")
```

Only covers the five top-level DeploymentParameters fields (not poolParams or maxInfrastructureFeeBPS). The deployment signer authority (`TOKENMASTER_SIGNER_ROLE`) signs over this digest to authorize a deployment.

## Pool Initialization Parameters

### StandardPoolInitializationParameters
```
initialSupplyRecipient              address
initialSupplyAmount                 uint256
minBuySpreadBPS                     uint256
maxBuySpreadBPS                     uint256
maxBuyFeeBPS                        uint256
maxBuyDemandFeeBPS                  uint256
minSellSpreadBPS                    uint256
maxSellSpreadBPS                    uint256
maxSellFeeBPS                       uint256
maxSpendCreatorShareBPS             uint256
creatorEmissionRateNumerator        uint256
creatorEmissionRateDenominator      uint256
creatorEmissionsHardCap             uint256
initialBuyParameters                StandardPoolBuyParameters
initialSellParameters               StandardPoolSellParameters
initialSpendParameters              StandardPoolSpendParameters
initialPausedState                  uint256
```

### StablePoolInitializationParameters
```
initialSupplyRecipient              address
initialSupplyAmount                 uint256
maxBuyFeeBPS                        uint256
maxSellFeeBPS                       uint256
initialBuyParameters                StablePoolBuyParameters
initialSellParameters               StablePoolSellParameters
stablePairedPricePerToken           PairedPricePerToken
```

**PairedPricePerToken:**
```
numerator                           uint96
denominator                         uint96
```

### PromotionalPoolInitializationParameters
```
initialSupplyRecipient              address
initialSupplyAmount                 uint256
initialBuyParameters                PromotionalPoolBuyParameters
```

**PromotionalPoolBuyParameters:**
```
buyCostPairedTokenNumerator         uint96
buyCostPoolTokenDenominator         uint96
```
