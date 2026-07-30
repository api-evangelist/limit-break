# TokenMaster Hook Interfaces

## ITokenMasterBuyHook

```solidity
interface ITokenMasterBuyHook {
    function tokenMasterBuyHook(
        address tokenMasterToken,
        address buyer,
        bytes32 creatorBuyIdentifier,
        uint256 amountPurchased,
        bytes calldata hookExtraData
    ) external;
}
```

## ITokenMasterSellHook

```solidity
interface ITokenMasterSellHook {
    function tokenMasterSellHook(
        address tokenMasterToken,
        address seller,
        bytes32 creatorSellIdentifier,
        uint256 amountSold,
        bytes calldata hookExtraData
    ) external;
}
```

## ITokenMasterSpendHook

```solidity
interface ITokenMasterSpendHook {
    function tokenMasterSpendHook(
        address tokenMasterToken,
        address spender,
        bytes32 creatorSpendIdentifier,
        uint256 multiplier,
        bytes calldata hookExtraData
    ) external;
}
```

## Key Rules

- Hooks are called by the **Router**, NOT by the user directly.
- **MUST** validate `msg.sender == TOKENMASTER_ROUTER`.
- **MUST** validate the `tokenMasterToken` and `creatorIdentifier` match expected values.
- For spend hooks: `multiplier` is passed, NOT the token amount. Actual amount = `baseValue * multiplier` (optionally adjusted by oracle).
- `hookExtraData`: arbitrary bytes passed through from the user's transaction. Decoding is entirely the hook's responsibility.
