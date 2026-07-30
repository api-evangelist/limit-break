# Apptoken Categories & Patterns

## Overview

Apptokens are ERC-20C tokens whose behavior is shaped by the protocols in the Apptoken ecosystem. The Apptoken Cookbook defines 6 categories of patterns. A single token can combine patterns from multiple categories to create complex, application-specific behavior.

Each pattern is implemented through one or more of these mechanisms:
- **Transfer Validator policies** -- operator whitelists/blacklists, modular rulesets, receiver constraints
- **LBAMM token hooks** -- per-token logic that fires on every swap (before/after)
- **LBAMM pool hooks** -- per-pool logic for liquidity and fee customization
- **LBAMM standard hook configs** -- declarative configuration flags on the standard hook contract
- **TokenMaster pool settings** -- mint/burn pool parameters, reserve ratios, spend mechanics
- **Trusted Forwarder gating** -- EIP-712 signature requirements, signer-controlled access

---

## 1. Client-Side Patterns

**Purpose**: User acquisition, engagement mechanics, and security features that are primarily driven by off-chain (web2) application logic, with on-chain enforcement via Trusted Forwarder gating.

### User Acquisition
- **What**: Token rewards for new user signups, referrals, social sharing
- **Protocols**: Trusted Forwarder (permissioned mode), LBAMM or TokenMaster for token distribution
- **Mechanism**: Backend validates acquisition event, signs EIP-712 message, user claims tokens through permissioned forwarder

### Quest / Task Gating
- **What**: Tokens earned by completing in-app tasks, achievements, or engagement milestones
- **Protocols**: Trusted Forwarder (permissioned mode)
- **Mechanism**: Backend tracks task completion, authorizes token claim via signed forwarder message. On-chain, the forwarder only relays if the signature is valid -- preventing unauthorized claims.

### Security (2FA)
- **What**: Require two-factor authentication before high-value token operations
- **Protocols**: Trusted Forwarder (permissioned mode)
- **Mechanism**: Backend requires 2FA verification before signing the forwarder message. Without the signed message, the on-chain operation cannot proceed.

### Offerwalls
- **What**: Earn tokens by completing advertiser offers (surveys, app installs, etc.)
- **Protocols**: Trusted Forwarder (permissioned mode), TokenMaster or LBAMM for distribution
- **Mechanism**: Backend validates offer completion with ad network, signs claim authorization

---

## 2. Trade Patterns

**Purpose**: Control how and when tokens can be traded, who can trade them, and what fees apply. These are the most diverse patterns, covering everything from simple trading fees to complex conditional trading rules.

### Exclusive Pairs
- **What**: Token can only be traded against specific counter-assets (e.g., only WETH, not USDC)
- **Protocols**: Transfer Validator (whitelist only LBAMM), LBAMM (pool creation restricted)
- **Mechanism**: Token hook's `validatePoolCreation` validates the paired token. Only approved pairs can have pools created.

### Price Ranges
- **What**: Trading restricted to a price band (e.g., token cannot trade below $0.50 or above $2.00)
- **Protocols**: LBAMM token hooks
- **Mechanism**: Token hook's `beforeSwap` checks the resulting price against configured bounds, reverting if the swap would push the price outside the range.

### Fixed Ratios
- **What**: Token trades at a fixed exchange rate (e.g., always 1:1 with USDC)
- **Protocols**: LBAMM (Fixed pool type)
- **Mechanism**: Fixed pool type enforces a constant exchange rate. Combined with exclusive pair restriction to prevent other pool types.

### Pay-to-Trade (Trading Fees)
- **What**: Creator earns a fee on every trade
- **Protocols**: LBAMM token hooks, LBAMM standard hook configs
- **Mechanism**: Token hook's `beforeSwap` or `afterSwap` collects a fee (in BPS) from the trade. Standard hook provides declarative fee configuration. Fees accrue to the creator's designated address.

