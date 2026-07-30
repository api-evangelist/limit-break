# Token Hook Patterns

## Pattern 1: Fee Collection Hook (Buy/Sell Fees)

Charges different fees on buys vs sells. Common for token issuers wanting to tax trading.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {ILimitBreakAMMTokenHook} from "@limitbreak/lbamm-core/src/interfaces/hooks/ILimitBreakAMMTokenHook.sol";
import {SwapContext, HookSwapParams, PoolCreationDetails, LiquidityContext, LiquidityModificationParams, LiquidityCollectFeesParams} from "@limitbreak/lbamm-core/src/DataTypes.sol";

contract FeeCollectionHook is ILimitBreakAMMTokenHook {
    address public immutable AMM;         // The LBAMM contract
    address public immutable TOKEN;       // The token this hook is for
    uint256 public buyFeeBPS;             // Fee when token is being bought (tokenOut)
    uint256 public sellFeeBPS;            // Fee when token is being sold (tokenIn)
    address public feeRecipient;
    address public owner;

    modifier onlyAMM() { require(msg.sender == AMM, "Only AMM"); _; }

    constructor(address amm_, address token_, uint256 buyFeeBPS_, uint256 sellFeeBPS_, address feeRecipient_) {
        AMM = amm_;
        TOKEN = token_;
        buyFeeBPS = buyFeeBPS_;
        sellFeeBPS = sellFeeBPS_;
        feeRecipient = feeRecipient_;
        owner = msg.sender;
    }

    function hookFlags() external pure returns (uint32 requiredFlags, uint32 supportedFlags) {
        // Require beforeSwap and afterSwap to implement buy/sell fees
        requiredFlags = 1 << 0 | 1 << 1;  // beforeSwap + afterSwap
        supportedFlags = requiredFlags;
    }

    function beforeSwap(
        SwapContext calldata,
        HookSwapParams calldata swapParams,
        bytes calldata
    ) external onlyAMM returns (uint256 fee) {
        // beforeSwap fee is on input token (input swap) or output token (output swap)
        if (swapParams.inputSwap) {
            // Input swap: beforeSwap fee is on tokenIn
            // If our token is tokenIn, this is a SELL
            if (swapParams.hookForInputToken) {
                fee = (swapParams.amount * sellFeeBPS) / 10_000;
            }
        } else {
            // Output swap: beforeSwap fee is on tokenOut
            // If our token is tokenOut (hookForInputToken=false), this is a BUY
            if (!swapParams.hookForInputToken) {
                fee = (swapParams.amount * buyFeeBPS) / 10_000;
            }
        }
    }

    function afterSwap(
        SwapContext calldata,
        HookSwapParams calldata swapParams,
        bytes calldata
    ) external onlyAMM returns (uint256 fee) {
        // afterSwap fee is on output token (input swap) or input token (output swap)
        if (swapParams.inputSwap) {
            // Input swap: afterSwap fee is on tokenOut
            // If our token is tokenOut (hookForInputToken=false), this is a BUY
            if (!swapParams.hookForInputToken) {
                fee = (swapParams.amount * buyFeeBPS) / 10_000;
            }
        } else {
            // Output swap: afterSwap fee is on tokenIn
            // If our token is tokenIn, this is a SELL
            if (swapParams.hookForInputToken) {
                fee = (swapParams.amount * sellFeeBPS) / 10_000;
            }
        }
    }

    // Unused callbacks -- minimal implementations
    function validatePoolCreation(bytes32, address, bool, PoolCreationDetails calldata, bytes calldata) external {}
    function validateHandlerOrder(address, bool, address, address, uint256, uint256, bytes calldata, bytes calldata) external {}
    function validateCollectFees(bool, LiquidityContext calldata, LiquidityCollectFeesParams calldata, uint256, uint256, bytes calldata) external pure returns (uint256, uint256) { return (0, 0); }
    function validateAddLiquidity(bool, LiquidityContext calldata, LiquidityModificationParams calldata, uint256, uint256, uint256, uint256, bytes calldata) external pure returns (uint256, uint256) { return (0, 0); }
    function validateRemoveLiquidity(bool, LiquidityContext calldata, LiquidityModificationParams calldata, uint256, uint256, uint256, uint256, bytes calldata) external pure returns (uint256, uint256) { return (0, 0); }
    function beforeFlashloan(address, address, uint256, address, bytes calldata) external pure returns (address, uint256) { return (address(0), 0); }
    function validateFlashloanFee(address, address, uint256, address, uint256, address, bytes calldata) external pure returns (bool) { return false; }
    function tokenHookManifestUri() external pure returns (string memory) { return ""; }
}
```

## Pattern 2: Swap Gating Hook (Whitelist/Blacklist)

Gates swap access based on executor identity.

```solidity
contract SwapGatingHook is ILimitBreakAMMTokenHook {
    mapping(address => bool) public whitelisted;
    address public owner;

    function hookFlags() external pure returns (uint32 requiredFlags, uint32 supportedFlags) {
        requiredFlags = 1 << 0;  // beforeSwap only
        supportedFlags = requiredFlags;
    }

    function beforeSwap(
        SwapContext calldata context,
        HookSwapParams calldata,
        bytes calldata
    ) external returns (uint256 fee) {
        require(whitelisted[context.executor], "Not whitelisted");
        return 0;
    }

    // ... other callbacks as no-ops
}
```

## Pattern 3: Liquidity Whitelist Hook

Restricts who can provide liquidity for this token.

```solidity
contract LPWhitelistHook is ILimitBreakAMMTokenHook {
    mapping(address => bool) public approvedLPs;

    function hookFlags() external pure returns (uint32 requiredFlags, uint32 supportedFlags) {
        requiredFlags = 1 << 2 | 1 << 3;  // addLiquidity + removeLiquidity
        supportedFlags = requiredFlags | 1 << 5;  // also support poolCreation
    }

    function validateAddLiquidity(
        bool,
        LiquidityContext calldata context,
        LiquidityModificationParams calldata,
        uint256, uint256, uint256, uint256,
        bytes calldata
    ) external returns (uint256, uint256) {
        require(approvedLPs[context.provider], "LP not approved");
        return (0, 0);
    }

    function validateRemoveLiquidity(
        bool,
        LiquidityContext calldata context,
        LiquidityModificationParams calldata,
        uint256, uint256, uint256, uint256,
        bytes calldata
    ) external returns (uint256, uint256) {
        require(approvedLPs[context.provider], "LP not approved");
        return (0, 0);
    }
    // ... other callbacks as no-ops
}
```

## Pattern 4: Flashloan Fee Hook

Enables flashloans with custom fee logic.

```solidity
contract FlashloanFeeHook is ILimitBreakAMMTokenHook {
    uint256 public flashloanFeeBPS = 30; // 0.3%

    function hookFlags() external pure returns (uint32 requiredFlags, uint32 supportedFlags) {
        requiredFlags = 1 << 7;  // flashloans
        supportedFlags = requiredFlags | (1 << 8);  // also support flashloan fee validation
    }

    function beforeFlashloan(
        address,
        address loanToken,
        uint256 loanAmount,
        address,
        bytes calldata
    ) external returns (address feeToken, uint256 fee) {
        // Charge fee in the same token as the loan
        feeToken = loanToken;
        fee = (loanAmount * flashloanFeeBPS) / 10_000;
    }

    function validateFlashloanFee(
        address, address, uint256, address, uint256, address, bytes calldata
    ) external pure returns (bool) {
        return true; // Allow this token as fee token for other flashloans
    }
    // ... other callbacks as no-ops
}
```

## Hook Flag Combinations by Use Case

| Use Case | Flags (bits) | Notes |
|---|---|---|
| Buy/sell fees | 0, 1 | beforeSwap + afterSwap |
| Swap gating only | 0 | beforeSwap (revert to block) |
| LP restrictions | 2, 3 | addLiquidity + removeLiquidity |
| Pool creation control | 5 | poolCreation |
| Full fee management | 0, 1, 2, 3, 4, 6 | All fee callbacks + manages own fees |
| Flashloan support | 7 | beforeFlashloan |
| Handler order gating | 9 | handlerOrderValidate |
| Kitchen sink | 0-9 | All flags enabled |

## Security Considerations

1. **Reentrancy**: Hooks are called during AMM execution. Be cautious about external calls within hooks.
2. **Fee bounds**: Hook fees on liquidity ops are capped by `maxHookFee0`/`maxHookFee1` -- if your fee exceeds the cap, the operation reverts.
3. **Gas limits**: Keep hook logic bounded. Unbounded loops or heavy computation can make tokens unusable.
4. **Access control**: Only the LBAMM core should call hook callbacks. Validate `msg.sender == amm` in all callbacks (see `onlyAMM` modifier in Pattern 1).
5. **Return values**: Always return correct values. Returning excessive fees in swap hooks can cause the swap to revert.
6. **hookForToken0/hookForInputToken**: Always check these booleans to know which side of the operation you're validating.
