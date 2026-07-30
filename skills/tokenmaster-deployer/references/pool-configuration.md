# TokenMaster Pool Configuration

Pool-specific configuration after deployment.

## Standard Pool Settings

### Buy Parameters
```solidity
pool.setBuyParameters(StandardPoolBuyParameters({
    buySpreadBPS: 100,                          // uint16
    buyFeeBPS: 200,                             // uint16
    buyCostPairedTokenNumerator: 1e18,          // uint96
    buyCostPoolTokenDenominator: 1e18,          // uint96
    useTargetSupply: false,                     // bool
    reserved: 0,                                // uint24
    buyDemandFeeBPS: 0,                         // uint16
    targetSupplyBaseline: 0,                    // uint48
    targetSupplyBaselineScaleFactor: 0,         // uint8
    targetSupplyGrowthRatePerSecond: 0,         // uint96
    targetSupplyBaselineTimestamp: 0            // uint48
}));
```

### Sell Parameters
```solidity
pool.setSellParameters(StandardPoolSellParameters({
    sellSpreadBPS: 100,                         // uint16
    sellFeeBPS: 200                             // uint16
}));
```

### Spend Parameters
```solidity
pool.setSpendParameters(StandardPoolSpendParameters({
    creatorShareBPS: 5000                       // uint16
}));
```

### Emissions
```solidity
// Hard cap can only decrease
pool.setEmissionsHardCap(1_000_000e18);

// Claim emissions -- forfeited amount must re-accrue
pool.claimEmissions(claimToAddress, forfeitAmount);
```

### Additional Standard Pool Operations
- `transferCreatorShareToMarket` -- increases token price by moving creator share to market share
- Pause flags for buys/sells/spends individually via PAUSE_FLAG_BUYS, PAUSE_FLAG_SELLS, PAUSE_FLAG_SPENDS

### Guardrails (Immutable from Deployment)
```
minBuySpreadBPS
maxBuySpreadBPS
maxBuyFeeBPS
maxBuyDemandFeeBPS
minSellSpreadBPS
maxSellSpreadBPS
maxSellFeeBPS
maxSpendCreatorShareBPS
```
These cannot be changed after deployment. All parameter updates must stay within guardrail bounds.

---

## Stable Pool Settings

### Buy Parameters
```solidity
pool.setBuyParameters(StablePoolBuyParameters({
    buyFeeBPS: 200                              // uint16
}));
```

### Sell Parameters
```solidity
pool.setSellParameters(StablePoolSellParameters({
    sellFeeBPS: 200                             // uint16
}));
```

### Guardrails
```
maxBuyFeeBPS
maxSellFeeBPS
```

### Note
Price ratio (PairedPricePerToken numerator/denominator) is immutable from deployment.

---

## Promotional Pool Settings

### Buy Parameters
```solidity
pool.setBuyParameters(PromotionalPoolBuyParameters({
    buyCostPairedTokenNumerator: 1e18,          // uint96
    buyCostPoolTokenDenominator: 1e18           // uint96
}));
```
No guardrails on promotional pool buy parameters.

### Mint/Burn Roles
```solidity
// Grant minting capability
pool.grantRole(MINTER_ROLE, minterAddress);

// Grant burning capability
pool.grantRole(BURNER_ROLE, burnerAddress);
```

---

## Common Operations (All Pool Types)

### Withdraw Creator Share
```solidity
router.withdrawCreatorShare(token, withdrawTo, withdrawAmount);
```

### Ownership (Two-Step Transfer)
```solidity
// Step 1: Current owner initiates transfer
pool.transferOwnership(newOwner);

// Step 2: New owner accepts
pool.acceptOwnership();

// Renouncing ownership is PREVENTED
```

### Role Management
```solidity
pool.grantRole(role, account);
pool.revokeRole(role, account);
```
