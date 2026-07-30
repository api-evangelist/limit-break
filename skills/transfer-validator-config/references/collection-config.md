# Collection Configuration API Reference

## Transfer Validator V5 Address

```
0x721C008fdff27BF06E7E123956E2Fe03B63342e3
```

## Core Functions

### setRulesetOfCollection

Sets the ruleset, global options, and ruleset options for a collection.

```solidity
function setRulesetOfCollection(
    address collection,     // Token contract address
    uint8 rulesetId,        // 0=default, 1=vanilla, 2=soulbound, 3=blacklist, 4=whitelist, 255=opt-out
    address customRuleset,  // Must be address(0) unless rulesetId is 255
    uint8 globalOptions,    // Bitmask: GO0=0x01, GO1=0x02, GO2=0x04, GO3=0x08
    uint16 rulesetOptions   // Bitmask: WLO0=0x0001, WLO1=0x0002, WLO2=0x0004, WLO3=0x0008, WLO4=0x0010
) external;
```

**Access control:** Only callable by the collection owner or admin.

**Parameters explained:**
- `rulesetId`: Selects the validation logic. ID 0 auto-updates to latest whitelist. ID 255 requires a custom ruleset address.
- `customRuleset`: Only used with rulesetId 255. For all other rulesetIds, this MUST be `address(0)`.
- `globalOptions`: 8-bit bitmask. OR the desired GO flags together. Set to 0 for no global options.
- `rulesetOptions`: 16-bit bitmask. OR the desired WLO flags together. Only meaningful for whitelist rulesets (ID 0 or 4). Set to 0 for default whitelist behavior.

### applyListToCollection

Applies a list to a collection. The list determines which addresses/code hashes are on the whitelist, blacklist, authorizers, etc.

```solidity
function applyListToCollection(
    address collection,  // Token contract address
    uint48 id            // List ID (0 = default list)
) external;
```

**Access control:** Only callable by the collection owner or admin.

### setTransferValidator (on the token contract)

The token contract itself must point to the Transfer Validator. This is called on the token contract, not on the validator.

```solidity
// Called on the token contract (ERC721-C / ERC1155-C)
function setTransferValidator(address validator) external;
```

Set to `0x721C008fdff27BF06E7E123956E2Fe03B63342e3` for Transfer Validator V5.

### setTokenTypeOfCollection

Sets the token type for a collection. Used for Permit-C integration.

```solidity
function setTokenTypeOfCollection(address collection, uint16 tokenType) external;
```

**Access control:** Only callable by the collection owner or admin.

**Token type values:** `0` = unknown, `721` = ERC721, `1155` = ERC1155.

### setExpansionSettingsOfCollection

Sets expansion words and datums for a collection. These are key-value pairs reserved for future expansion data used by custom rulesets or extensions.

```solidity
function setExpansionSettingsOfCollection(
    address collection,
    ExpansionWord[] calldata expansionWords,
    ExpansionDatum[] calldata expansionDatums
) external;
```

**Access control:** Only callable by the collection owner or admin.

**Struct definitions (from DataTypes.sol):**

```solidity
// Fixed-size 32-byte key-value pair for expansion data.
struct ExpansionWord {
    bytes32 key;
    bytes32 value;
}

// Variable-length key-value pair for expansion data.
struct ExpansionDatum {
    bytes32 key;
    bytes value;
}
```

### getCollectionExpansionWords

Returns the expansion word values for a collection given an array of keys.

```solidity
function getCollectionExpansionWords(
    address tokenAddress,
    bytes32[] calldata keys
) external view returns (bytes32[] memory values);
```

### getCollectionExpansionDatums

Returns the expansion datum values for a collection given an array of keys.

```solidity
function getCollectionExpansionDatums(
    address tokenAddress,
    bytes32[] calldata keys
) external view returns (bytes[] memory values);
```

---

## Default Configuration

When a collection sets its transfer validator to V5 without any further configuration:

- **Ruleset:** Whitelist (ID 0, auto-updating)
- **Global Options:** 0 (no global options)
- **Ruleset Options:** 0 (OTC allowed, no receiver restrictions)
- **List:** Default list (ID 0, managed by Limit Break)
- **Authorization Mode:** Enabled (authorizers can override validation)

This default provides:
- Transfers through Payment Processor and TokenMaster are allowed.
- Seaport with royalty enforcement is authorized.
- Direct wallet-to-wallet (OTC) transfers are allowed.
- All receiver types accepted (smart wallets, unverified EOAs).

---

## Step-by-Step Workflows

### 1. Deploy New Token with V5

```solidity
// In your token contract constructor or initializer:
setTransferValidator(0x721C008fdff27BF06E7E123956E2Fe03B63342e3);
```

