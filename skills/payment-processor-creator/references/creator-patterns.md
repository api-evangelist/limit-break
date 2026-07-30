# Common Creator Configuration Patterns

All examples below are Foundry scripts. The Collection Settings Registry address is `0x9A1D001A842c5e6C74b33F2aeEdEc07F0Cb20BC4`. The Payment Processor V3 address is `0x9a1D00000000fC540e2000560054812452eB5366`.

## Table of Contents

- [Pattern 1: Use Defaults](#pattern-1-use-defaults)
- [Pattern 2: Custom Payment Method Whitelist](#pattern-2-custom-payment-method-whitelist)
- [Pattern 3: Pricing Constraints](#pattern-3-pricing-constraints)
- [Pattern 4: Royalty Bounty](#pattern-4-royalty-bounty)
- [Pattern 5: Trusted Channels Only](#pattern-5-trusted-channels-only)
- [Pattern 6: Full Game Economy](#pattern-6-full-game-economy)

---

## Pattern 1: Use Defaults

No additional configuration is needed. Payment Processor lazily loads default settings on the first trade. The collection automatically gets the default payment method whitelist (native currency, wrapped native, WETH, USDC) with no pricing constraints.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

// No configuration script needed.
// Simply deploy your NFT contract -- the default payment settings
// are loaded automatically on the first trade via Payment Processor.
//
// Default behavior:
//   - Payment methods: native currency, WETH, USDC (native + bridged)
//   - No pricing constraints
//   - EIP-2981 royalties honored if implemented
//   - All channels allowed

contract ExplainDefaults is Script {
    function run() external pure {
        // Nothing to do -- lazy loading handles everything.
    }
}
```

---

## Pattern 2: Custom Payment Method Whitelist

Create a custom whitelist with a game currency token and set it on the collection. Useful when you want to restrict trading to specific ERC20 tokens.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface ISettingsRegistry {
    function createPaymentMethodWhitelist(string calldata name) external returns (uint32);
    function whitelistPaymentMethod(uint32 id, address[] calldata methods, address[] calldata ppSync) external;
    function setCollectionPaymentSettings(
        address tokenAddress,
        CollectionPaymentSettingsParams calldata params,
        bytes32[] calldata dataExtensions,
        bytes[] calldata dataSettings,
        bytes32[] calldata wordExtensions,
        bytes32[] calldata wordSettings,
        address[] calldata ppSync
    ) external;
}

struct CollectionPaymentSettingsParams {
    uint8 paymentSettings;
    uint32 paymentMethodWhitelistId;
    address constrainedPricingPaymentMethod;
    uint16 royaltyBackfillNumerator;
    address royaltyBackfillReceiver;
    uint16 royaltyBountyNumerator;
    address exclusiveBountyReceiver;
    uint16 extraData;
    uint120 collectionMinimumFloorPrice;
    uint120 collectionMaximumCeilingPrice;
}

contract SetupCustomWhitelist is Script {
    address constant REGISTRY = 0x9A1D001A842c5e6C74b33F2aeEdEc07F0Cb20BC4;
    address constant PP = 0x9a1D00000000fC540e2000560054812452eB5366;

    function run() external {
        address collection = vm.envAddress("COLLECTION");
        address gameCurrency = vm.envAddress("GAME_CURRENCY");

        vm.startBroadcast();

        // 1. Create a new payment method whitelist
        uint32 whitelistId = ISettingsRegistry(REGISTRY).createPaymentMethodWhitelist("Game Currency Whitelist");

        // 2. Add the game currency to the whitelist
        address[] memory methods = new address[](1);
        methods[0] = gameCurrency;
        address[] memory ppSync = new address[](1);
        ppSync[0] = PP;
        ISettingsRegistry(REGISTRY).whitelistPaymentMethod(whitelistId, methods, ppSync);

        // 3. Set collection to use CustomPaymentMethodWhitelist (value 2)
        CollectionPaymentSettingsParams memory params = CollectionPaymentSettingsParams({
            paymentSettings: 2,               // CustomPaymentMethodWhitelist
            paymentMethodWhitelistId: whitelistId,
            constrainedPricingPaymentMethod: address(0),
            royaltyBackfillNumerator: 0,
            royaltyBackfillReceiver: address(0),
            royaltyBountyNumerator: 0,
            exclusiveBountyReceiver: address(0),
            extraData: 0,
            collectionMinimumFloorPrice: 0,
            collectionMaximumCeilingPrice: 0
        });

        ISettingsRegistry(REGISTRY).setCollectionPaymentSettings(
            collection,
            params,
            new bytes32[](0),
            new bytes[](0),
            new bytes32[](0),
            new bytes32[](0),
            ppSync
        );

        vm.stopBroadcast();
    }
}
```

---

## Pattern 3: Pricing Constraints

Set collection-level floor and ceiling prices for a game NFT collection. Prevents items from being sold below or above the specified bounds. The `constrainedPricingPaymentMethod` specifies which token the price bounds are denominated in.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface ISettingsRegistry {
    function setCollectionPaymentSettings(
        address tokenAddress,
        CollectionPaymentSettingsParams calldata params,
        bytes32[] calldata dataExtensions,
        bytes[] calldata dataSettings,
        bytes32[] calldata wordExtensions,
        bytes32[] calldata wordSettings,
        address[] calldata ppSync
    ) external;
}

struct CollectionPaymentSettingsParams {
    uint8 paymentSettings;
    uint32 paymentMethodWhitelistId;
    address constrainedPricingPaymentMethod;
    uint16 royaltyBackfillNumerator;
    address royaltyBackfillReceiver;
    uint16 royaltyBountyNumerator;
    address exclusiveBountyReceiver;
    uint16 extraData;
    uint120 collectionMinimumFloorPrice;
    uint120 collectionMaximumCeilingPrice;
}

contract SetupPricingConstraints is Script {
    address constant REGISTRY = 0x9A1D001A842c5e6C74b33F2aeEdEc07F0Cb20BC4;
    address constant PP = 0x9a1D00000000fC540e2000560054812452eB5366;

    function run() external {
        address collection = vm.envAddress("COLLECTION");
        address weth = vm.envAddress("WETH");

        vm.startBroadcast();

        address[] memory ppSync = new address[](1);
        ppSync[0] = PP;

        // PricingConstraintsCollectionOnly (value 3): floor/ceiling at collection level
        CollectionPaymentSettingsParams memory params = CollectionPaymentSettingsParams({
            paymentSettings: 3,                           // PricingConstraintsCollectionOnly
            paymentMethodWhitelistId: 0,                  // uses default whitelist
            constrainedPricingPaymentMethod: weth,        // prices denominated in WETH
            royaltyBackfillNumerator: 0,
            royaltyBackfillReceiver: address(0),
            royaltyBountyNumerator: 0,
            exclusiveBountyReceiver: address(0),
            extraData: 0,
            collectionMinimumFloorPrice: 0.01 ether,      // minimum 0.01 WETH
            collectionMaximumCeilingPrice: 100 ether       // maximum 100 WETH
        });

        ISettingsRegistry(REGISTRY).setCollectionPaymentSettings(
            collection,
            params,
            new bytes32[](0),
            new bytes[](0),
            new bytes32[](0),
            new bytes32[](0),
            ppSync
        );

        vm.stopBroadcast();
    }
}
```

---

## Pattern 4: Royalty Bounty

Share 20% of royalties with marketplaces to incentivize trading volume. Any marketplace that facilitates a trade receives 20% of the royalty amount as a bounty.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface ISettingsRegistry {
    function setCollectionPaymentSettings(
        address tokenAddress,
        CollectionPaymentSettingsParams calldata params,
        bytes32[] calldata dataExtensions,
        bytes[] calldata dataSettings,
        bytes32[] calldata wordExtensions,
        bytes32[] calldata wordSettings,
        address[] calldata ppSync
    ) external;
}

struct CollectionPaymentSettingsParams {
    uint8 paymentSettings;
    uint32 paymentMethodWhitelistId;
    address constrainedPricingPaymentMethod;
    uint16 royaltyBackfillNumerator;
    address royaltyBackfillReceiver;
    uint16 royaltyBountyNumerator;
    address exclusiveBountyReceiver;
    uint16 extraData;
    uint120 collectionMinimumFloorPrice;
    uint120 collectionMaximumCeilingPrice;
}

contract SetupRoyaltyBounty is Script {
    address constant REGISTRY = 0x9A1D001A842c5e6C74b33F2aeEdEc07F0Cb20BC4;
    address constant PP = 0x9a1D00000000fC540e2000560054812452eB5366;

    function run() external {
        address collection = vm.envAddress("COLLECTION");

        vm.startBroadcast();

        address[] memory ppSync = new address[](1);
        ppSync[0] = PP;

        // Share 20% of royalties with marketplaces (2000 BPS = 20%)
        CollectionPaymentSettingsParams memory params = CollectionPaymentSettingsParams({
            paymentSettings: 0,                  // DefaultPaymentMethodWhitelist
            paymentMethodWhitelistId: 0,
            constrainedPricingPaymentMethod: address(0),
            royaltyBackfillNumerator: 0,
            royaltyBackfillReceiver: address(0),
            royaltyBountyNumerator: 2000,        // 20% of royalties to marketplace
            exclusiveBountyReceiver: address(0), // any marketplace gets the bounty
            extraData: 0,                        // not exclusive
            collectionMinimumFloorPrice: 0,
            collectionMaximumCeilingPrice: 0
        });

        ISettingsRegistry(REGISTRY).setCollectionPaymentSettings(
            collection,
            params,
            new bytes32[](0),
            new bytes[](0),
            new bytes32[](0),
            new bytes32[](0),
            ppSync
        );

        vm.stopBroadcast();
    }
}
```

---

## Pattern 5: Trusted Channels Only

Block trades from untrusted frontends. Only channels explicitly added via `addTrustedChannelForCollection` can execute trades for this collection.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface ISettingsRegistry {
    function setCollectionPaymentSettings(
        address tokenAddress,
        CollectionPaymentSettingsParams calldata params,
        bytes32[] calldata dataExtensions,
        bytes[] calldata dataSettings,
        bytes32[] calldata wordExtensions,
        bytes32[] calldata wordSettings,
        address[] calldata ppSync
    ) external;
    function addTrustedChannelForCollection(
        address tokenAddress,
        address[] calldata channels,
        address[] calldata ppSync
    ) external;
}

struct CollectionPaymentSettingsParams {
    uint8 paymentSettings;
    uint32 paymentMethodWhitelistId;
    address constrainedPricingPaymentMethod;
    uint16 royaltyBackfillNumerator;
    address royaltyBackfillReceiver;
    uint16 royaltyBountyNumerator;
    address exclusiveBountyReceiver;
    uint16 extraData;
    uint120 collectionMinimumFloorPrice;
    uint120 collectionMaximumCeilingPrice;
}

contract SetupTrustedChannelsOnly is Script {
    address constant REGISTRY = 0x9A1D001A842c5e6C74b33F2aeEdEc07F0Cb20BC4;
    address constant PP = 0x9a1D00000000fC540e2000560054812452eB5366;

    function run() external {
        address collection = vm.envAddress("COLLECTION");
        address trustedMarketplace = vm.envAddress("TRUSTED_MARKETPLACE");

        vm.startBroadcast();

        address[] memory ppSync = new address[](1);
        ppSync[0] = PP;

        // 1. Enable FLAG_BLOCK_TRADES_FROM_UNTRUSTED_CHANNELS (bit 1 = 0x02)
        CollectionPaymentSettingsParams memory params = CollectionPaymentSettingsParams({
            paymentSettings: 0,
            paymentMethodWhitelistId: 0,
            constrainedPricingPaymentMethod: address(0),
            royaltyBackfillNumerator: 0,
            royaltyBackfillReceiver: address(0),
            royaltyBountyNumerator: 0,
            exclusiveBountyReceiver: address(0),
            extraData: 0x02,                     // FLAG_BLOCK_TRADES_FROM_UNTRUSTED_CHANNELS
            collectionMinimumFloorPrice: 0,
            collectionMaximumCeilingPrice: 0
        });

        ISettingsRegistry(REGISTRY).setCollectionPaymentSettings(
            collection,
            params,
            new bytes32[](0),
            new bytes[](0),
            new bytes32[](0),
            new bytes32[](0),
            ppSync
        );

        // 2. Add trusted marketplace channel
        address[] memory channels = new address[](1);
        channels[0] = trustedMarketplace;
        ISettingsRegistry(REGISTRY).addTrustedChannelForCollection(collection, channels, ppSync);

        vm.stopBroadcast();
    }
}
```

---

## Pattern 6: Full Game Economy

Combine custom payment method whitelist, pricing constraints, trusted channels, and royalty backfill in a single configuration script. This is the most comprehensive setup for a game economy with controlled trading.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";

interface ISettingsRegistry {
    function createPaymentMethodWhitelist(string calldata name) external returns (uint32);
    function whitelistPaymentMethod(uint32 id, address[] calldata methods, address[] calldata ppSync) external;
    function setCollectionPaymentSettings(
        address tokenAddress,
        CollectionPaymentSettingsParams calldata params,
        bytes32[] calldata dataExtensions,
        bytes[] calldata dataSettings,
        bytes32[] calldata wordExtensions,
        bytes32[] calldata wordSettings,
        address[] calldata ppSync
    ) external;
    function addTrustedChannelForCollection(
        address tokenAddress,
        address[] calldata channels,
        address[] calldata ppSync
    ) external;
    function setTokenPricingBounds(
        address tokenAddress,
        uint256[] calldata tokenIds,
        RegistryPricingBounds[] calldata pricingBounds,
        address[] calldata ppSync
    ) external;
}

struct CollectionPaymentSettingsParams {
    uint8 paymentSettings;
    uint32 paymentMethodWhitelistId;
    address constrainedPricingPaymentMethod;
    uint16 royaltyBackfillNumerator;
    address royaltyBackfillReceiver;
    uint16 royaltyBountyNumerator;
    address exclusiveBountyReceiver;
    uint16 extraData;
    uint120 collectionMinimumFloorPrice;
    uint120 collectionMaximumCeilingPrice;
}

struct RegistryPricingBounds {
    bool isSet;
    uint120 floorPrice;
    uint120 ceilingPrice;
}

contract SetupFullGameEconomy is Script {
    address constant REGISTRY = 0x9A1D001A842c5e6C74b33F2aeEdEc07F0Cb20BC4;
    address constant PP = 0x9a1D00000000fC540e2000560054812452eB5366;

    function run() external {
        address collection = vm.envAddress("COLLECTION");
        address gameCurrency = vm.envAddress("GAME_CURRENCY");
        address royaltyReceiver = vm.envAddress("ROYALTY_RECEIVER");
        address trustedMarketplace = vm.envAddress("TRUSTED_MARKETPLACE");
        address gameExchange = vm.envAddress("GAME_EXCHANGE");

        vm.startBroadcast();

        address[] memory ppSync = new address[](1);
        ppSync[0] = PP;

        // 1. Create custom payment method whitelist with game currency
        uint32 whitelistId = ISettingsRegistry(REGISTRY).createPaymentMethodWhitelist("Game Economy Whitelist");

        address[] memory methods = new address[](1);
        methods[0] = gameCurrency;
        ISettingsRegistry(REGISTRY).whitelistPaymentMethod(whitelistId, methods, ppSync);

        // 2. Configure collection with all settings combined
        //    - CustomPaymentMethodWhitelist (not PricingConstraints, because we use custom whitelist)
        //      Note: Use paymentSettings=4 (PricingConstraints) if you want pricing bounds
        //      with the default whitelist. For custom whitelist + pricing, use value 2 with
        //      collection-level bounds in the params.
        //
        //    IMPORTANT: When paymentSettings=2 (CustomPaymentMethodWhitelist), the registry
        //    ignores constrainedPricingPaymentMethod, collectionMinimumFloorPrice, and
        //    collectionMaximumCeilingPrice. Those fields only take effect with
        //    paymentSettings=3 (PricingConstraintsCollectionOnly) or
        //    paymentSettings=4 (PricingConstraints). If you need pricing bounds with a
        //    custom whitelist, use paymentSettings=4 with the default whitelist instead.
        //    - Royalty backfill at 5% (500 BPS)
        //    - FLAG_BLOCK_TRADES_FROM_UNTRUSTED_CHANNELS (bit 1)
        //    - FLAG_USE_BACKFILL_AS_ROYALTY_SOURCE (bit 2) for gas savings
        //    extraData = 0x02 | 0x04 = 0x06
        CollectionPaymentSettingsParams memory params = CollectionPaymentSettingsParams({
            paymentSettings: 2,                          // CustomPaymentMethodWhitelist
            paymentMethodWhitelistId: whitelistId,
            constrainedPricingPaymentMethod: address(0),   // Ignored with paymentSettings=2
            royaltyBackfillNumerator: 500,               // 5% royalty backfill
            royaltyBackfillReceiver: royaltyReceiver,
            royaltyBountyNumerator: 1000,                // 10% of royalties to marketplace
            exclusiveBountyReceiver: address(0),
            extraData: 0x06,                             // BLOCK_UNTRUSTED + USE_BACKFILL
            collectionMinimumFloorPrice: 0,              // Ignored with paymentSettings=2
            collectionMaximumCeilingPrice: 0             // Ignored with paymentSettings=2
        });

        ISettingsRegistry(REGISTRY).setCollectionPaymentSettings(
            collection,
            params,
            new bytes32[](0),
            new bytes[](0),
            new bytes32[](0),
            new bytes32[](0),
            ppSync
        );

        // 3. Add trusted channels
        address[] memory channels = new address[](2);
        channels[0] = trustedMarketplace;
        channels[1] = gameExchange;
        ISettingsRegistry(REGISTRY).addTrustedChannelForCollection(collection, channels, ppSync);

        // 4. Set token-level pricing bounds for legendary items (token IDs 1-3)
        uint256[] memory tokenIds = new uint256[](3);
        tokenIds[0] = 1;
        tokenIds[1] = 2;
        tokenIds[2] = 3;

        RegistryPricingBounds[] memory bounds = new RegistryPricingBounds[](3);
        for (uint256 i = 0; i < 3; i++) {
            bounds[i] = RegistryPricingBounds({
                isSet: true,
                floorPrice: 10000 ether,       // legendary floor: 10,000 game tokens
                ceilingPrice: 5000000 ether     // legendary ceiling: 5M game tokens
            });
        }

        ISettingsRegistry(REGISTRY).setTokenPricingBounds(collection, tokenIds, bounds, ppSync);

        vm.stopBroadcast();
    }
}
```
