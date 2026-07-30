# TokenMaster Deployment Workflow

## Step-by-Step Deployment

### Step 1: Choose Pool Type and Factory Address

| Pool Type | Factory Address |
|-----------|----------------|
| Standard | `0x000000c5F2DF717F497BeAcCE161F8b042310d17` |
| Stable | `0x0000006a50a9c9Efae8875266ff222579fC2F449` |
| Promotional | `0x00000014D04B7d1Cad1960eA8980A9af5De2104e` |

### Step 2: Construct PoolDeploymentParameters

```solidity
PoolDeploymentParameters memory poolParams = PoolDeploymentParameters({
    name: "TokenName",
    symbol: "TKN",
    tokenDecimals: 18,
    initialOwner: deployer,
    pairedToken: address(0),             // address(0) = native token
    initialPairedTokenToDeposit: 0,
    encodedInitializationArgs: encodedInitArgs,
    defaultTransferValidator: transferValidator,
    useRouterForPairedTransfers: false,
    partnerFeeRecipient: address(0),
    partnerFeeBPS: 0
});
```

### Step 3: Encode Initialization Args

Pool-specific initialization parameters must be ABI-encoded:

```solidity
// Standard Pool
bytes memory encodedInitArgs = abi.encode(StandardPoolInitializationParameters({...}));

// Stable Pool
bytes memory encodedInitArgs = abi.encode(StablePoolInitializationParameters({...}));

// Promotional Pool
bytes memory encodedInitArgs = abi.encode(PromotionalPoolInitializationParameters({...}));
```

### Step 4: Construct DeploymentParameters

```solidity
DeploymentParameters memory deployParams = DeploymentParameters({
    tokenFactory: factoryAddress,
    tokenSalt: bytes32(uint256(1)),     // unique salt
    tokenAddress: computedAddress,
    blockTransactionsFromUntrustedChannels: false,
    restrictPairingToLists: false,
    poolParams: poolParams,
    maxInfrastructureFeeBPS: 250
});
```

### Step 5: Compute Deterministic Address

```solidity
address computedAddress = factory.computeDeploymentAddress(
    tokenSalt,
    poolParams,
    pairedValueIn,
    infrastructureFeeBPS
);
```

### Step 6: Deploy Token

```solidity
// signature only needed if router has signing authority
router.deployToken{value: msg.value}(deploymentParameters, signature);
```

- For native token pairing: send `msg.value` equal to `initialPairedTokenToDeposit`
- For ERC20 pairing: approve Router before calling

### Step 7: Post-Deployment Settings

```solidity
// Update token settings on Router
router.updateTokenSettings(token, blockTransactionsFromUntrustedChannels, restrictPairingToLists);

// Set order signer for advanced orders
router.setOrderSigner(token, signerAddress, true);

// Configure trusted channels (if blockTransactionsFromUntrustedChannels = true)
router.setTokenAllowedTrustedChannel(token, channel, true);

// Configure pairing restrictions (if restrictPairingToLists = true)
router.setTokenAllowedPairToDeployer(token, deployer, true);
router.setTokenAllowedPairToToken(token, pairedToken, true);
```

## Important Notes

- **useRouterForPairedTransfers:** Set `true` when pairing with an ERC20-C token that has a default operator whitelist including the Router but not individual pools.
- **pairedToken = address(0):** Indicates native token (ETH) pairing. Send `msg.value` for initial deposit.
- **tokenSalt:** Must be unique per deployer per factory. Used for CREATE2 deterministic addressing.
- **setTokenAllowedTrustedChannel:** This function exists on the `TokenMasterRouter` implementation but is NOT declared in `ITokenMasterRouter.sol`. Integrators using the interface ABI must cast to the concrete type or use low-level calls to access it.
- **DEPLOYMENT_TYPEHASH:** For signed deployments, the EIP-712 typehash covers only five fields: `keccak256("DeploymentParameters(address tokenFactory,bytes32 tokenSalt,address tokenAddress,bool blockTransactionsFromUntrustedChannels,bool restrictPairingToLists)")`. The `poolParams` and `maxInfrastructureFeeBPS` fields are NOT part of the signed digest. The signer authority is the `TOKENMASTER_SIGNER_ROLE` holder in the Role Server.
