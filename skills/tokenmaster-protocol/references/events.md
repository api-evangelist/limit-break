# TokenMaster Events

## Router Admin Events

```solidity
event InfrastructureFeeUpdated(uint16 infrastructureFeeBPS);
event AllowedTokenFactoryUpdated(address indexed tokenFactory, bool allowed);
```

## Token Lifecycle Events

```solidity
event TokenMasterTokenDeployed(address indexed tokenMasterToken, address indexed pairedToken, address indexed tokenFactory);

event TokenSettingsUpdated(
    address indexed tokenMasterToken,
    bool blockTransactionsFromUntrustedChannels,
    bool restrictPairingToLists
);

event TrustedChannelUpdated(
    address indexed tokenAddress,
    address indexed channel,
    bool allowed
);

event AllowedPairToDeployersUpdated(
    address indexed tokenAddress,
    address indexed deployer,
    bool allowed
);

event AllowedPairToTokensUpdated(
    address indexed tokenAddress,
    address indexed tokenAllowedToPair,
    bool allowed
);
```

## Order Fill Events

```solidity
event BuyOrderFilled(
    address indexed tokenMasterToken,
    address indexed buyer,
    uint256 amountPurchased,
    uint256 totalCost
);

event SellOrderFilled(
    address indexed tokenMasterToken,
    address indexed seller,
    uint256 amountSold,
    uint256 totalReceived
);

event SpendOrderFilled(
    address indexed tokenMasterToken,
    bytes32 indexed creatorSpendIdentifier,
    address indexed spender,
    uint256 amountSpent,
    uint256 multiplier
);
```

## Order Configuration Events

```solidity
event OrderSignerUpdated(
    address indexed tokenMasterToken,
    address indexed signer,
    bool allowed
);

event BuyOrderDisabled(
    address indexed tokenMasterToken,
    bytes32 indexed creatorBuyIdentifier,
    bool disabled
);

event SellOrderDisabled(
    address indexed tokenMasterToken,
    bytes32 indexed creatorSellIdentifier,
    bool disabled
);

event SpendOrderDisabled(
    address indexed tokenMasterToken,
    bytes32 indexed creatorSpendIdentifier,
    bool disabled
);
```

## Partner Fee Events

```solidity
event PartnerFeeRecipientProposed(
    address indexed tokenAddress,
    address proposedPartnerFeeRecipient
);

event PartnerFeeRecipientUpdated(
    address indexed tokenAddress,
    address partnerFeeRecipient
);
```

## Pool Events (emitted by pool contracts)

```solidity
// Emitted by ITokenMasterERC20C implementations
event CreatorShareWithdrawn(address to, uint256 withdrawAmount, uint256 infrastructureAmount, uint256 partnerAmount);
event CreatorShareTransferredToMarket(address to, uint256 transferAmount, uint256 infrastructureAmount, uint256 partnerAmount);

// Emitted by ICreatorEmissionsPool implementations (StandardPool)
event CreatorEmissionsHardCapUpdated(uint256 newHardCapAmount);
event CreatorEmissionsClaimed(address to, uint256 claimedAmount, uint256 forfeitedAmount);

// Emitted by pool contracts when parameters are updated
event BuyParametersUpdated();
event SellParametersUpdated();
event SpendParametersUpdated();  // StandardPool only
```

## Fee Distribution

Four shares tracked via `pairedTokenShares()`:

| Share | Purpose |
|-------|---------|
| market | Backs token value (bonding curve reserves) |
| creator | Withdrawable by creator via `withdrawCreatorShare` |
| infrastructure | Protocol fee (withdrawn via `withdrawFees`) |
| partner | Optional partner fee (withdrawn via `withdrawFees`) |

## Partner Fee Transfer

Two-step process:
1. `partnerProposeFeeReceiver` -- proposes a new partner fee recipient
2. `acceptProposedPartnerFeeReceiver` -- proposed recipient accepts
