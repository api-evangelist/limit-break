# TokenMaster Pool Types

## Standard Pool

### Pricing
Market value = marketPairedTokenShare / totalSupply() (bonding curve).

### Buy
Market value + buySpread + buyFee + demandFee (if supply > target). Spread > 0 increases market value.

### Sell
Market value - sellSpread - sellFee. Spread > 0 increases market value.

### Spend
No payout to spender. Value to creator share at creatorShareBPS rate. Remainder stays in market (increases value).

### Emissions
Time-based minting at CREATOR_EMISSION_RATE_NUMERATOR / DENOMINATOR per second. Hard cap only decreases.

### Settings Structs

**StandardPoolInitializationParameters:** All guardrails + initial params.

**StandardPoolBuyParameters:**
```
buySpreadBPS                        uint16
buyFeeBPS                           uint16
buyCostPairedTokenNumerator         uint96
buyCostPoolTokenDenominator         uint96
useTargetSupply                     bool
reserved                            uint24
buyDemandFeeBPS                     uint16
targetSupplyBaseline                uint48
targetSupplyBaselineScaleFactor     uint8
targetSupplyGrowthRatePerSecond     uint96
targetSupplyBaselineTimestamp        uint48
```

**StandardPoolSellParameters:**
```
sellSpreadBPS                       uint16
sellFeeBPS                          uint16
```

**StandardPoolSpendParameters:**
```
creatorShareBPS                     uint16
```

### Target Supply Formula
```
baseline = targetSupplyBaseline * 10^scaleFactor
expected = baseline + growthRate * (timestamp - baselineTimestamp)
```

### Guardrails (Immutable at Deployment)
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

### Functions
- `setBuyParameters(StandardPoolBuyParameters)`
- `setSellParameters(StandardPoolSellParameters)`
- `setSpendParameters(StandardPoolSpendParameters)`
- `setEmissionsHardCap(uint256)` -- can only decrease
- `claimEmissions(address claimTo, uint256 forfeitAmount)` -- forfeited must re-accrue
- `getBuyParameters()` -> StandardPoolBuyParameters
- `getSellParameters()` -> StandardPoolSellParameters
- `getSpendParameters()` -> StandardPoolSpendParameters
- `targetSupply()` -> (bool useTargetSupply, uint256 target)
- `getParameterGuardrails()` -> (uint16 minBuySpreadBPS, uint16 maxBuySpreadBPS, uint16 maxBuyFeeBPS, uint16 maxBuyDemandFeeBPS, uint16 minSellSpreadBPS, uint16 maxSellSpreadBPS, uint16 maxSellFeeBPS, uint16 maxSpendCreatorShareBPS)
- `getCreatorEmissions()` -> (uint256 claimed, uint256 claimable, uint256 hardCap, uint48 lastClaim, uint128 creatorEmissionRateNumerator, uint128 creatorEmissionRateDenominator)

### Pausable
Buys, sells, spends individually pausable via PAUSE_FLAG_BUYS, PAUSE_FLAG_SELLS, PAUSE_FLAG_SPENDS.

---

## Stable Pool

### Pricing
Fixed ratio PairedPricePerToken (numerator:uint96, denominator:uint96), immutable.

### Buy
stablePrice + buyFee.

### Sell
stablePrice - sellFee.

### Spend
Burns tokens, no market value adjustment.

### Settings Structs

**StablePoolInitializationParameters:** Initialization parameters including immutable price ratio.

**StablePoolBuyParameters:**
```
buyFeeBPS                           uint16
```

**StablePoolSellParameters:**
```
sellFeeBPS                          uint16
```

### Guardrails
```
maxBuyFeeBPS
maxSellFeeBPS
```

### Functions
- `setBuyParameters(StablePoolBuyParameters)`
- `setSellParameters(StablePoolSellParameters)`
- `getBuyParameters()` -> StablePoolBuyParameters
- `getSellParameters()` -> StablePoolSellParameters
- `getStablePriceRatio()` -> (numerator, denominator)
- `getParameterGuardrails()` -> (uint16 maxBuyFeeBPS, uint16 maxSellFeeBPS)

### Limitations
No emissions. No pausable. No transfer-to-market -- StablePool does not override `transferCreatorShareToMarket` from BondedPool, which reverts with `TokenMasterERC20__OperationNotSupportedByPool()`. Only StandardPool provides a working implementation.

---

## Promotional Pool

### Pricing
No market value. Flat buy cost (buyCostPairedTokenNumerator:uint96, buyCostPoolTokenDenominator:uint96).

### Buy
Flat cost. All value goes to creator share.

### Sell
NOT SUPPORTED (reverts).

### Spend
Burns tokens.

### Direct Mint/Burn
Via roles: MINTER_ROLE, BURNER_ROLE.

### Functions
- `setBuyParameters(PromotionalPoolBuyParameters)`
- `getBuyParameters()` -> PromotionalPoolBuyParameters
- `mint(address to, uint256 amount)`
- `mintBatch(address[] toAddresses, uint256[] amounts)`
- `burn(address from, uint256 amount)`

---

## Common Pool Functions (All Types)

- `PAIRED_TOKEN()` -> address
- `pairedTokenShares()` -> (marketShare, creatorShare, infrastructureShare, partnerShare)
- `withdrawUnrelatedToken(address token, address to, uint256 amount)`
- Ownership: two-step transfer (transferOwnership + acceptOwnership). Renouncing PREVENTED.

## withdrawFees Authorization

`Router.withdrawFees(ITokenMasterERC20C[] tokenMasterTokens)` can be called by:

1. **`TOKENMASTER_FEE_COLLECTOR_ROLE` holder** -- authorized to withdraw fees from all tokens in a single call.
2. **`partnerFeeRecipient` for a specific token** -- authorized only for tokens where `msg.sender` matches `tokenSettings[token].partnerFeeRecipient`. If the caller is not the fee collector role holder, the function checks each token individually and reverts on any token where the caller is not the partner fee recipient.

In both cases, infrastructure fees go to the `TOKENMASTER_FEE_RECEIVER_ROLE` holder and partner fees go to the token's `partnerFeeRecipient`.
