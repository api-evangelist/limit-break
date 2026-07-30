# AMM Standard Hook API Reference

## HookTokenSettings

The central configuration struct for the AMM Standard Hook:

```solidity
struct HookTokenSettings {
    bool initialized;              // Always set to true by registry
    bool tradingIsPaused;          // Pause all trading for this token
    bool blockDirectSwaps;         // Block direct (P2P) swaps
    bool checkDisabledPools;       // Check registry for disabled pools
    uint16 tokenFeeBuyBPS;         // Fee on token when bought (in token)
    uint16 tokenFeeSellBPS;        // Fee on token when sold (in token)
    uint16 pairedFeeBuyBPS;        // Fee on paired token when bought (in paired token)
    uint16 pairedFeeSellBPS;       // Fee on paired token when sold (in paired token)
    uint16 minFeeAmount;           // Minimum fee amount (BPS)
    uint16 maxFeeAmount;           // Maximum fee amount (BPS)
    uint56 poolTypeWhitelistId;    // 0 = no restriction
    uint56 pairedTokenWhitelistId; // 0 = no restriction
    uint56 lpWhitelistId;          // 0 = no restriction
}
```

## CreatorHookSettingsRegistry (`ICreatorHookSettingsRegistry`)

The registry is the admin interface for token issuers to configure the AMM Standard Hook.

### Who Can Administer
For any token, settings can be set by:
- The token contract itself
- The token's owner (via `Ownable`)
- The token's default admin (via role-based access)

### Core Functions

#### Set Token Settings
```solidity
function setTokenSettings(
    address token,
    HookTokenSettings calldata settings,
    bytes32[] memory dataExtensions,      // Extension data keys
    bytes[] memory dataSettings,          // Extension data values
    bytes32[] memory wordExtensions,      // Extension word keys
    bytes32[] memory wordSettings,        // Extension word values
    address[] calldata hooksToSync        // Hooks to sync settings to
) external;
```

#### Manage Whitelists

**LP Whitelist:**
```solidity
function createLpWhitelist(string calldata name) external returns (uint256 listId);
function updateLpWhitelist(uint256 listId, address[] calldata accounts, bool add, address[] calldata hooksToSync) external;
function transferLpWhitelistOwnership(uint256 listId, address newOwner) external;
function renounceLpWhitelistOwnership(uint256 listId) external;
```

**Pair Token Whitelist:**
```solidity
function createPairTokenWhitelist(string calldata name) external returns (uint256 listId);
function updatePairTokenWhitelist(uint256 listId, address[] calldata tokens, bool add, address[] calldata hooksToSync) external;
function transferPairTokenWhitelistOwnership(uint256 listId, address newOwner) external;
function renouncePairTokenWhitelistOwnership(uint256 listId) external;
```

**Pool Type Whitelist:**
```solidity
function createPoolTypeWhitelist(string calldata name) external returns (uint256 listId);
function updatePoolTypeWhitelist(uint256 listId, address[] calldata poolTypes, bool add, address[] calldata hooksToSync) external;
function transferPoolTypeWhitelistOwnership(uint256 listId, address newOwner) external;
function renouncePoolTypeWhitelistOwnership(uint256 listId) external;
```

#### Pool Management
```solidity
function setPoolDisabled(address token, bytes32 poolId, bool disable) external;
function isPoolDisabled(bytes32 poolId) external view returns (bool);
```

#### Expansion Settings
```solidity
function setExpansionSettingsOfCollection(
    address token,
    ExpansionWord[] calldata expansionWords,   // key-value pairs (bytes32 key, bytes32 value)
    ExpansionDatum[] calldata expansionDatums  // key-value pairs (bytes32 key, bytes value)
) external;
```

Sets extensible key-value storage for a token without pushing to hooks. Read via `getTokenExtendedData` / `getTokenExtendedWords`.

