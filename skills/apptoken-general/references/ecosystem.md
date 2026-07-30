# Cross-Protocol Integration Model

## Overview

The Apptoken ecosystem is a layered architecture where each protocol serves a distinct role. Tokens flow through these layers, with each layer enforcing its own rules. The key insight is that **Transfer Validator is the foundation** -- every other protocol must be whitelisted as an operator to move tokens, giving creators a single point of control over their entire token economy.

## Architecture Layers

```
+-------------------------------------------------------------+
|  Applications (web2 / web3 frontends)                       |
+-------------------------------------------------------------+
        |
+-------------------------------------------------------------+
|  Trusted Forwarder (attribution & gating layer)             |
+-------------------------------------------------------------+
        |                    |                    |
+------------------+ +------------------+ +------------------+
| Payment Processor| | LBAMM            | | TokenMaster      |
| (NFT sales)      | | (ERC-20C trading)| | (backed tokens)  |
+------------------+ +------------------+ +------------------+
        |                    |                    |
+-------------------------------------------------------------+
|  Transfer Validator (foundation -- operator whitelisting)    |
+-------------------------------------------------------------+
        |
+-------------------------------------------------------------+
|  ERC-721C / ERC-1155C / ERC-20C tokens                      |
+-------------------------------------------------------------+
```

## Transfer Validator -- Foundation Layer

**Role**: Enforces which operator contracts can move tokens. Every token transfer (including those initiated by Payment Processor, LBAMM, or TokenMaster) passes through the Transfer Validator check in the token's `_beforeTokenTransfer` hook.

**Integration pattern**: When a creator wants to enable a protocol (e.g., LBAMM) for their token, they add that protocol's contract address to the Transfer Validator's operator whitelist. If a protocol is not whitelisted, it cannot move the creator's tokens -- period.

This means creators have a **single kill switch**: removing a protocol from the whitelist instantly prevents it from transferring tokens, regardless of any other configuration.

## LBAMM -- Exclusive Trading Venue

**Applies to**: ERC-20C tokens

**Role**: When configured as the exclusive trading venue for an ERC-20C token, LBAMM provides:
- AMM-based price discovery with multiple pool types (dynamic, fixed, single-provider)
- Token hooks that enforce per-token policies on every swap (fees, access control, time windows, rewards)
- Pool hooks that enforce per-pool policies (LP restrictions, dynamic fees)
- Handler system for custom settlement (permits, CLOB order books)

**Integration with Transfer Validator**: LBAMM's core contract address must be whitelisted as an operator in the Transfer Validator. When a token is configured to only trade on LBAMM (by whitelisting only LBAMM and no DEX routers), it becomes the exclusive trading venue.

**Integration with Token Hooks**: ERC-20C tokens can register token hooks in LBAMM that fire on every swap. These hooks implement the apptoken behavior patterns (trade fees, time restrictions, access gating, etc.).

## Payment Processor -- NFT Marketplace Layer

**Applies to**: ERC-721C, ERC-1155C tokens

**Role**: Provides a creator-controlled NFT sale and marketplace protocol:
- Royalty enforcement (on-chain, not optional for buyers)
- Creator-configured sale settings (pricing bounds, payment methods, sale timing)
- Multiple pricing modes (individual, collection-level, ascending/descending auctions)
- Cosigner-based security for off-chain order validation

**Integration with Transfer Validator**: Payment Processor's contract address must be whitelisted as an operator. When only Payment Processor is whitelisted (and not general marketplace contracts like OpenSea/Blur), all NFT trades must go through the Payment Processor, enforcing royalties.

## TokenMaster -- Backed Token Layer

**Applies to**: ERC-20C tokens

**Role**: Provides mint/burn pool mechanics for ERC-20C tokens:
- Tokens are minted by depositing reserve assets (e.g., WETH, USDC) into pools
- Tokens are burned to withdraw reserve assets
- Price stability through backing mechanics
- Monetizable spends -- creators earn revenue when tokens are spent (burned) in-app