### Tax
- **What**: Transfer tax applied on every token movement (not just trades)
- **Protocols**: Transfer Validator (custom validation logic), LBAMM token hooks
- **Mechanism**: On LBAMM swaps, token hook deducts a percentage. For direct transfers, custom logic in the token's `_beforeTokenTransfer` can enforce tax (though this requires careful implementation to avoid breaking protocol interactions).

### Asset Ownership Requirements
- **What**: Must hold a specific NFT or token to trade
- **Protocols**: LBAMM token hooks, Trusted Forwarder (permissioned mode)
- **Mechanism**: Token hook's `beforeSwap` checks the swapper's balance of the required asset. Alternatively, Trusted Forwarder's backend validates ownership before signing.

### Lotteries
- **What**: Random chance of bonus tokens on trades, or lottery-style distribution
- **Protocols**: LBAMM token hooks
- **Mechanism**: Token hook's `afterSwap` uses on-chain randomness (or VRF) to determine if the trader receives bonus tokens from a reward pool.

---

## 3. Temporal Patterns

**Purpose**: Time-based restrictions and schedules for token operations.

### Vesting
- **What**: Tokens unlock gradually over time (linear, cliff, or milestone-based)
- **Protocols**: LBAMM token hooks, Trusted Forwarder (permissioned mode)
- **Mechanism**: Token hook's `beforeSwap` checks if the sender's tokens are vested. Vesting schedules can be stored in the hook contract or validated off-chain via the forwarder's signer.

### Expiries
- **What**: Tokens become non-transferable or non-tradeable after a deadline
- **Protocols**: LBAMM token hooks, Transfer Validator
- **Mechanism**: Token hook's `beforeSwap` checks `block.timestamp` against expiry. After expiry, swaps revert. Transfer Validator policies can also be updated post-expiry to freeze all transfers.

### Event-Based Time Windows
- **What**: Trading only allowed during specific windows (e.g., game seasons, promotional periods)
- **Protocols**: LBAMM standard hook configs, LBAMM token hooks
- **Mechanism**: Standard hook supports declarative start/end timestamps for trading windows. Custom token hooks can implement more complex schedules (recurring windows, dynamic windows based on external events).

---

## 4. Spend-Only Patterns

**Purpose**: Tokens designed to be consumed (burned) rather than traded, creating pure utility or promotional economies.

### Promotional Tokens
- **What**: Tokens given away for free that can only be spent in-app, not traded or transferred
- **Protocols**: Transfer Validator (restrictive policy), Trusted Forwarder (permissioned mode)
- **Mechanism**: Transfer Validator configured to block all transfers except to the burn address or designated spend contract. Forwarder gates spend operations to only the issuing app.

### Pure Game Resources
- **What**: In-game currency or resources (gold, wood, energy) that exist on-chain but are not tradeable
- **Protocols**: Transfer Validator (restrictive policy), LBAMM token hooks (if limited trading desired)
- **Mechanism**: Transfer Validator whitelist contains only the game contracts. No DEX or marketplace operators are whitelisted. Tokens can be earned and spent within the game but cannot leave the game's contract ecosystem.

### Pure Utility Tokens
- **What**: Tokens that represent access rights, votes, or service credits -- consumed on use
- **Protocols**: Transfer Validator, Trusted Forwarder (permissioned mode)
- **Mechanism**: Similar to promotional tokens but with specific spend contracts that provide utility (access a service, cast a vote, unlock content). Transfer restrictions prevent secondary market trading.

---

## 5. Compliant Patterns

**Purpose**: Regulatory compliance features for tokens that must adhere to legal requirements.

### KYC (Know Your Customer)
- **What**: Only KYC-verified users can hold, trade, or receive the token
- **Protocols**: Transfer Validator (receiver constraints), Trusted Forwarder (permissioned mode), EOA Registry
- **Mechanism**: Transfer Validator configured with receiver allowlist -- only addresses that have passed KYC are added. Trusted Forwarder's backend validates KYC status before signing transaction authorizations. EOA Registry ensures receivers are not smart contracts.

