# TokenMaster Transaction Flows

## Buy Flow

### Basic Buy
```solidity
router.buyTokens{value: msg.value}(BuyOrder({
    tokenMasterToken: tokenAddress,
    tokensToBuy: amount,
    pairedValueIn: maxCost
}));
```
- For native token: include value as `msg.value`
- For ERC20: approve Router first, then call without value

### Advanced Buy
```solidity
router.buyTokensAdvanced{value: msg.value}(
    BuyOrder({
        tokenMasterToken: tokenAddress,
        tokensToBuy: amount,
        pairedValueIn: maxCost
    }),
    SignedOrder({
        creatorIdentifier: identifier,
        tokenMasterOracle: address(0),    // address(0) to skip oracle
        baseToken: tokenAddress,
        baseValue: value,
        maxPerWallet: maxWallet,
        maxTotal: maxTotal,
        expiration: deadline,
        hook: address(0),                 // address(0) for no hook
        signature: sig,
        cosignature: cosig,
        hookExtraData: "",
        oracleExtraData: ""
    }),
    PermitTransfer({
        permitProcessor: address(0),      // address(0) for direct approval
        nonce: 0,
        permitAmount: 0,
        expiration: 0,
        signedPermit: ""
    })
);
```
- Set `hook = address(0)` for basic signed order (no post-execution hook)
- Set `permitProcessor = address(0)` for direct approval (no PermitC)

---

## Sell Flow

### Basic Sell
```solidity
router.sellTokens(SellOrder({
    tokenMasterToken: tokenAddress,
    tokensToSell: amount,
    minimumOut: minReceived
}));
```

### Advanced Sell
```solidity
router.sellTokensAdvanced(
    SellOrder({
        tokenMasterToken: tokenAddress,
        tokensToSell: amount,
        minimumOut: minReceived
    }),
    SignedOrder({
        creatorIdentifier: identifier,
        tokenMasterOracle: address(0),
        baseToken: tokenAddress,
        baseValue: value,
        maxPerWallet: maxWallet,
        maxTotal: maxTotal,
        expiration: deadline,
        hook: hookAddress,               // always requires hook in signed order
        signature: sig,
        cosignature: cosig,
        hookExtraData: hookData,
        oracleExtraData: ""
    })
);
```

---

## Spend Flow

Spend always requires a signed order.

```solidity
router.spendTokens(
    SpendOrder({
        tokenMasterToken: tokenAddress,
        multiplier: multiplierCount,     // number of times to execute baseValue
        maxAmountToSpend: maxSpend       // slippage protection
    }),
    SignedOrder({
        creatorIdentifier: identifier,
        tokenMasterOracle: address(0),
        baseToken: tokenAddress,
        baseValue: value,
        maxPerWallet: maxWallet,
        maxTotal: maxTotal,
        expiration: deadline,
        hook: hookAddress,
        signature: sig,
        cosignature: cosig,
        hookExtraData: hookData,
        oracleExtraData: ""
    })
);
```

- `multiplier`: number of times to execute the signed order's baseValue
- `maxAmountToSpend`: slippage protection (total tokens that can be spent)

---

## Tracking Functions

### Buy Tracking
```solidity
(uint256 totalBought, uint256 totalWalletBought, bool orderDisabled, bool signatureValid, bool cosignatureValid) =
    router.getBuyTrackingData(token, signedOrder, buyer);
```

### Sell Tracking
```solidity
(uint256 totalSold, uint256 totalWalletSold, bool orderDisabled, bool signatureValid, bool cosignatureValid) =
    router.getSellTrackingData(token, signedOrder, seller);
```

### Spend Tracking
```solidity
(uint256 totalMultipliersSpent, uint256 totalWalletMultipliersSpent, bool orderDisabled, bool signatureValid, bool cosignatureValid) =
    router.getSpendTrackingData(token, signedOrder, spender);
```

---

## Disabling Orders

Only callable by owner, admin, or ORDER_MANAGER role.

```solidity
router.disableBuyOrder(token, signedOrder, true);    // true = disabled
router.disableSellOrder(token, signedOrder, true);
router.disableSpendOrder(token, signedOrder, true);
```

---

## Hook Behavior Differences

**CRITICAL: `sellTokensAdvanced` ALWAYS calls the hook.** There is no `address(0)` check. If `signedOrder.hook` is `address(0)`, the call will revert because it attempts `ITokenMasterSellHook(address(0)).tokenMasterSellHook(...)`.

This differs from buy and spend:

| Function | Hook Behavior |
|----------|--------------|
| `buyTokensAdvanced` | Optional -- checks `signedOrder.hook != address(0)` before calling (`executeHook` flag) |
| `sellTokensAdvanced` | **Always called** -- no `address(0)` check, will revert if hook is zero address |
| `spendTokens` | Optional -- checks `signedOrder.hook != address(0)` before calling |

Every advanced sell order MUST specify a valid hook contract address that implements `ITokenMasterSellHook`.
