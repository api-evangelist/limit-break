# Ruleset System

## Ruleset Security Properties

### Whitelist Ruleset (ID 0 / 4)

The whitelist ruleset is the most commonly used and most configurable. It validates transfers by checking whether the caller (operator) and/or the sender's code hash are on a whitelist.

- **ID 0** (default): Automatically updates when Limit Break deploys new ruleset versions. Collections using ID 0 always get the latest whitelist ruleset logic.
- **ID 4** (explicit): Same validation logic as ID 0, but pinned to a specific version. Does not auto-update when Limit Break deploys improvements. Use ID 4 when you want whitelist behavior but need version stability.

**Default behavior (no WLO flags):** Transfers through whitelisted callers are allowed. Direct wallet-to-wallet (OTC) transfers are also allowed. Non-whitelisted marketplace or protocol callers are blocked.

**What it blocks:** Any transfer initiated by a caller whose address or code hash is not on the whitelist, unless the transfer is an OTC transfer (caller == from or caller == to).

**What it allows:** Transfers through whitelisted operators/contracts, and OTC transfers (unless WLO0 is set).

### Vanilla Ruleset (ID 1)

No transfer restrictions. All transfers are allowed regardless of caller, sender, or receiver. This effectively disables transfer validation while keeping the validator contract attached (useful for temporary pausing of restrictions).

### Soulbound Ruleset (ID 2)

Blocks all transfers. Tokens cannot be transferred between wallets. Minting (from == address(0)) is allowed, but this exception is enforced at the **token contract level** (in the ERC721-C/ERC1155-C `_beforeTokenTransfer` hook), not in the validator's soulbound ruleset itself. The ruleset unconditionally rejects all transfers; the token contract simply skips calling the validator when `from == address(0)`.

### Blacklist Ruleset (ID 3)

The inverse of whitelist. Transfers through any caller are allowed UNLESS the caller's address or code hash is on the blacklist.

**What it blocks:** Transfers initiated by callers whose address or code hash appears on the blacklist. The caller is always checked against the blacklist, including OTC transfers (where `caller == from`).

**What it allows:** Everything else. There is no special OTC exemption -- the caller is always validated against the blacklist regardless of whether it is an OTC or operator-initiated transfer.

---

## Whitelist Validation Logic

The whitelist ruleset has 5 option bits (WLO0-WLO4) producing multiple configuration combinations. Below are the key configurations and their validation behavior.

### Base Configurations (WLO0 scenarios)

#### WLO0 = 0 (OTC Allowed -- Default)

When `caller == from` (direct transfer by owner):
- Transfer is **allowed** -- this is an OTC transfer.

When `caller != from` (operator-initiated transfer):
- Caller address or caller code hash must be on the whitelist.
- If not whitelisted, transfer is **denied**.

With GO3 (Default List Extension Mode) enabled:
- The collection's custom list entries are checked IN ADDITION to the default list entries.
- A caller whitelisted on either the default list or the collection's custom list passes validation.

#### WLO0 = 1 (Block All OTC)

When `caller == from` (direct transfer by owner):
- Transfer is **denied** -- OTC is blocked.
- The caller must still be on the whitelist (same rules as operator-initiated).

When `caller != from` (operator-initiated transfer):
- Caller address or caller code hash must be on the whitelist.
- If not whitelisted, transfer is **denied**.

This forces all transfers to go through whitelisted protocols (e.g., Payment Processor, marketplaces with royalty enforcement).

### WLO1: EIP-7702 Delegate OTC

Controls OTC behavior for accounts with EIP-7702 delegates.

**When WLO0 = 0 (OTC allowed) and the sender has code:**
- WLO1 = 1: 7702-delegated EOAs can OTC freely (non-delegated smart wallets still subject to WLO2).
- WLO1 = 0: 7702-delegated EOAs must have their delegate on the 7702 Delegate Whitelist (list type 4) or 7702 Delegate Whitelist Extension Contracts (list type 5). If not whitelisted, OTC is **denied**.