#### Pricing Bounds
```solidity
function setPricingBounds(
    address token,
    address[] calldata pairTokens,
    uint160[] calldata minSqrtPriceX96,
    uint160[] calldata maxSqrtPriceX96,
    address[] calldata hooksToSync
) external;
```

#### View Functions

**Token Settings:**
```solidity
function getTokenSettings(address token) external view returns (HookTokenSettings memory);
function isTokenInitialized(address token) external view returns (bool);
function getPriceBounds(address token, address pairToken) external view returns (PricingBounds memory);
```

**Extension Data:**
```solidity
function getTokenExtendedData(address token, bytes32[] calldata extensions) external view returns (bytes[] memory data);
function getTokenExtendedWords(address token, bytes32[] calldata extensions) external view returns (bytes32[] memory words);
```

**LP Whitelist:**
```solidity
function getLpWhitelistOwner(uint256 listId) external view returns (address owner);
function getLpsInList(uint256 listId) external view returns (address[] memory accounts);
function isWhitelistedLp(uint256 listId, address account) external view returns (bool);
```

**Pair Token Whitelist:**
```solidity
function getPairTokenWhitelistOwner(uint256 listId) external view returns (address owner);
function getPairTokensInList(uint256 listId) external view returns (address[] memory tokens);
function isWhitelistedPairToken(uint256 listId, address token) external view returns (bool);
```

**Pool Type Whitelist:**
```solidity
function getPoolTypeWhitelistOwner(uint256 listId) external view returns (address owner);
function getPoolTypesInList(uint256 listId) external view returns (address[] memory poolTypes);
function isWhitelistedPoolType(uint256 listId, address poolType) external view returns (bool);
```

## IAMMStandardHook

The hook contract itself (implements `ILimitBreakAMMTokenHook`). Most configuration is done through the Registry, which then syncs to the hook.

### Registry Sync Functions (called by registry)
```solidity
function registryUpdateTokenSettings(address token, HookTokenSettings calldata settings) external;
function registryUpdateWhitelistPairToken(uint256 id, address[] calldata tokens, bool added) external;
function registryUpdateWhitelistLpAddress(uint256 id, address[] calldata addresses, bool added) external;
function registryUpdateWhitelistPoolType(uint256 id, address[] calldata poolTypes, bool added) external;
function registryUpdatePricingBounds(address token, address[] calldata pairTokens, uint160[] calldata min, uint160[] calldata max) external;
```

### View Functions
```solidity
function isWhitelistedPairToken(uint256 whitelistId, address token) external view returns (bool);
function isWhitelistedLiquidityProvider(uint256 whitelistId, address account) external view returns (bool);
```

## Sync Workflow

1. Token issuer calls Registry to set settings / update whitelists
2. Registry stores the settings
3. Registry calls hook's `registryUpdate*` functions to sync
4. Hook's local cache is updated
5. Note: The cache can be out of sync by design -- the hook reads from its local cache, not the registry

## LBAMM Core Token Settings

In addition to the Registry, the token issuer must call LBAMM core to set the hook:

```solidity
// On LBAMM core:
amm.setTokenSettings(
    address(token),
    address(standardHook),
    packedSettings  // Bit flags for which callbacks are active
);
```

The AMM Standard Hook's **actual supported flags** are:
- Bit 0: beforeSwap (for fees/gating)
- Bit 1: afterSwap (for fees)
- Bit 2: addLiquidity (for LP whitelist)
- Bit 5: poolCreation (for pair token / pool type whitelist)
- Bit 7: flashloans (always reverts -- blocks flashloans)
- Bit 8: flashloansValidateFee (always reverts)
- Bit 9: handlerOrderValidate (for pricing bounds on transfer handler orders)

**NOT supported** (setting these will fail or cause reverts):
- Bit 3: removeLiquidity -- `validateRemoveLiquidity` always reverts with `AMMStandardHook__HookFunctionNotSupported`
- Bit 4: collectFees -- `validateCollectFees` always reverts
- Bit 6: hookManagesFees -- not in supported flags
