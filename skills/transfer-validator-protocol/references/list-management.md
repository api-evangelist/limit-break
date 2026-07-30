# List Management

## List ID Structure

- **List ID 0**: The default list, managed by Limit Break. Contains the standard set of whitelisted protocols, authorizers, smart wallets, and EIP-7702 delegates. Cannot be renounced. Automatically updated by Limit Break.
- **List IDs 1 - 281474976710655** (uint48 max): Custom lists created by any user. The creator becomes the list owner and can manage entries.

## List Types

Each list has 6 sub-lists identified by a list type ID:

| List Type ID | Name | Description |
|---|---|---|
| 0 | Blacklist | Addresses/code hashes blocked from initiating transfers (used with Blacklist ruleset) |
| 1 | Whitelist | Addresses/code hashes allowed to initiate transfers (used with Whitelist ruleset) |
| 2 | Authorizers | Smart contracts that can authorize transfers (override ruleset validation) |
| 3 | Whitelist Extension Contracts | Addresses of contracts implementing `IWhitelistExtension` that provide additional transfer authorization logic |
| 4 | EIP-7702 Delegate Whitelist | Addresses of EIP-7702 delegates that are whitelisted for OTC transfers |
| 5 | EIP-7702 Delegate Whitelist Extension Contracts | Addresses of contracts implementing `IWhitelistExtension` that extend 7702 delegate whitelist coverage |

## Default List Contents (List ID 0)

The default list (managed by Limit Break) includes:

**Whitelist (type 1):**
- Payment Processor contracts
- TokenMaster contracts
- Other Limit Break ecosystem contracts

**Authorizers (type 2):**
- Seaport Royalty Enforcing Zone contracts

**Whitelist Extension Contracts (type 3):**
- Trusted smart wallet code hashes (Safe, Kernel, etc.)

**EIP-7702 Delegate Whitelist (type 4):**
- Trusted EIP-7702 delegate addresses

**EIP-7702 Delegate Whitelist Extension Contracts (type 5):**
- Extension contracts that provide additional 7702 delegate whitelist validation

---

## List Management Functions

### List Creation

```solidity
// Create a new empty list. Caller becomes the owner.
function createList(string calldata name) external returns (uint48 id);

// Create a new list by copying entries from an existing list.
// NOTE: Only copies list types 0 (Blacklist), 1 (Whitelist), and 2 (Authorizers).
// List types 3-5 (Whitelist Extension, EIP-7702 Delegate lists) are NOT copied.
function createListCopy(string calldata name, uint48 sourceListId) external returns (uint48 id);

// Create a new list by copying only specific list types from an existing list.
function createListCopy(string calldata name, uint48 sourceListId, uint8[] calldata listTypes) external returns (uint48 id);
```

### List Ownership

```solidity
// Transfer ownership of a list to a new address. Only callable by current owner.
function reassignOwnershipOfList(uint48 id, address newOwner) external;

// Permanently renounce ownership, making the list immutable. Cannot be called on list 0.
function renounceOwnershipOfList(uint48 id) external;
```

### Adding Entries

```solidity
// Add accounts (addresses) to a specific list type. Only callable by list owner.
function addAccountsToList(uint48 id, uint8 listType, address[] calldata accounts) external;

// Add code hashes to a specific list type. Only callable by list owner.
// NOTE: codehash cannot be bytes32(0).
function addCodeHashesToList(uint48 id, uint8 listType, bytes32[] calldata codehashes) external;
```

### Removing Entries

```solidity
// Remove accounts from a specific list type. Only callable by list owner.
function removeAccountsFromList(uint48 id, uint8 listType, address[] calldata accounts) external;

// Remove code hashes from a specific list type. Only callable by list owner.
function removeCodeHashesFromList(uint48 id, uint8 listType, bytes32[] calldata codehashes) external;
```

---

## Query Functions

### List Ownership

```solidity
// Returns the owner of a list.
function listOwners(uint48 id) external view returns (address);
```

### Collection Security Policy

```solidity
// Returns the full security policy for a collection as a CollectionSecurityPolicy struct.
function getCollectionSecurityPolicy(address collection) external view returns (CollectionSecurityPolicy memory);

struct CollectionSecurityPolicy {
    uint8 rulesetId;
    uint48 listId;
    address customRuleset;
    uint8 globalOptions;
    uint16 rulesetOptions;
    uint16 tokenType;       // Token standard: 0=unknown, 721=ERC721, 1155=ERC1155
}
```

### List Contents

```solidity
// Get all accounts on a specific list type.
function getListAccounts(uint48 id, uint8 listType) external view returns (address[] memory);

// Get all code hashes on a specific list type.
function getListCodeHashes(uint48 id, uint8 listType) external view returns (bytes32[] memory);
```

### Membership Checks

```solidity
// Check if an account is on a specific list type.
function isAccountInList(uint48 id, uint8 listType, address account) external view returns (bool);

// Check if a code hash is on a specific list type.
function isCodeHashInList(uint48 id, uint8 listType, bytes32 codehash) external view returns (bool);
```

### Collection-Scoped Queries

