# AMM Standard Hook Configuration Examples

## Example 1: Basic Token with Buy/Sell Fees

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {Script} from "forge-std/Script.sol";

contract ConfigureTokenFees is Script {
    address constant AMM = 0x...;
    address constant REGISTRY = 0x...;
    address constant STANDARD_HOOK = 0x...;
    address constant MY_TOKEN = 0x...;

    function run() external {
        vm.startBroadcast();

        // 1. Set hook settings in registry
        HookTokenSettings memory settings = HookTokenSettings({
            initialized: true,
            tradingIsPaused: false,
            blockDirectSwaps: false,
            checkDisabledPools: false,
            tokenFeeBuyBPS: 200,   // 2% fee when token is bought (paid in token)
            tokenFeeSellBPS: 300,  // 3% fee when token is sold (paid in token)
            pairedFeeBuyBPS: 0,    // No paired token fees
            pairedFeeSellBPS: 0,
            minFeeAmount: 0,
            maxFeeAmount: 0,
            poolTypeWhitelistId: 0,       // No pool type restrictions
            pairedTokenWhitelistId: 0,    // No paired token restrictions
            lpWhitelistId: 0              // No LP restrictions
        });

        address[] memory hooksToSync = new address[](1);
        hooksToSync[0] = STANDARD_HOOK;

        ICreatorHookSettingsRegistry(REGISTRY).setTokenSettings(
            MY_TOKEN,
            settings,
            new bytes32[](0), new bytes[](0),     // No extensions
            new bytes32[](0), new bytes32[](0),   // No word extensions
            hooksToSync
        );

        // 2. Set token settings in LBAMM core
        uint32 packedSettings =
            (1 << 0) |  // beforeSwap
            (1 << 1);   // afterSwap
        ILimitBreakAMMTokenSettings(AMM).setTokenSettings(MY_TOKEN, STANDARD_HOOK, packedSettings);

        vm.stopBroadcast();
    }
}
```

## Example 2: Paired Token Whitelist + LP Whitelist

Restrict which tokens can be paired and who can provide liquidity.

```solidity
contract ConfigureWhitelists is Script {
    function run() external {
        vm.startBroadcast();

        // 1. Create pair token whitelist
        uint256 pairListId = ICreatorHookSettingsRegistry(REGISTRY)
            .createPairTokenWhitelist("My Token Pairs");

        address[] memory pairTokens = new address[](2);
        pairTokens[0] = WETH;
        pairTokens[1] = USDC;

        address[] memory hooksToSync = new address[](1);
        hooksToSync[0] = STANDARD_HOOK;

        ICreatorHookSettingsRegistry(REGISTRY).updatePairTokenWhitelist(
            pairListId, pairTokens, true, hooksToSync
        );

        // 2. Create LP whitelist
        uint256 lpListId = ICreatorHookSettingsRegistry(REGISTRY)
            .createLpWhitelist("Approved LPs");

        address[] memory lps = new address[](2);
        lps[0] = LP_1;
        lps[1] = LP_2;

        ICreatorHookSettingsRegistry(REGISTRY).updateLpWhitelist(
            lpListId, lps, true, hooksToSync
        );

        // 3. Set settings with whitelist IDs
        HookTokenSettings memory settings = HookTokenSettings({
            initialized: true,
            tradingIsPaused: false,
            blockDirectSwaps: false,
            checkDisabledPools: false,
            tokenFeeBuyBPS: 100,
            tokenFeeSellBPS: 100,
            pairedFeeBuyBPS: 0,
            pairedFeeSellBPS: 0,
            minFeeAmount: 0,
            maxFeeAmount: 0,
            poolTypeWhitelistId: 0,
            pairedTokenWhitelistId: uint56(pairListId),
            lpWhitelistId: uint56(lpListId)
        });

        ICreatorHookSettingsRegistry(REGISTRY).setTokenSettings(
            MY_TOKEN, settings,
            new bytes32[](0), new bytes[](0),
            new bytes32[](0), new bytes32[](0),
            hooksToSync
        );

        // 4. Set core flags (need pool creation + liquidity hooks for whitelists)
        // NOTE: Do NOT set bit 3 (removeLiquidity) -- AMMStandardHook reverts on it
        uint32 packedSettings =
            (1 << 0) |  // beforeSwap
            (1 << 1) |  // afterSwap
            (1 << 2) |  // addLiquidity (for LP whitelist)
            (1 << 5) |  // poolCreation (for pair token whitelist)
            (1 << 9);   // handlerOrderValidate (for pricing bounds)

        ILimitBreakAMMTokenSettings(AMM).setTokenSettings(MY_TOKEN, STANDARD_HOOK, packedSettings);

        vm.stopBroadcast();
    }
}
```

## Example 3: Price Bounds

Restrict the price range at which pools can be created or traded.

```solidity
contract ConfigurePriceBounds is Script {
    function run() external {
        vm.startBroadcast();

        address[] memory pairTokens = new address[](1);
        pairTokens[0] = WETH;

        uint160[] memory minPrices = new uint160[](1);
        uint160[] memory maxPrices = new uint160[](1);

        // Set price bounds: token can only trade between 0.001 and 1000 ETH
        minPrices[0] = 2505414483750479;        // sqrt(0.001) * 2^96
        maxPrices[0] = 2505414483750479311864;   // sqrt(1000) * 2^96

        address[] memory hooksToSync = new address[](1);
        hooksToSync[0] = STANDARD_HOOK;

        ICreatorHookSettingsRegistry(REGISTRY).setPricingBounds(
            MY_TOKEN, pairTokens, minPrices, maxPrices, hooksToSync
        );

        vm.stopBroadcast();
    }
}
```

## Example 4: Pause Trading + Disable Specific Pool

```solidity
contract PauseAndDisable is Script {
    function run() external {
        vm.startBroadcast();

        // Pause all trading
        HookTokenSettings memory settings = ICreatorHookSettingsRegistry(REGISTRY).getTokenSettings(MY_TOKEN);
        settings.tradingIsPaused = true;
        settings.checkDisabledPools = true;

        address[] memory hooksToSync = new address[](1);
        hooksToSync[0] = STANDARD_HOOK;

        ICreatorHookSettingsRegistry(REGISTRY).setTokenSettings(
            MY_TOKEN, settings,
            new bytes32[](0), new bytes[](0),
            new bytes32[](0), new bytes32[](0),
            hooksToSync
        );

        // Or disable a specific pool (without pausing everything)
        ICreatorHookSettingsRegistry(REGISTRY).setPoolDisabled(MY_TOKEN, poolId, true);

        vm.stopBroadcast();
    }
}
```

## Trading Pause vs Pool Disable

| Feature | `tradingIsPaused` | `setPoolDisabled` |
|---|---|---|
| Scope | All pools with this token | Single pool |
| Requires `checkDisabledPools` | No | Yes |
| Affects swaps | Yes | Yes |
| Affects liquidity | Depends on hook flags | Depends on implementation |
| Reversible | Yes (set to false) | Yes (set disable=false) |
| Who can set | Token admin | Token contract, owner, or admin of either token in the pair |

## Token Fee vs Paired Token Fee

Fees are split across `beforeSwap` (charged on the "specified" side) and `afterSwap` (charged on the "unspecified" side). The fee is always a percentage of `swapParams.amount`, whose denomination varies by hook phase and swap direction.

| Field | Applied When | beforeSwap charges on | afterSwap charges on |
|---|---|---|---|
| `tokenFeeBuyBPS` | Token is being bought | output amount (output swap) | output amount (input swap) |
| `tokenFeeSellBPS` | Token is being sold | input amount (input swap) | input amount (output swap) |
| `pairedFeeBuyBPS` | Token is being bought | input amount (input swap) | input amount (output swap) |
| `pairedFeeSellBPS` | Token is being sold | output amount (output swap) | output amount (input swap) |

"Buy" = the hook's token is the output (user is acquiring it). "Sell" = the hook's token is the input (user is disposing of it).
