# Position Hook Interface (`ILimitBreakAMMLiquidityHook`)

## Interface Definition

```solidity
interface ILimitBreakAMMLiquidityHook {
    /// @notice Validates adding liquidity and returns hook fees
    /// @param context LiquidityContext with provider, token0, token1, positionId
    /// @param params The LiquidityModificationParams for this operation
    /// @param deposit0 Amount of token0 being deposited
    /// @param deposit1 Amount of token1 being deposited
    /// @param fees0 LP fees in token0 being claimed alongside the deposit
    /// @param fees1 LP fees in token1 being claimed alongside the deposit
    /// @param hookData Arbitrary data passed by the caller for hook logic
    /// @return hookFee0 Fee in token0 charged by this hook
    /// @return hookFee1 Fee in token1 charged by this hook
    function validatePositionAddLiquidity(
        LiquidityContext calldata context,
        LiquidityModificationParams calldata params,
        uint256 deposit0,
        uint256 deposit1,
        uint256 fees0,
        uint256 fees1,
        bytes calldata hookData
    ) external returns (uint256 hookFee0, uint256 hookFee1);

    /// @notice Validates removing liquidity and returns hook fees
    function validatePositionRemoveLiquidity(
        LiquidityContext calldata context,
        LiquidityModificationParams calldata params,
        uint256 withdraw0,
        uint256 withdraw1,
        uint256 fees0,
        uint256 fees1,
        bytes calldata hookData
    ) external returns (uint256 hookFee0, uint256 hookFee1);

    /// @notice Validates collecting fees and returns hook fees
    function validatePositionCollectFees(
        LiquidityContext calldata context,
        LiquidityCollectFeesParams calldata params,
        uint256 fees0,
        uint256 fees1,
        bytes calldata hookData
    ) external returns (uint256 hookFee0, uint256 hookFee1);

    /// @notice Returns URI pointing to a JSON manifest describing the hook
    function liquidityHookManifestUri() external view returns (string memory);
}
```

## LiquidityContext

Provided to all position hook callbacks:

```solidity
struct LiquidityContext {
    address provider;      // The LP performing the operation
    address token0;        // Pool's token0
    address token1;        // Pool's token1
    bytes32 positionId;    // The pool-type-specific position identifier
}
```

## Position Identity

The hook address is embedded in the base position ID:

```
basePositionId = keccak256(abi.encodePacked(provider, liquidityHook, poolId))
```

This has important consequences:
- Two positions from the same provider in the same pool but with different hooks are **separate positions**
- Positions created with `liquidityHook = address(0)` are different from those with a hook
- To remove liquidity or collect fees from a hooked position, the same hook address must be specified
- A provider's liquidity is split across as many positions as distinct (provider, hook, pool) combinations they use

## Hook Call Ordering (Liquidity Operations)

For add/remove liquidity and collect fees:

1. **Token0 hook** (Tier 1): `validateAddLiquidity()` / `validateRemoveLiquidity()` / `validateCollectFees()`
2. **Token1 hook** (Tier 1): same callbacks
3. **Position hook** (Tier 3): `validatePositionAddLiquidity()` / etc.
4. **Pool hook** (Tier 2): `validatePoolAddLiquidity()` / etc.

Position hooks execute after token hooks but before pool hooks. Any tier can revert to block the operation.

## Fee Aggregation

Hook fees from all tiers (token hooks, position hooks, pool hooks) are aggregated per token:

```
totalHookFee0 = token0HookFee0 + token1HookFee0 + positionHookFee0 + poolHookFee0
totalHookFee1 = token0HookFee1 + token1HookFee1 + positionHookFee1 + poolHookFee1
```

If `totalHookFee0 > maxHookFee0` or `totalHookFee1 > maxHookFee1`, the transaction reverts. Callers set these caps in `LiquidityModificationParams.maxHookFee0` / `maxHookFee1`.

## How LPs Interact with Position Hooks

LPs specify the position hook when calling liquidity functions:

```solidity
LiquidityModificationParams memory params = LiquidityModificationParams({
    liquidityHook: address(myPositionHook),  // Set the position hook
    poolId: poolId,
    minLiquidityAmount0: 0,
    minLiquidityAmount1: 0,
    maxLiquidityAmount0: type(uint256).max,
    maxLiquidityAmount1: type(uint256).max,
    maxHookFee0: 1000,    // Max hook fee cap
    maxHookFee1: 1000,
    poolParams: poolTypeSpecificParams
});

// Pass hook-specific data in the liquidityHook field of LiquidityHooksExtraData
LiquidityHooksExtraData memory hooksData = LiquidityHooksExtraData({
    token0Hook: "",
    token1Hook: "",
    liquidityHook: abi.encode(someHookSpecificData),  // Data for position hook
    poolHook: ""
});

ILimitBreakAMMLiquidity(amm).addLiquidity(params, hooksData);
```