**When WLO0 = 1 (OTC blocked):**
- WLO1 = 1: 7702-delegated EOAs can OTC even though OTC is otherwise blocked.
- WLO1 = 0: 7702-delegated EOAs are blocked like all other OTC.

Note: Plain EOAs (no code) are unaffected by WLO1 -- they can always OTC when WLO0 = 0.

### WLO2: Smart Wallet OTC

Controls OTC behavior for smart wallets (contracts without 7702 delegation).

**When WLO0 = 0 (OTC allowed) and the sender has code but is NOT a 7702 delegate:**
- WLO2 = 1: Smart wallets can OTC freely.
- WLO2 = 0: Smart wallets must be on the Whitelist (list type 1) or Whitelist Extension Contracts (list type 3). If not whitelisted, OTC is **denied**.

**When WLO0 = 1 (OTC blocked):**
- WLO2 = 1: Smart wallets can OTC even though OTC is otherwise blocked.
- WLO2 = 0: Smart wallets are blocked like all other OTC.

Note: Plain EOAs (no code) are unaffected by WLO2 -- they can always OTC when WLO0 = 0.

### WLO3: Block Smart Wallet Receivers

When WLO3 is enabled:

- If the `to` address has code (is a smart contract), the transfer is **denied**.
- Only EOA receivers (addresses with no code) are allowed.
- This prevents tokens from being sent to smart contract wallets that may not be able to interact with royalty-enforcing marketplaces.

WLO3 applies independently of WLO0. It can be combined with any other options.

### WLO4: Block Unverified EOA Receivers

When WLO4 is enabled:

- If the `to` address is an EOA (no code) and is NOT registered in the EOA Registry, the transfer is **denied**.
- Only EOAs that have verified themselves in the EOA Registry can receive tokens.
- This provides additional assurance that the receiver is a real user-controlled wallet.

WLO4 applies independently of WLO0. It can be combined with any other options.

### Combined Configuration Summary (15 Configurations)

| WLO0 | WLO1 | WLO2 | WLO3 | WLO4 | Behavior |
|---|---|---|---|---|---|
| 0 | - | - | 0 | 0 | Default: Whitelisted callers + OTC allowed |
| 0 | - | - | 1 | 0 | Whitelisted callers + OTC allowed, block smart contract receivers |
| 0 | - | - | 0 | 1 | Whitelisted callers + OTC allowed, block unverified EOA receivers |
| 0 | - | - | 1 | 1 | Whitelisted callers + OTC allowed, block smart contract + unverified EOA receivers |
| 1 | 0 | 0 | 0 | 0 | All OTC blocked, whitelisted callers only |
| 1 | 1 | 0 | 0 | 0 | OTC blocked except EIP-7702 delegates, whitelisted callers only |
| 1 | 0 | 1 | 0 | 0 | OTC blocked except smart wallets, whitelisted callers only |
| 1 | 1 | 1 | 0 | 0 | OTC blocked except EIP-7702 delegates and smart wallets, whitelisted callers only |
| 1 | 0 | 0 | 1 | 0 | All OTC blocked, whitelisted callers only, block smart contract receivers |
| 1 | 1 | 0 | 1 | 0 | OTC blocked except 7702 delegates, whitelisted callers, block smart contract receivers |
| 1 | 0 | 1 | 1 | 0 | OTC blocked except smart wallets, whitelisted callers, block smart contract receivers |
| 1 | 1 | 1 | 1 | 0 | OTC blocked except 7702+smart wallets, whitelisted callers, block smart contract receivers |
| 1 | 0 | 0 | 0 | 1 | All OTC blocked, whitelisted callers only, block unverified EOA receivers |
| 1 | 0 | 0 | 1 | 1 | All OTC blocked, whitelisted callers only, block smart contract + unverified EOA receivers |
| 1 | 1 | 1 | 0 | 1 | OTC blocked except 7702+smart wallets, whitelisted callers, block unverified EOA receivers |

