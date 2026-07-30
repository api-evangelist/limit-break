# Position Hook Patterns

## Table of Contents

- [Pattern 1: Time-Lock Hook](#pattern-1-time-lock-hook)
- [Pattern 2: Fee-on-Deposit Hook](#pattern-2-fee-on-deposit-hook)
- [Pattern 3: Allowlist-Gated Hook](#pattern-3-allowlist-gated-hook)
- [Pattern 4: Early Withdrawal Penalty Hook](#pattern-4-early-withdrawal-penalty-hook)
- [Common Design Considerations](#common-design-considerations)

## Pattern 1: Time-Lock Hook

Prevents LPs from removing liquidity before a minimum lock period. Useful for protocol stability, preventing JIT liquidity attacks, or incentivizing long-term LP commitment.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {ILimitBreakAMMLiquidityHook} from "@limitbreak/lbamm-core/src/interfaces/hooks/ILimitBreakAMMLiquidityHook.sol";
import {LiquidityContext, LiquidityModificationParams, LiquidityCollectFeesParams} from "@limitbreak/lbamm-core/src/DataTypes.sol";

contract TimeLockPositionHook is ILimitBreakAMMLiquidityHook {
    error TimeLockPositionHook__PositionLocked();
    error TimeLockPositionHook__CallerNotAMM();

    address public immutable amm;
    uint256 public immutable lockDuration;

    // positionId => timestamp when liquidity was first added
    mapping(bytes32 => uint256) public positionDepositTime;

    constructor(address _amm, uint256 _lockDuration) {
        amm = _amm;
        lockDuration = _lockDuration;
    }

    modifier onlyAMM() {
        if (msg.sender != amm) revert TimeLockPositionHook__CallerNotAMM();
        _;
    }

    function validatePositionAddLiquidity(
        LiquidityContext calldata context,
        LiquidityModificationParams calldata,
        uint256, uint256, uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256, uint256) {
        // Record deposit time on first add (don't overwrite on subsequent adds)
        if (positionDepositTime[context.positionId] == 0) {
            positionDepositTime[context.positionId] = block.timestamp;
        }
        return (0, 0);
    }

    function validatePositionRemoveLiquidity(
        LiquidityContext calldata context,
        LiquidityModificationParams calldata,
        uint256, uint256, uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256, uint256) {
        uint256 depositTime = positionDepositTime[context.positionId];
        if (depositTime != 0 && block.timestamp < depositTime + lockDuration) {
            revert TimeLockPositionHook__PositionLocked();
        }
        return (0, 0);
    }

    function validatePositionCollectFees(
        LiquidityContext calldata,
        LiquidityCollectFeesParams calldata,
        uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256, uint256) {
        // Fee collection is always allowed -- only withdrawals are locked
        return (0, 0);
    }

    function liquidityHookManifestUri() external pure returns (string memory) {
        return "";
    }
}
```

## Pattern 2: Fee-on-Deposit Hook

Charges a percentage fee when LPs add liquidity. The fee is accrued in LBAMM core and can be collected by the hook contract later. Useful for protocol revenue or discouraging frequent repositioning.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {ILimitBreakAMMLiquidityHook} from "@limitbreak/lbamm-core/src/interfaces/hooks/ILimitBreakAMMLiquidityHook.sol";
import {LiquidityContext, LiquidityModificationParams, LiquidityCollectFeesParams} from "@limitbreak/lbamm-core/src/DataTypes.sol";

contract DepositFeePositionHook is ILimitBreakAMMLiquidityHook {
    error DepositFeePositionHook__CallerNotAMM();

    address public immutable amm;
    uint256 public immutable feeBPS; // Fee in basis points (100 = 1%)

    constructor(address _amm, uint256 _feeBPS) {
        amm = _amm;
        feeBPS = _feeBPS;
    }

    modifier onlyAMM() {
        if (msg.sender != amm) revert DepositFeePositionHook__CallerNotAMM();
        _;
    }

    function validatePositionAddLiquidity(
        LiquidityContext calldata,
        LiquidityModificationParams calldata,
        uint256 deposit0,
        uint256 deposit1,
        uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256 hookFee0, uint256 hookFee1) {
        // Charge a percentage of the deposit as a hook fee
        hookFee0 = (deposit0 * feeBPS) / 10_000;
        hookFee1 = (deposit1 * feeBPS) / 10_000;
    }

    function validatePositionRemoveLiquidity(
        LiquidityContext calldata,
        LiquidityModificationParams calldata,
        uint256, uint256, uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256, uint256) {
        return (0, 0);
    }

    function validatePositionCollectFees(
        LiquidityContext calldata,
        LiquidityCollectFeesParams calldata,
        uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256, uint256) {
        return (0, 0);
    }

    function liquidityHookManifestUri() external pure returns (string memory) {
        return "";
    }
}
```

## Pattern 3: Allowlist-Gated Hook

Only whitelisted addresses can add liquidity through this hook. Useful for KYC-gated LP programs, institutional liquidity pools, or invite-only LP opportunities.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ILimitBreakAMMLiquidityHook} from "@limitbreak/lbamm-core/src/interfaces/hooks/ILimitBreakAMMLiquidityHook.sol";
import {LiquidityContext, LiquidityModificationParams, LiquidityCollectFeesParams} from "@limitbreak/lbamm-core/src/DataTypes.sol";

contract AllowlistPositionHook is ILimitBreakAMMLiquidityHook, Ownable {
    error AllowlistPositionHook__CallerNotAMM();
    error AllowlistPositionHook__ProviderNotAllowed();

    address public immutable amm;
    mapping(address => bool) public allowedProviders;

    constructor(address _amm) Ownable(msg.sender) {
        amm = _amm;
    }

    modifier onlyAMM() {
        if (msg.sender != amm) revert AllowlistPositionHook__CallerNotAMM();
        _;
    }

    function setProviderAllowed(address provider, bool allowed) external onlyOwner {
        allowedProviders[provider] = allowed;
    }

    function setProvidersAllowed(address[] calldata providers, bool allowed) external onlyOwner {
        for (uint256 i = 0; i < providers.length; i++) {
            allowedProviders[providers[i]] = allowed;
        }
    }

    function validatePositionAddLiquidity(
        LiquidityContext calldata context,
        LiquidityModificationParams calldata,
        uint256, uint256, uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256, uint256) {
        if (!allowedProviders[context.provider]) {
            revert AllowlistPositionHook__ProviderNotAllowed();
        }
        return (0, 0);
    }

    function validatePositionRemoveLiquidity(
        LiquidityContext calldata,
        LiquidityModificationParams calldata,
        uint256, uint256, uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256, uint256) {
        // Always allow removal -- don't trap liquidity
        return (0, 0);
    }

    function validatePositionCollectFees(
        LiquidityContext calldata,
        LiquidityCollectFeesParams calldata,
        uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256, uint256) {
        return (0, 0);
    }

    function liquidityHookManifestUri() external pure returns (string memory) {
        return "";
    }
}
```

## Pattern 4: Early Withdrawal Penalty Hook

Charges a declining penalty fee for removing liquidity before a maturity date. The penalty starts high and decreases linearly to zero. Combines time-lock incentives with flexibility -- LPs can always exit, but pay a fee for doing so early.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {ILimitBreakAMMLiquidityHook} from "@limitbreak/lbamm-core/src/interfaces/hooks/ILimitBreakAMMLiquidityHook.sol";
import {LiquidityContext, LiquidityModificationParams, LiquidityCollectFeesParams} from "@limitbreak/lbamm-core/src/DataTypes.sol";

contract EarlyWithdrawalPenaltyHook is ILimitBreakAMMLiquidityHook {
    error EarlyWithdrawalPenaltyHook__CallerNotAMM();

    address public immutable amm;
    uint256 public immutable maturityDuration;  // Time until penalty reaches zero
    uint256 public immutable maxPenaltyBPS;     // Maximum penalty at deposit time

    mapping(bytes32 => uint256) public positionDepositTime;

    constructor(address _amm, uint256 _maturityDuration, uint256 _maxPenaltyBPS) {
        amm = _amm;
        maturityDuration = _maturityDuration;
        maxPenaltyBPS = _maxPenaltyBPS;
    }

    modifier onlyAMM() {
        if (msg.sender != amm) revert EarlyWithdrawalPenaltyHook__CallerNotAMM();
        _;
    }

    function validatePositionAddLiquidity(
        LiquidityContext calldata context,
        LiquidityModificationParams calldata,
        uint256, uint256, uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256, uint256) {
        if (positionDepositTime[context.positionId] == 0) {
            positionDepositTime[context.positionId] = block.timestamp;
        }
        return (0, 0);
    }

    function validatePositionRemoveLiquidity(
        LiquidityContext calldata context,
        LiquidityModificationParams calldata,
        uint256 withdraw0,
        uint256 withdraw1,
        uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256 hookFee0, uint256 hookFee1) {
        uint256 depositTime = positionDepositTime[context.positionId];
        if (depositTime == 0) return (0, 0);

        uint256 elapsed = block.timestamp - depositTime;
        if (elapsed >= maturityDuration) return (0, 0);

        // Linear decay: penalty = maxPenalty * (remaining / total)
        uint256 remaining = maturityDuration - elapsed;
        uint256 penaltyBPS = (maxPenaltyBPS * remaining) / maturityDuration;

        hookFee0 = (withdraw0 * penaltyBPS) / 10_000;
        hookFee1 = (withdraw1 * penaltyBPS) / 10_000;
    }

    function validatePositionCollectFees(
        LiquidityContext calldata,
        LiquidityCollectFeesParams calldata,
        uint256, uint256,
        bytes calldata
    ) external onlyAMM returns (uint256, uint256) {
        return (0, 0);
    }

    function liquidityHookManifestUri() external pure returns (string memory) {
        return "";
    }
}
```

## Common Design Considerations

### Always Allow Removals (or Charge a Fee)
Avoid trapping liquidity permanently. If you gate `validatePositionAddLiquidity`, always allow `validatePositionRemoveLiquidity` to succeed (possibly with a fee). Users should always be able to exit a position.

### Validate `msg.sender`
Always check `msg.sender == amm` in callbacks. Without this, anyone can call the hook functions directly and potentially manipulate state.

### Position ID Stability
The `positionId` in `LiquidityContext` is derived from the provider, hook address, pool ID, and pool-type-specific parameters (e.g., tick range for dynamic pools). Use it as a stable key for tracking per-position state.

### Fee Collection
Hook fees are accrued in LBAMM core, not transferred to the hook directly. To collect accrued fees, the hook (or its owner) must call the AMM's fee collection functions. If the token's `hookManagesFees` flag (bit 6) is set on the token hook, fees are tracked per-hook and collected via `collectHookFeesByHook`.