That's it for default configuration. The token now uses V5 with the default whitelist ruleset and default list.

### 2. Migrate Existing Token to V5

```solidity
// Step 1: Update the transfer validator on the token contract
IToken(collection).setTransferValidator(0x721C008fdff27BF06E7E123956E2Fe03B63342e3);

// Step 2 (optional): Configure ruleset and options
ITransferValidator(0x721C008fdff27BF06E7E123956E2Fe03B63342e3).setRulesetOfCollection(
    collection,
    0,           // default ruleset (auto-update)
    address(0),  // no custom ruleset
    0,           // no global options
    0            // no ruleset options (OTC allowed)
);

// Step 3 (optional): Apply a custom list
ITransferValidator(0x721C008fdff27BF06E7E123956E2Fe03B63342e3).applyListToCollection(
    collection,
    0            // default list
);
```

### 3. Configure Custom Ruleset Options

```solidity
address TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

// Block all OTC transfers + allow smart wallet OTC
ITransferValidator(TV5).setRulesetOfCollection(
    collection,
    0,           // default whitelist ruleset
    address(0),  // no custom ruleset
    0,           // no global options
    0x0001 | 0x0004  // WLO0 (Block All OTC) + WLO2 (Allow Smart Wallet OTC) = 0x0005
);
```

### 4. Opt Out of Automatic Updates

```solidity
address TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;
address SPECIFIC_RULESET = 0x...; // The specific whitelist ruleset contract address

ITransferValidator(TV5).setRulesetOfCollection(
    collection,
    255,              // opt-out ruleset ID
    SPECIFIC_RULESET, // pin to this specific ruleset contract
    0,                // global options
    0                 // ruleset options
);
```

### 5. Create and Apply Custom List

```solidity
address TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

// Step 1: Create a new list (copies default list entries)
uint48 listId = ITransferValidator(TV5).createListCopy("My Custom List", 0);

// Step 2: Add custom protocol to whitelist
address[] memory accounts = new address[](1);
accounts[0] = MY_PROTOCOL_ADDRESS;
ITransferValidator(TV5).addAccountsToList(listId, 1, accounts); // listType 1 = Whitelist

// Step 3: Apply list to collection
ITransferValidator(TV5).applyListToCollection(collection, listId);
```

### 6. Extend Default List (Instead of Replacing)

```solidity
address TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

// Step 1: Create a new empty list (do NOT copy from default)
uint48 listId = ITransferValidator(TV5).createList("My Extension List");

// Step 2: Add your custom protocol addresses to the whitelist
address[] memory accounts = new address[](1);
accounts[0] = MY_PROTOCOL_ADDRESS;
ITransferValidator(TV5).addAccountsToList(listId, 1, accounts); // listType 1 = Whitelist

// Step 3: Apply list to collection
ITransferValidator(TV5).applyListToCollection(collection, listId);

// Step 4: Enable GO3 (Default List Extension Mode) so the custom list EXTENDS the default list
ITransferValidator(TV5).setRulesetOfCollection(
    collection,
    0,           // default ruleset
    address(0),  // no custom ruleset
    0x08,        // GO3 = Default List Extension Mode
    0            // no ruleset options
);
```

This way, the collection's whitelist includes everything on the default list PLUS the custom entries.

### 7. Freeze Accounts

```solidity
address TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

// Step 1: Enable Account Freezing Mode (GO2)
ITransferValidator(TV5).setRulesetOfCollection(
    collection,
    0,           // default ruleset
    address(0),  // no custom ruleset
    0x04,        // GO2 = Account Freezing Mode
    0            // no ruleset options
);

// Step 2: Freeze specific accounts
address[] memory toFreeze = new address[](2);
toFreeze[0] = SUSPICIOUS_ADDRESS_1;
toFreeze[1] = SUSPICIOUS_ADDRESS_2;
ITransferValidator(TV5).freezeAccountsForCollection(collection, toFreeze);

// Step 3 (later): Unfreeze accounts
address[] memory toUnfreeze = new address[](1);
toUnfreeze[0] = SUSPICIOUS_ADDRESS_1;
ITransferValidator(TV5).unfreezeAccountsForCollection(collection, toUnfreeze);
```

### 8. Simulate Transfers

```solidity
address TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

// Check if a transfer would be allowed
(bool allowed, bytes4 errorCode) = ITransferValidator(TV5).validateTransferSim(
    collection, // The token collection address
    caller,     // The address initiating the transfer (operator)
    from,       // The current token owner
    to          // The intended recipient
);

// Note: Authorization mode overrides are NOT taken into account in simulation.
// If an authorizer would approve the transfer in practice, the simulation may still return false.
```
