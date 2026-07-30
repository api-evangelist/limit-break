# LBAMM Execution Model

## Core Concepts

### Execution
A single top-level call into LBAMM core (e.g., `singleSwap`, `multiSwap`, `addLiquidity`, `removeLiquidity`, `collectFees`, flashloan). It is atomic: if any required validation or hook invocation reverts, the entire execution reverts.

### Execution Context
Non-spoofable metadata constructed by core at execution start. Encodes:
- **Executor** (`msg.sender`): The LBAMM caller. No `tx.origin` concept exists.
- **Recipient**: Address receiving output assets (may differ from executor).
- **Hop index**: Zero-based position in multi-hop route.
- **Operation-specific metadata**: Tokens, amounts, fees.

The execution context cannot be modified by hooks.

### Executor
The LBAMM `msg.sender`. If an external router calls LBAMM, that router is the executor for hook-policy purposes. There is no `tx.origin` passthrough.

### Recipient
Address that receives output assets. Set by the caller in the swap order or liquidity params.

### Route / Hop / HopIndex
- **Route**: Ordered sequence of hops in `multiSwap`
- **Hop**: A single pool swap within a route
- **hopIndex**: Zero-based index of the current hop

### Single-Hop vs Multi-Hop
- `singleSwap`: Exactly one hop
- `multiSwap`: Multiple hops with per-hop LP + token hook fees; execution-level fees (exchange, fee-on-top) apply once

## Protocol-Level Guarantees

1. **Single execution context per top-level call** -- No nested execution contexts
2. **Atomicity** -- Any revert causes entire execution to revert
3. **Deterministic hook invocation surface** -- Hooks called in defined order based on flags
4. **Core-enforced settlement authority** -- Core owns reserves, manages all transfers
5. **Policy attachment only at defined scopes** -- Token/pool/position hooks only

## Explicit Non-Guarantees

- **Economic outcomes**: No profitability guarantees for LPs, swappers, or hooks
- **Hook safety**: Hooks are external; core guarantees invocation surfaces only
- **Cross-protocol enforcement**: LBAMM does not enforce external protocol rules
- **Absence of revert**: Hooks or pool types may revert
- **Backward compatibility of custom pool types**: Pool type upgrades are not guaranteed

## Trust Boundaries

| Component | Trusted For | Trusted By |
|---|---|---|
| Core AMM | Custody, accounting, context, enforcement | Everyone |
| Token hooks | Token-level policy | Anyone interacting with that token |
| Pool hooks | Pool-level policy | Pool creators and participants |
| Position hooks | Position-level policy | Position providers |
| Pool types | Liquidity math, swap math | Pool creators and LPs |

## Re-entrancy Protection

LBAMM uses guard flags for re-entrancy protection:

```solidity
// Base flags
SWAP_GUARD_FLAG                    = 1 << 2                           // Any swap
LIQUIDITY_GUARD_FLAG               = 1 << 7                           // Any liquidity op
FLASHLOAN_GUARD_FLAG               = 1 << 11                          // Flashloan

// Compound flags (OR with parent — checking SWAP_GUARD_FLAG catches all swap types)
POOL_SWAP_GUARD_FLAG               = 1 << 3 | SWAP_GUARD_FLAG         // Pool-based swap
SINGLE_POOL_SWAP_GUARD_FLAG        = 1 << 4 | POOL_SWAP_GUARD_FLAG    // Single swap
MULTI_POOL_SWAP_GUARD_FLAG         = 1 << 5 | POOL_SWAP_GUARD_FLAG    // Multi swap
DIRECT_SWAP_GUARD_FLAG             = 1 << 6 | SWAP_GUARD_FLAG         // Direct swap
ADD_LIQUIDITY_GUARD_FLAG           = 1 << 8 | LIQUIDITY_GUARD_FLAG    // Add liquidity
REMOVE_LIQUIDITY_GUARD_FLAG        = 1 << 9 | LIQUIDITY_GUARD_FLAG    // Remove liquidity
COLLECT_FEES_LIQUIDITY_GUARD_FLAG  = 1 << 10 | LIQUIDITY_GUARD_FLAG   // Collect fees
```

The guard system is hierarchical: setting `SINGLE_POOL_SWAP_GUARD_FLAG` (= `1<<4 | 1<<3 | 1<<2`) also sets `POOL_SWAP_GUARD_FLAG` and `SWAP_GUARD_FLAG` bits. This means checking `_isReentrancyFlagSet(SWAP_GUARD_FLAG)` will match any swap-related operation.

## Practical Guidance

### For Router/Aggregator Builders
- You are the executor; hooks see you as `msg.sender`
- Construct `SwapOrder` and fee structs carefully
- Handle token approvals before calling LBAMM
- The AMM enforces `deadline` and `limitAmount`

### For Token Integrators
- Read token hook configuration via `getTokenSettings(token)` first
- Check if hooks impose fees or restrictions
- Respect `maxHookFee0`/`maxHookFee1` in liquidity operations

### For Hook Authors
- Document revert conditions, fee behavior, and external dependencies
- Keep work bounded and deterministic
- Hooks MAY call the AMM during execution to collect fees (queued transfers)
- Hooks MUST revert to prevent an operation (returning 0 fees still allows it)
- Hook fees are accrued in core, not transferred immediately

## Direct Swap

`directSwap` is a peer-to-peer exchange at a fixed rate without AMM pool liquidity:
- The executor provides output tokens directly to the recipient
- The executor receives input tokens at the predetermined rate
- No pool state is affected
- Pool hooks are NOT supported (only token hooks run)
- Used with transfer handlers for permit-based or CLOB-based settlement