```solidity
// Get accounts on the list applied to a collection for a specific list type.
function getListAccountsByCollection(address collection, uint8 listType) external view returns (address[] memory);

// Get code hashes on the list applied to a collection for a specific list type.
function getListCodeHashesByCollection(address collection, uint8 listType) external view returns (bytes32[] memory);

// Check if an account is on the list applied to a collection for a specific list type.
function isAccountInListByCollection(address collection, uint8 listType, address account) external view returns (bool);

// Check if a code hash is on the list applied to a collection for a specific list type.
function isCodeHashInListByCollection(address collection, uint8 listType, bytes32 codehash) external view returns (bool);
```

---

## Transfer Simulation Functions

Three overloaded versions of `validateTransferSim` allow simulating transfer validation without executing:

```solidity
// Basic simulation: check if a transfer is allowed for a collection.
function validateTransferSim(
    address collection,
    address caller,
    address from,
    address to
) external view returns (bool isTransferAllowed, bytes4 errorCode);

// Simulation with tokenId: same as above but includes tokenId context.
function validateTransferSim(
    address collection,
    address caller,
    address from,
    address to,
    uint256 tokenId
) external view returns (bool isTransferAllowed, bytes4 errorCode);

// Simulation with tokenId and amount: same as above but includes tokenId and amount context.
function validateTransferSim(
    address collection,
    address caller,
    address from,
    address to,
    uint256 tokenId,
    uint256 amount
) external view returns (bool isTransferAllowed, bytes4 errorCode);
```

**Important:** Authorization mode overrides are NOT taken into account in simulation. The simulation only checks the ruleset logic. If an authorizer would approve the transfer in practice, the simulation may still return `false`.

---

## Validation Entry Points

These functions are called by token contracts in their `_beforeTokenTransfer` hook. They revert on invalid transfers.

```solidity
// Basic validation (no tokenId or amount context).
function validateTransfer(address caller, address from, address to) external view;

// Validation with tokenId context (used by ERC721).
function validateTransfer(address caller, address from, address to, uint256 tokenId) external view;

// Validation with tokenId and amount context (used by ERC1155).
function validateTransfer(address caller, address from, address to, uint256 tokenId, uint256 amount) external;
```

The `msg.sender` is the token collection contract. The validator looks up the collection's security policy based on `msg.sender` and applies the configured ruleset, list, and options.

---

## Additional View Functions

```solidity
// Check if an account is a verified EOA via the EOA Registry.
function isVerifiedEOA(address account) external view returns (bool);

// Returns the last created list ID (uint48). New lists will have IDs > this value.
function lastListId() external view returns (uint48);

// Check if a ruleset contract address is registered with the validator.
function isRulesetRegistered(address ruleset) external view returns (bool);

// Returns the ruleset contract address bound to a given ruleset ID.
// Returns address(0) for rulesetId 255 (fixed/custom).
function boundRuleset(uint8 rulesetId) external view returns (address);
```

---

## Account Freezing

Account freezing allows collection owners to prevent specific addresses from sending or receiving tokens for their collection. **GO2 (Account Freezing Mode)** must be enabled in global options for frozen accounts to be **checked during transfer validation**. However, accounts can be frozen/unfrozen regardless of whether GO2 is currently enabled.

```solidity
// Freeze accounts for a specific collection. Only callable by collection owner/admin.
function freezeAccountsForCollection(address collection, address[] calldata accountsToFreeze) external;

// Unfreeze accounts for a specific collection. Only callable by collection owner/admin.
function unfreezeAccountsForCollection(address collection, address[] calldata accountsToUnfreeze) external;

// Get all frozen accounts for a collection.
function getFrozenAccountsByCollection(address collection) external view returns (address[] memory);

// Check if a specific account is frozen for a collection.
function isAccountFrozenForCollection(address collection, address account) external view returns (bool);
```

When an account is frozen for a collection:
- The account cannot send tokens from that collection.
- The account cannot receive tokens from that collection.
- Freezing is per-collection, not global.

---

## Events

### List Events

```solidity
event CreatedList(uint256 indexed id, string name);
event ReassignedListOwnership(uint256 indexed id, address indexed newOwner);
event AddedAccountToList(uint8 indexed kind, uint48 indexed id, address indexed account);
event AddedCodeHashToList(uint8 indexed kind, uint48 indexed id, bytes32 indexed codehash);
event RemovedAccountFromList(uint8 indexed kind, uint48 indexed id, address indexed account);
event RemovedCodeHashFromList(uint8 indexed kind, uint48 indexed id, bytes32 indexed codehash);
```

### Collection Configuration Events

```solidity
event AppliedListToCollection(address indexed collection, uint48 indexed id);
event SetCollectionRuleset(address indexed collection, uint8 indexed rulesetId, address indexed customRuleset);
event SetCollectionSecurityPolicyOptions(address indexed collection, uint8 globalOptions, uint16 rulesetOptions);
```

### Account Freezing Events

```solidity
event AccountFrozenForCollection(address indexed collection, address indexed account);
event AccountUnfrozenForCollection(address indexed collection, address indexed account);
```