---

## Authorizer System

Authorizers are trusted smart contracts that can override whitelist or blacklist validation for specific transfers. The primary use case is royalty enforcement.

### How Authorizers Work

1. When a transfer is validated, the validator checks if any authorizer on the collection's list can authorize the transfer.
2. An authorizer contract implements a callback that returns whether the transfer should be allowed.
3. If an authorizer approves the transfer, it passes validation even if the caller is not on the whitelist (or is on the blacklist).

### Current Authorizers

- **Seaport Royalty Enforcing Zone**: Authorizes transfers that go through Seaport with proper royalty payment. This is on the default authorizer list (list type 2).

### Authorization Mode (GO0)

- When GO0 is **not set** (default): Authorizers can override validation. A transfer that fails ruleset checks can still pass if an authorizer approves it.
- When GO0 **is set** (Disable Authorization Mode): Authorizers are completely ignored. All transfers must pass the ruleset checks directly. Use this for maximum control.

### Wildcard Operators (GO1)

- When GO1 is **not set** (default): Wildcard operator authorizers apply to all operators. An authorizer that approves any operator can authorize transfers from any caller.
- When GO1 **is set** (No Authorizer Wildcard Operators): Only authorizers tied to specific operators are considered. Wildcard authorizations are ignored.

### Authorizer API Functions

These functions are called by authorizer-enabled protocols (e.g., Seaport) to bracket authorized transfers. The "before" call marks the transfer as pre-authorized in transient storage, and the "after" call clears the authorization.

#### beforeAuthorizedTransfer

```solidity
// Sets the authorized operator for a specific token transfer.
function beforeAuthorizedTransfer(address operator, address token, uint256 tokenId) external;

// Sets the authorized operator for all token transfers on a collection (collection-level).
function beforeAuthorizedTransfer(address operator, address token) external;

// Sets a wildcard operator for a specific token transfer (any operator authorized).
function beforeAuthorizedTransfer(address token, uint256 tokenId) external;
```

#### afterAuthorizedTransfer

```solidity
// Clears the authorized operator for a specific token transfer.
function afterAuthorizedTransfer(address token, uint256 tokenId) external;

// Clears the authorized operator for all token transfers on a collection.
function afterAuthorizedTransfer(address token) external;
```

#### beforeAuthorizedTransferWithAmount

```solidity
// Sets a wildcard operator with an amount for a specific token (ERC1155).
function beforeAuthorizedTransferWithAmount(address token, uint256 tokenId, uint256 amount) external;

// Sets a specific operator with an amount for a specific token (ERC1155).
function beforeAuthorizedTransferWithAmount(address operator, address token, uint256 tokenId, uint256 amount) external;
```

#### afterAuthorizedTransferWithAmount

```solidity
// Clears the authorized operator and amount for a specific token.
function afterAuthorizedTransferWithAmount(address token, uint256 tokenId) external;
```

---

## Automatic Updates vs Opt-Out

### Automatic Updates (Ruleset ID 0)

When a collection uses Ruleset ID 0 (the default):
- The collection automatically uses the latest whitelist ruleset version deployed by Limit Break.
- If a new version of the whitelist ruleset is deployed with bug fixes or improvements, the collection benefits immediately.
- No action required by the collection owner.

### Opt-Out (Ruleset ID 255)

To pin a collection to a specific ruleset version and prevent automatic updates:

1. Set `rulesetId` to `255`.
2. Provide the `customRuleset` address -- the specific deployed ruleset contract to use.
3. The collection will always use that exact ruleset contract, regardless of any future updates.

This is useful for collections that want guaranteed stability and have audited the specific ruleset version they are using.

**Important:** When using any rulesetId other than 255, the `customRuleset` parameter must be `address(0)`.
