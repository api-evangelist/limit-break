# Supported Chains

## Deployment Model

All Apptoken ecosystem contracts use **deterministic CREATE2 deployment** -- the same contract address on every supported chain. If a contract address is listed in these skills, it is the same address on all chains where the protocol is deployed.

## Core Infrastructure (all chains)

These contracts are deployed on every supported chain at the same addresses:

| Contract | Address |
|----------|---------|
| Transfer Validator V5 | `0x721C008fdff27BF06E7E123956E2Fe03B63342e3` |
| EOA Registry | `0xE0A0004Dfa318fc38298aE81a666710eaDCEba5C` |
| Trusted Forwarder Factory | `0xFF0000B6c4352714cCe809000d0cd30A0E0c8DcE` |
| Wrapped Native | `0x6000030000842044000077551D00cfc6b4005900` |

## Protocol-Specific Contracts

| Contract | Address | Notes |
|----------|---------|-------|
| Payment Processor V3 | `0x9a1D00000000fC540e2000560054812452eB5366` | NFT trading (ERC-721C, ERC-1155C) |
| PP Encoder | `0x9A1D00C3a699f491037745393a0592AC6b62421D` | Calldata encoding for PP |
| PP Settings Registry | `0x9A1D001A842c5e6C74b33F2aeEdEc07F0Cb20BC4` | Creator settings management |
| TokenMaster Router | `0x0E00009d00d1000069ed00A908e00081F5006008` | ERC-20C token pools |
| TM Standard Factory | `0x000000c5F2DF717F497BeAcCE161F8b042310d17` | Standard pool deployment |
| TM Stable Factory | `0x0000006a50a9c9Efae8875266ff222579fC2F449` | Stable pool deployment |
| TM Promotional Factory | `0x00000014D04B7d1Cad1960eA8980A9af5De2104e` | Promotional pool deployment |

## Checking Chain Support

To verify a protocol is deployed on a specific chain, check that the contract address has code:

```solidity
require(address(0x721C008fdff27BF06E7E123956E2Fe03B63342e3).code.length > 0, "TV not deployed on this chain");
```

## Dependencies (forge install)

| Command | Contents |
|---------|----------|
| `forge install limitbreakinc/creator-token-standards` | ERC-721C, ERC-1155C, ERC-20C base contracts |
| `forge install limitbreakinc/lbamm-core` | LBAMM core interfaces, hooks, integrators |
| `forge install limitbreakinc/lbamm-hooks-and-handlers` | LBAMM standard hook, PermitTransferHandler, CLOBTransferHandler |
| `forge install limitbreakinc/lbamm-pool-type-fixed` | LBAMM Fixed pool type |
| `forge install limitbreakinc/lbamm-pool-type-single-provider` | LBAMM Single Provider pool type |
| `forge install limitbreakinc/amm-pool-type-dynamic` | LBAMM Dynamic pool type |
| `forge install limitbreakinc/tm-tokenmaster` | TokenMaster interfaces and pool contracts |
| `forge install limitbreakinc/creator-token-transfer-validator` | Transfer Validator V5 (includes PermitC) |
| `forge install limitbreakinc/payment-processor` | Payment Processor V3 |
| `forge install limitbreakinc/PermitC` | PermitC standalone |
| `forge install limitbreakinc/wrapped-native` | Wrapped Native (WNATIVE) interfaces and contract |