### Multi-Jurisdiction Compliance
- **What**: Different rules for different jurisdictions (e.g., US users cannot trade, EU users have different limits)
- **Protocols**: Trusted Forwarder (permissioned mode), LBAMM token hooks
- **Mechanism**: Forwarder's backend checks user jurisdiction and applies appropriate rules before signing. Token hooks can enforce on-chain limits per jurisdiction if jurisdiction data is available on-chain (e.g., via oracle or registry).

---

## 6. Backed Patterns

**Purpose**: Tokens with guaranteed minimum value through TokenMaster reserve backing, enabling monetizable spend economies.

### Price-Stable Tokens
- **What**: Tokens maintain a floor price through reserve backing in TokenMaster pools
- **Protocols**: TokenMaster (mint/burn pools via Router)
- **Mechanism**: TokenMaster pool holds reserve assets (e.g., WETH, USDC). Users buy and sell tokens directly through the TokenMaster Router. Tokens can always be burned to redeem the reserve at the pool's rate, establishing a price floor. Optionally, LBAMM can be added as a secondary trading layer for DEX-style price discovery above the floor.

### Monetized Spends
- **What**: When users spend (burn) tokens in-app, the creator captures revenue from the released reserves
- **Protocols**: TokenMaster (spend mechanics), Trusted Forwarder (gated spend)
- **Mechanism**: User spends tokens via a Trusted Forwarder-gated burn. TokenMaster releases the reserve assets, splitting them between the creator (revenue) and optionally the user (partial refund). This creates a sustainable revenue model: users buy tokens (funding reserves), use tokens in-app, and creators earn from the spend.

### Combined Backed + Trade Patterns (Advanced)
- **What**: Backed tokens that float on a DEX with TokenMaster providing a price floor
- **Protocols**: TokenMaster + LBAMM + Transfer Validator
- **Mechanism**: Bootstrap supply via TokenMaster buys, then disable buys and add DEX liquidity on LBAMM. TokenMaster sells stay enabled as a floor — if the DEX price dips below the TokenMaster sell price, holders can burn tokens to redeem reserves, establishing a hard floor. Buys must be disabled on TokenMaster to prevent arbitrage: without this, bots can buy cheaply from TokenMaster and sell into the DEX, draining the reserves.
- **Lifecycle**: (1) Deploy TokenMaster pool with buys enabled → (2) Bootstrap initial token supply via TokenMaster buys → (3) Disable TokenMaster buys → (4) Add LBAMM pool with DEX liquidity → (5) Token now floats on LBAMM with TokenMaster sell-only floor
- Transfer Validator enforces that trading only happens through authorized protocols. This is an advanced configuration -- most backed tokens only need TokenMaster.

---

## Pattern Composition

Apptokens commonly combine patterns from multiple categories. Examples:

| Apptoken Type | Categories Combined | Key Protocols |
|---------------|-------------------|---------------|
| Game currency with real value | Backed + Spend-Only + Trade (pay-to-trade) | TokenMaster, LBAMM, Transfer Validator |
| Compliant loyalty token | Compliant (KYC) + Client-Side (quests) + Temporal (expiry) | Trusted Forwarder, Transfer Validator, LBAMM hooks |
| Premium membership token | Backed + Trade (exclusive pairs) + Client-Side (acquisition) | TokenMaster, LBAMM, Trusted Forwarder |
| Promotional airdrop | Spend-Only (promotional) + Temporal (expiry) | Transfer Validator, Trusted Forwarder |
| Regulated security token | Compliant (multi-jurisdiction) + Temporal (vesting) + Trade (price ranges) | Trusted Forwarder, LBAMM hooks, Transfer Validator |

The modular protocol architecture means new patterns can be created by combining existing protocol capabilities without deploying new infrastructure -- only configuration and hook logic change.