**Primary trading**: Tokens are bought and sold directly through the TokenMaster Router. The Router handles minting (buy) and burning (sell) against the pool's reserves -- no separate DEX is required.

**Optional LBAMM integration**: For tokens that want to float on a DEX with TokenMaster as a price floor, LBAMM can be added as a secondary trading layer. The correct pattern is: bootstrap supply via TokenMaster buys, then disable TokenMaster buys and add LBAMM liquidity. TokenMaster sells stay enabled as a floor (holders can always burn to redeem reserves). Buys must be disabled to prevent arbitrage — otherwise bots buy from TokenMaster and sell into the DEX, draining reserves. This is an advanced configuration, not the default.

**Integration with Transfer Validator**: TokenMaster's contract address must be whitelisted as an operator to perform mint/burn operations that involve token transfers.

## Trusted Forwarder -- Attribution & Gating Layer

**Factory address**: `0xFF0000B6c4352714cCe809000d0cd30A0E0c8DcE`

**Role**: Sits in front of any protocol to provide:
- **Attribution**: Appends the original caller's address to calldata, enabling web2 apps to route transactions while preserving user identity
- **Gating**: Optionally restricts who can call a protocol to only those with a valid EIP-712 signature from a designated signer

### Two Modes

**Open mode** (no signer configured):
- Relays all transactions to the target contract
- Appends the caller's address to the calldata (last 20 bytes)
- Any address can call -- no permission required

**Permissioned mode** (signer configured):
- Requires an EIP-712 signature from the designated signer before relaying
- The signer is typically a backend service that validates business logic (KYC, account status, rate limits) before signing
- Enables web2-style access control over on-chain operations

### CRITICAL: Refund Behavior

Trusted Forwarders **cannot receive refund payments** -- they have no `fallback` or `receive` function. When a protocol sends a refund (e.g., Payment Processor refunding excess ETH), the refund goes to the **appended caller address** (the original user), not the forwarder. This is by design and ensures users always receive their refunds.

### Deployment

Each app typically deploys its own Trusted Forwarder via the factory:
```
Factory: 0xFF0000B6c4352714cCe809000d0cd30A0E0c8DcE
```
The factory creates minimal proxy forwarders that point to a specific target contract (e.g., LBAMM, Payment Processor, TokenMaster).

## Wrapped Native -- Infrastructure Layer

**Address**: `0x6000030000842044000077551D00cfc6b4005900`

**Role**: Gas-efficient WETH replacement used across the ecosystem wherever native currency wrapping is needed.

### Key Features

- **Payable approve/transfer**: Combines deposit + approve or deposit + transfer into a single transaction, saving gas
- **depositTo(address)**: Deposit ETH and credit it to another address
- **withdrawSplit(address[], uint256[])**: Withdraw and split ETH to multiple recipients in one call
- **Permit functions**: Custom EIP-712 permits (PermitTransfer, PermitWithdrawal) for gasless approvals -- not EIP-2612

### Usage in Ecosystem

- LBAMM pools that pair ERC-20C tokens against native currency use Wrapped Native
- Payment Processor can accept Wrapped Native for NFT purchases
- TokenMaster pools can use Wrapped Native as a reserve asset

## EOA Registry

**Address**: `0xE0A0004Dfa318fc38298aE81a666710eaDCEba5C`

**Role**: On-chain registry of verified externally owned accounts (EOAs). Used by Transfer Validator rulesets that restrict transfers to verified EOAs only, preventing smart contract wallets from receiving tokens (useful for preventing wash trading, MEV, and certain attack vectors).

### Verification Process

1. User signs the literal string `"EOA"` using `personal_sign` (EIP-191)
2. Submit the signature components to the on-chain contract:
   ```solidity
   eoaRegistry.verifySignatureVRS(v, r, s);
   ```
3. The account is permanently registered as a verified EOA
4. Transfer Validator can then check `eoaRegistry.isVerifiedEOA(address)` during transfer validation
