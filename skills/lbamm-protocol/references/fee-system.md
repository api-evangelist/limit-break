# LBAMM Fee System

## Fee Surfaces

| Surface | Scope | Denomination | Bounds |
|---|---|---|---|
| Exchange fee | Execution-level | `tokenIn` | BPS <= 10,000 (input) / < 10,000 (output) |
| Fee-on-top | Execution-level | `tokenIn` | <= amountSpecified (input) / no cap (output) |
| LP fee | Hop-level | Hop's input token | Core-enforced caps; 55_555 = dynamic via hook |
| Hop fee | Per-token per hop | Hop's input/output token | Token-specific protocol fee (BPS in `TokenSettings.hopFeeBPS`) |
| Token hook fees (swap) | Per hop | Per token hook rules | Cannot overconsume trade |
| Token hook fees (liquidity) | Liquidity ops | token0/token1 | maxHookFee0/1 |
| Pool hook fees (liquidity) | Liquidity ops | token0/token1 | maxHookFee0/1 |
| Position hook fees (liquidity) | Liquidity ops | token0/token1 | maxHookFee0/1 |
| Protocol fees | Multiple | Depends on surface | Queryable fee rates |
| Flashloan fee | Flashloan | feeToken (may differ from loan token) | No explicit cap |

## Fee Structs

```solidity
// BPS-based exchange fee
struct BPSFeeWithRecipient {
    address recipient;
    uint256 BPS;
}

// Flat fee-on-top
struct FlatFeeWithRecipient {
    address recipient;
    uint256 amount;
}
```

## Swap Fee Ordering

### Input-Based Swap (`amountSpecified > 0`)
1. Fee-on-top removed first from input amount
2. Exchange fee computed on remainder (BPS of remaining input)
3. Protocol fees taken from both fee-on-top and exchange fee
4. Remaining amount goes to pool swap
5. Per-hop: LP fee applied within pool type swap math
6. Per-hop: Token hook `beforeSwap` fees (on input token) and `afterSwap` fees (on output token)

```
amountIn
  └─ feeOnTop (flat, subtracted first)
  └─ exchangeFee (BPS of remainder)
  └─ protocolFees (BPS of each fee above)
  └─ net amount → pool swap
       └─ LP fee (within pool math)
       └─ hook fees (beforeSwap on input, afterSwap on output)
```

### Output-Based Swap (`amountSpecified < 0`)
1. Pool math determines net input for desired output
2. Exchange fee scales upward: `fee = input * feeBPS / (MAX_BPS - feeBPS)`
3. Fee-on-top added last
4. Protocol fees taken from both

```
desiredOutput → pool swap → net amountIn
  └─ + exchangeFee (BPS scaled up)
  └─ + feeOnTop (flat, added last)
  └─ + protocolFees
  = total amountIn from user
```

## Fee Calculation Library (`FeeHelper.sol`)

### Input Swap Fee Calculation
```solidity
function calculateAmountAfterFeesSwapByInput(
    InternalSwapCache memory swapCache,
    BPSFeeWithRecipient calldata exchangeFee,
    FlatFeeWithRecipient calldata feeOnTop,
    ProtocolFeeStructure memory protocolFeeStructure
) internal pure;
```
- Subtracts `feeOnTop.amount` from `swapCache.amountIn`
- Calculates exchange fee: `amountIn * exchangeFeeBPS / MAX_BPS`
- Calculates protocol fees on both
- Updates `swapCache.amountIn` to post-fee amount

### Output Swap Fee Calculation
```solidity
function calculateAmountAfterFeesSwapByOutput(
    InternalSwapCache memory swapCache,
    FlatFeeWithRecipient calldata feeOnTop,
    BPSFeeWithRecipient calldata exchangeFee,
    ProtocolFeeStructure memory protocolFeeStructure
) internal pure;
```
- Calculates exchange fee: `amountIn * feeBPS / (MAX_BPS - feeBPS)` (rounded up)
- Adds `feeOnTop.amount` to `swapCache.amountIn`
- Calculates protocol fees on both

## Liquidity Operation Fee Ordering

### Hook Validation Order
1. Token0 hook → returns `(hookFee0, hookFee1)`
2. Token1 hook → returns `(hookFee0, hookFee1)`
3. Position hook → returns `(hookFee0, hookFee1)`
4. Pool hook → returns `(hookFee0, hookFee1)`

All hook fees are aggregated per token and compared against `maxHookFee0`/`maxHookFee1` supplied by the caller.

### Fee Netting
- **Add liquidity**: Hook fees increase the deposit amount (provider pays more)
- **Remove liquidity**: Hook fees reduce the withdrawal amount (provider receives less)
- **Collect fees**: Hook fees reduce the collected fee amount

## Settlement Model

1. **Calculation**: Fees computed during execution
2. **Accrual**: Hook fees accrued in core balances (not transferred immediately)
3. **Transfer**: Queued transfers execute FIFO after primary operation finalizes
4. **Collection**: Hook fees collectible later by the hook contract

All operations are atomic -- any revert causes the entire execution to revert.

## Multi-Hop Invariants

- Exchange fee: applied once (execution-level)
- Fee-on-top: applied once (execution-level)
- LP fees: per hop
- Token hook swap fees: per hop
- Execution-level fees do not compound across hops

## Protocol Fee Structure

```solidity
struct ProtocolFeeStructure {
    uint16 lpFeeBPS;        // Protocol fee on LP fees
    uint16 exchangeFeeBPS;  // Protocol fee on exchange fees
    uint16 feeOnTopBPS;     // Protocol fee on fee-on-top
}
```

Protocol fees are taken as a percentage of each fee surface and accumulated in `protocolFees[token]` mapping.

## Internal Fee Tracking Caches

### InternalSwapCache (fee-relevant fields)

Used during swap execution to track fee calculations across hops. Key fee fields:

```solidity
struct InternalSwapCache {
    // Execution-level fee tracking
    ProtocolFeeStructure protocolFeeStructure; // Active protocol fee config
    uint16 defaultProtocolLPFeeBPS;            // Default protocol LP fee if not overridden
    uint256 feeOnTopAmount;                    // Flat fee-on-top amount
    uint256 exchangeFeeAmount;                 // Exchange fee amount
    uint256 protocolExchangeFeeAmount;         // Protocol share of exchange fee
    uint256 protocolFeeFromFees;               // Protocol fee derived from other fees

    // Per-hop fee tracking
    uint256 expectedLPFee;                     // Expected pool fee on input swaps
    uint256 expectedProtocolLPFee;             // Expected protocol fee of pool fee
    uint256 protocolFee;                       // Protocol fee for this swap
    uint256 tokenInTokenInFee;                 // Input token hook fee (on input)
    uint256 tokenOutTokenInFee;                // Output token hook fee (on input)
    uint256 tokenInTokenOutFee;                // Input token hook fee (on output)
    uint256 tokenOutTokenOutFee;               // Output token hook fee (on output)
    // ... plus swap context, pool state, and memory pointers
}
```

### InternalLiquidityModificationCache

Used during liquidity operations to cache deposit/withdrawal amounts and fees:

```solidity
struct InternalLiquidityModificationCache {
    uint256 amount0;  // Amount to deposit or withdraw of token0
    uint256 amount1;  // Amount to deposit or withdraw of token1
    uint256 fees0;    // Fees for the position in token0
    uint256 fees1;    // Fees for the position in token1
}
```
