---
name: apptoken-designer
description: Design an apptoken -- map high-level requirements to protocols, configuration steps, and implementation skills
user-invocable: true
argument-hint: "[description of your token - what it does, who uses it, how it should behave]"
---

# Apptoken Designer

Take a high-level token idea and produce a concrete implementation plan: which protocols to use, what to configure, and which skills to invoke.

## Instructions

1. **Parse the user's vision** from `$ARGUMENTS` -- understand the token's purpose, target users, economic model, and behavioral constraints.

2. **Clarify critical decisions early** -- before building the plan, check if $ARGUMENTS leaves any of these ambiguous: reserve-backed (TokenMaster) vs unbacked, permissioned vs permissionless, target chain. If the token's core economic model is unclear, ask in a single message. If the vision is clear enough to produce a useful plan, proceed -- step 5 will flag remaining decisions for the user. Note: when the user asks for a TokenMaster token, the primary buy/sell mechanism is the TokenMaster Router -- don't ask about LBAMM/DEX trading unless the user specifically wants it.

3. **Classify into apptoken pattern categories** (a token can combine multiple):

   | Category | Signals | Key Protocols |
   |----------|---------|---------------|
   | **Client-Side** | User acquisition, quests, 2FA, offerwalls | Trusted Forwarder (permissioned mode) |
   | **Trade** | Trading fees, exclusive pairs, price ranges, fixed ratios, asset gating, lotteries | LBAMM token hooks, standard hook, Transfer Validator |
   | **Temporal** | Vesting, expiries, time windows | LBAMM token hooks, standard hook |
   | **Spend-Only** | Promotional tokens, game resources, utility tokens, no trading | Transfer Validator (restrictive policy) |
   | **Compliant** | KYC, multi-jurisdiction | Transfer Validator, Trusted Forwarder |
   | **Backed** | Price floor, reserve backing, monetized spends | TokenMaster pools (buy/sell via Router), optionally LBAMM for secondary DEX trading |

4. **Map to a protocol stack** -- determine which protocols are needed and in what order:

   - **All apptokens** need: ERC-20C token contract + Transfer Validator configuration
   - **Backed tokens** add: TokenMaster pool deployment (Standard, Stable, or Promotional) -- tokens are bought/sold through the TokenMaster Router directly
   - **DEX-tradeable tokens** add: LBAMM pool creation + token hook (or standard hook) configuration -- this is optional for backed tokens that want additional DEX liquidity
   - **Gated tokens** add: Trusted Forwarder deployment in permissioned mode
   - **NFT collections** use: ERC-721C/ERC-1155C + Payment Processor configuration instead of LBAMM/TokenMaster

5. **Produce an implementation plan** with these sections:

   **a. Token Contract** -- Which token standard to use, any custom overrides needed

   **b. Transfer Validator Setup** -- Ruleset selection, list configuration, operator whitelisting

   **c. Protocol Configuration** -- Per-protocol setup steps (LBAMM pools/hooks, TokenMaster deployment, Payment Processor settings)

   **d. Skill Sequence** -- Ordered list of skills to invoke, with example arguments:
   ```
   1. /transfer-validator-config [whitelist TokenMaster as operator]
   2. /tokenmaster-deployer [Standard pool with USDC pairing, 5% creator fee]
   3. /tokenmaster-integrator [buy/sell integration via Router]
   ```

   **e. Architecture Diagram** -- ASCII diagram showing protocol interactions

6. **Flag important decisions** the user needs to make:
   - Reserve-backed (TokenMaster) vs unbacked (LBAMM-only)
   - Additional DEX trading (LBAMM) on top of TokenMaster Router -- optional, not default
   - Permissioned (Trusted Forwarder) vs permissionless
   - Custom hook logic vs standard hook configuration (only relevant if using LBAMM)

## Reference Files

- Protocol knowledge: `apptoken-general` (ecosystem overview, token standards, pattern categories)

## Protocol Selection Guide

| User Wants | Protocols Needed | Primary Skills |
|------------|-----------------|----------------|
| Simple tradeable token | ERC-20C, TV, LBAMM | `/transfer-validator-config`, `/lbamm-standard-hook`, `/lbamm-pool` |
| Token with price floor | ERC-20C, TV, TokenMaster | `/tokenmaster-deployer`, `/transfer-validator-config` |
| Backed token + DEX float | ERC-20C, TV, TokenMaster, LBAMM | Bootstrap via TM buys → disable TM buys → add LBAMM liquidity (TM sells stay as floor). `/tokenmaster-deployer`, `/transfer-validator-config`, `/lbamm-standard-hook`, `/lbamm-pool` |
| Game currency (no trading) | ERC-20C, TV | `/transfer-validator-config` (restrictive whitelist) |
| NFT collection with royalties | ERC-721C, TV, PP | `/transfer-validator-config`, `/payment-processor-creator` |
| NFT marketplace (buy/sell/trade) | ERC-721C, TV, PP | `/transfer-validator-config`, `/payment-processor-creator`, `/payment-processor-exchange` |
| Compliant token with KYC | ERC-20C, TV, Trusted Forwarder | `/transfer-validator-config`, custom forwarder setup |
| Token with custom swap logic | ERC-20C, TV, LBAMM + custom hook | `/lbamm-token-hook`, `/lbamm-pool`, `/transfer-validator-config` |
| Fixed-rate stablecoin swap | ERC-20C, TV, LBAMM (Fixed pool) | `/lbamm-pool` (Fixed type), `/transfer-validator-config` |
| Gasless swap execution | ERC-20C, TV, LBAMM + PermitC | `/lbamm-permit-handler`, `/lbamm-pool`, `/transfer-validator-config` |
| On-chain limit order book | ERC-20C, TV, LBAMM CLOB | `/lbamm-clob`, `/lbamm-pool`, `/transfer-validator-config` |

## Key Architecture Principle

Transfer Validator is the foundation layer. Every other protocol (LBAMM, TokenMaster, Payment Processor) must be whitelisted as an operator in the Transfer Validator before it can move tokens. This gives creators a single kill switch over their entire token economy.

## Note on Trusted Forwarder

The Trusted Forwarder provides web2 app routing and optional permissioning (Client-Side and Compliant token categories). There is no dedicated skill for Trusted Forwarder deployment -- guide the user through manual setup using the Trusted Forwarder Factory at `0xFF0000B6c4352714cCe809000d0cd30A0E0c8DcE`. See `apptoken-general/references/ecosystem.md` for details.
