# Common Configuration Patterns

All examples below are Foundry scripts. The Transfer Validator V5 address is `0x721C008fdff27BF06E7E123956E2Fe03B63342e3`.

## Table of Contents

- [Pattern 1: Basic -- Use Defaults](#pattern-1-basic----use-defaults)
- [Pattern 2: Custom Whitelist Extending Defaults](#pattern-2-custom-whitelist-extending-defaults)
- [Pattern 3: Block All OTC Transfers](#pattern-3-block-all-otc-transfers)
- [Pattern 4: Maximum Security](#pattern-4-maximum-security)
- [Pattern 5: Soulbound (Non-Transferable)](#pattern-5-soulbound-non-transferable)
- [Pattern 6: Blacklist Mode](#pattern-6-blacklist-mode)
- [Pattern 7: Account Freezing](#pattern-7-account-freezing)
- [Pattern 8: Fixed Ruleset (No Auto-Updates)](#pattern-8-fixed-ruleset-no-auto-updates)

---

## Pattern 1: Basic -- Use Defaults

No additional configuration is needed beyond pointing the token contract to V5. The collection automatically gets the default whitelist ruleset with OTC allowed and the default list.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface IToken {
    function setTransferValidator(address validator) external;
}

contract SetupBasic is Script {
    address constant TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

    function run() external {
        address collection = vm.envAddress("COLLECTION");

        vm.startBroadcast();

        // Just point the token to V5 -- defaults handle everything
        IToken(collection).setTransferValidator(TV5);

        vm.stopBroadcast();
    }
}
```

---

## Pattern 2: Custom Whitelist Extending Defaults

Create a custom list with your protocol, and use GO3 to extend (not replace) the default list.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface IToken {
    function setTransferValidator(address validator) external;
}

interface ITransferValidator {
    function createList(string calldata name) external returns (uint48 id);
    function addAccountsToList(uint48 id, uint8 listType, address[] calldata accounts) external;
    function applyListToCollection(address collection, uint48 id) external;
    function setRulesetOfCollection(
        address collection,
        uint8 rulesetId,
        address customRuleset,
        uint8 globalOptions,
        uint16 rulesetOptions
    ) external;
}

contract SetupCustomWhitelist is Script {
    address constant TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

    function run() external {
        address collection = vm.envAddress("COLLECTION");
        address myProtocol = vm.envAddress("MY_PROTOCOL");

        vm.startBroadcast();

        // 1. Set transfer validator on the token
        IToken(collection).setTransferValidator(TV5);

        // 2. Create a new empty list for custom entries
        uint48 listId = ITransferValidator(TV5).createList("My Custom Whitelist");

        // 3. Add custom protocol to the whitelist (list type 1)
        address[] memory accounts = new address[](1);
        accounts[0] = myProtocol;
        ITransferValidator(TV5).addAccountsToList(listId, 1, accounts);

        // 4. Apply list to collection
        ITransferValidator(TV5).applyListToCollection(collection, listId);

        // 5. Enable GO3 (Default List Extension Mode) so our list EXTENDS the default
        //    globalOptions = 0x08 (GO3)
        ITransferValidator(TV5).setRulesetOfCollection(
            collection,
            0,           // default ruleset (auto-update)
            address(0),  // no custom ruleset
            0x08,        // GO3 = Default List Extension Mode
            0            // no WLO options
        );

        vm.stopBroadcast();
    }
}
```

---

## Pattern 3: Block All OTC Transfers

Force all transfers to go through whitelisted protocols. No direct wallet-to-wallet transfers.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface IToken {
    function setTransferValidator(address validator) external;
}

interface ITransferValidator {
    function setRulesetOfCollection(
        address collection,
        uint8 rulesetId,
        address customRuleset,
        uint8 globalOptions,
        uint16 rulesetOptions
    ) external;
}

contract SetupBlockOTC is Script {
    address constant TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

    function run() external {
        address collection = vm.envAddress("COLLECTION");

        vm.startBroadcast();

        // 1. Set transfer validator
        IToken(collection).setTransferValidator(TV5);

        // 2. Set WLO0 (Block All OTC)
        //    rulesetOptions = 0x0001 (WLO0)
        ITransferValidator(TV5).setRulesetOfCollection(
            collection,
            0,           // default ruleset
            address(0),  // no custom ruleset
            0,           // no global options
            0x0001       // WLO0 = Block All OTC
        );

        vm.stopBroadcast();
    }
}
```

---

## Pattern 4: Maximum Security

Block all OTC and restrict receiver types. Two variants:

### 4a: Block OTC + Block Smart Contract Receivers

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface IToken {
    function setTransferValidator(address validator) external;
}

interface ITransferValidator {
    function setRulesetOfCollection(
        address collection,
        uint8 rulesetId,
        address customRuleset,
        uint8 globalOptions,
        uint16 rulesetOptions
    ) external;
}

contract SetupMaxSecurityBlockContracts is Script {
    address constant TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

    function run() external {
        address collection = vm.envAddress("COLLECTION");

        vm.startBroadcast();

        IToken(collection).setTransferValidator(TV5);

        // WLO0 (Block All OTC) + WLO3 (Block Smart Wallet Receivers)
        // rulesetOptions = 0x0001 | 0x0008 = 0x0009
        ITransferValidator(TV5).setRulesetOfCollection(
            collection,
            0,           // default ruleset
            address(0),  // no custom ruleset
            0,           // no global options
            0x0009       // WLO0 + WLO3
        );

        vm.stopBroadcast();
    }
}
```

### 4b: Block OTC + Block Unverified EOA Receivers

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface IToken {
    function setTransferValidator(address validator) external;
}

interface ITransferValidator {
    function setRulesetOfCollection(
        address collection,
        uint8 rulesetId,
        address customRuleset,
        uint8 globalOptions,
        uint16 rulesetOptions
    ) external;
}

contract SetupMaxSecurityBlockUnverifiedEOA is Script {
    address constant TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

    function run() external {
        address collection = vm.envAddress("COLLECTION");

        vm.startBroadcast();

        IToken(collection).setTransferValidator(TV5);

        // WLO0 (Block All OTC) + WLO4 (Block Unverified EOA Receivers)
        // rulesetOptions = 0x0001 | 0x0010 = 0x0011
        ITransferValidator(TV5).setRulesetOfCollection(
            collection,
            0,           // default ruleset
            address(0),  // no custom ruleset
            0,           // no global options
            0x0011       // WLO0 + WLO4
        );

        vm.stopBroadcast();
    }
}
```

---

## Pattern 5: Soulbound (Non-Transferable)

Make tokens permanently bound to the minting wallet.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface IToken {
    function setTransferValidator(address validator) external;
}

interface ITransferValidator {
    function setRulesetOfCollection(
        address collection,
        uint8 rulesetId,
        address customRuleset,
        uint8 globalOptions,
        uint16 rulesetOptions
    ) external;
}

contract SetupSoulbound is Script {
    address constant TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

    function run() external {
        address collection = vm.envAddress("COLLECTION");

        vm.startBroadcast();

        IToken(collection).setTransferValidator(TV5);

        // Soulbound ruleset (ID 2)
        ITransferValidator(TV5).setRulesetOfCollection(
            collection,
            2,           // soulbound ruleset
            address(0),  // no custom ruleset
            0,           // no global options
            0            // no ruleset options (not applicable for soulbound)
        );

        vm.stopBroadcast();
    }
}
```

---

## Pattern 6: Blacklist Mode

Allow all transfers except through blacklisted operators.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface IToken {
    function setTransferValidator(address validator) external;
}

interface ITransferValidator {
    function createList(string calldata name) external returns (uint48 id);
    function addAccountsToList(uint48 id, uint8 listType, address[] calldata accounts) external;
    function applyListToCollection(address collection, uint48 id) external;
    function setRulesetOfCollection(
        address collection,
        uint8 rulesetId,
        address customRuleset,
        uint8 globalOptions,
        uint16 rulesetOptions
    ) external;
}

contract SetupBlacklist is Script {
    address constant TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

    function run() external {
        address collection = vm.envAddress("COLLECTION");
        address blockedOperator = vm.envAddress("BLOCKED_OPERATOR");

        vm.startBroadcast();

        IToken(collection).setTransferValidator(TV5);

        // 1. Switch to Blacklist ruleset (ID 3)
        ITransferValidator(TV5).setRulesetOfCollection(
            collection,
            3,           // blacklist ruleset
            address(0),  // no custom ruleset
            0,           // no global options
            0            // no ruleset options (not applicable for blacklist)
        );

        // 2. Create a custom list for blacklisted operators
        uint48 listId = ITransferValidator(TV5).createList("My Blacklist");

        // 3. Add blocked operator to blacklist (list type 0)
        address[] memory accounts = new address[](1);
        accounts[0] = blockedOperator;
        ITransferValidator(TV5).addAccountsToList(listId, 0, accounts);

        // 4. Apply list to collection
        ITransferValidator(TV5).applyListToCollection(collection, listId);

        vm.stopBroadcast();
    }
}
```

---

## Pattern 7: Account Freezing

Enable account freezing and freeze specific addresses.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface IToken {
    function setTransferValidator(address validator) external;
}

interface ITransferValidator {
    function setRulesetOfCollection(
        address collection,
        uint8 rulesetId,
        address customRuleset,
        uint8 globalOptions,
        uint16 rulesetOptions
    ) external;
    function freezeAccountsForCollection(address collection, address[] calldata accounts) external;
    function unfreezeAccountsForCollection(address collection, address[] calldata accounts) external;
    function getFrozenAccountsByCollection(address collection) external view returns (address[] memory);
    function isAccountFrozenForCollection(address collection, address account) external view returns (bool);
}

contract SetupAccountFreezing is Script {
    address constant TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

    function run() external {
        address collection = vm.envAddress("COLLECTION");
        address suspiciousAddr1 = vm.envAddress("FREEZE_ADDRESS_1");
        address suspiciousAddr2 = vm.envAddress("FREEZE_ADDRESS_2");

        vm.startBroadcast();

        IToken(collection).setTransferValidator(TV5);

        // 1. Enable GO2 (Account Freezing Mode)
        //    globalOptions = 0x04 (GO2)
        ITransferValidator(TV5).setRulesetOfCollection(
            collection,
            0,           // default ruleset
            address(0),  // no custom ruleset
            0x04,        // GO2 = Account Freezing Mode
            0            // no ruleset options
        );

        // 2. Freeze suspicious accounts
        address[] memory toFreeze = new address[](2);
        toFreeze[0] = suspiciousAddr1;
        toFreeze[1] = suspiciousAddr2;
        ITransferValidator(TV5).freezeAccountsForCollection(collection, toFreeze);

        vm.stopBroadcast();
    }
}
```

---

## Pattern 8: Fixed Ruleset (No Auto-Updates)

Pin the collection to a specific ruleset version using ruleset ID 255.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface IToken {
    function setTransferValidator(address validator) external;
}

interface ITransferValidator {
    function setRulesetOfCollection(
        address collection,
        uint8 rulesetId,
        address customRuleset,
        uint8 globalOptions,
        uint16 rulesetOptions
    ) external;
}

contract SetupFixedRuleset is Script {
    address constant TV5 = 0x721C008fdff27BF06E7E123956E2Fe03B63342e3;

    function run() external {
        address collection = vm.envAddress("COLLECTION");
        // The specific deployed ruleset contract to pin to
        address specificRuleset = vm.envAddress("SPECIFIC_RULESET_ADDRESS");

        vm.startBroadcast();

        IToken(collection).setTransferValidator(TV5);

        // Use ruleset ID 255 with a custom ruleset address to opt out of auto-updates
        ITransferValidator(TV5).setRulesetOfCollection(
            collection,
            255,              // opt-out ruleset ID
            specificRuleset,  // pin to this specific ruleset contract
            0,                // no global options
            0                 // no ruleset options
        );

        vm.stopBroadcast();
    }
}
```
