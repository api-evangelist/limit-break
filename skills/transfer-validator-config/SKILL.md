---
name: transfer-validator-config
description: Generate Transfer Validator V5 configuration scripts -- rulesets, lists, whitelisting, account freezing for Apptoken collections
user-invocable: true
disable-model-invocation: true
argument-hint: "[description of transfer validation requirements]"
---

# Transfer Validator V5 Configuration Generator

Generate Foundry scripts or transaction sequences to configure Transfer Validator V5 for a token collection.

## Project Setup

Before generating code, check the user's project is configured:
1. **Foundry**: `foundry.toml` must exist. If not: `forge init`
2. **Creator Token Standards**: `lib/creator-token-standards/` must exist. If not: `forge install limitbreakinc/creator-token-standards`
3. **Remappings**: `remappings.txt` needs `@limitbreak/creator-token-standards/=lib/creator-token-standards/`. Append if missing -- do not overwrite existing entries.
4. **Local testing**: For local development, use `apptoken-dev` to bootstrap an Anvil node with all Apptoken protocols at their deterministic addresses -- see `apptoken-general/references/project-setup.md`.

## Instructions

1. **Parse requirements** from `$ARGUMENTS` -- identify what the collection needs (ruleset, whitelisted operators, receiver constraints, account freezing, etc.)

2. **Resolve ambiguities before generating** -- if $ARGUMENTS mentions a collection without its address, operators to whitelist without specifying which protocols/addresses, or a ruleset preference isn't clear, ask in a single message. If clear, proceed.

3. **Determine the configuration steps needed:**
   - Ruleset selection (`setRulesetOfCollection`) -- Whitelist (0/4), Vanilla (1), Soulbound (2), Blacklist (3), or custom (255)
   - Global options (GO0-GO3) -- disable authorization, no wildcard authorizers, account freezing, default list extension
   - Whitelist options (WLO0-WLO4) -- OTC blocking, 7702 delegate OTC, smart wallet OTC, receiver constraints
   - List management -- create lists, add/remove accounts or code hashes
   - Account freezing -- freeze/unfreeze specific addresses (requires GO2 enabled)

4. **Generate a Foundry script** that calls the Transfer Validator at `0x721C008fdff27BF06E7E123956E2Fe03B63342e3`

5. **Explain each configuration choice** and its security implications

## When to Use This vs Other Skills

- **This skill**: Configure Transfer Validator -- rulesets, operator whitelists, receiver constraints, account freezing. This is the **foundation layer** for all apptokens.
- **lbamm-standard-hook**: Configure LBAMM-specific trading behavior (fees, pause, LP whitelists). Use *after* Transfer Validator is set up.
- **payment-processor-creator**: Configure NFT sale settings (pricing, royalties, trusted channels). Use *after* Transfer Validator is set up.

## Common Mistakes

- Forgetting to whitelist protocol operators (LBAMM, TokenMaster, Payment Processor) -- tokens become untradeable
- Setting `customRuleset` to non-zero when `rulesetId` is not 255 -- will revert
- Enabling account freezing (GO2) without first calling `setRulesetOfCollection` with the GO2 flag -- freezing has no effect
- Using WLO flags with a non-whitelist ruleset (ID 1, 2, or 3) -- WLO flags are ignored

## Reference Files

- `references/collection-config.md` -- Collection configuration API, workflow examples, expansion settings
- `references/config-patterns.md` -- Common configuration patterns with complete Foundry scripts
- Protocol knowledge: `transfer-validator-protocol`

## Related Skills

- **apptoken-general** -- ERC-721C/ERC-1155C/ERC-20C token standards and Transfer Validator integration patterns
