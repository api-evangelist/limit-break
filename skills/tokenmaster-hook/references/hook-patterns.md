# TokenMaster Hook Patterns

## Pattern 1: Buy Hook - Mint Promotional Tokens

After buying tokens, mint promotional tokens as a reward.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ITokenMasterBuyHook} from "./ITokenMasterBuyHook.sol";

interface IPromotionalPool {
    function mint(address to, uint256 amount) external;
}

contract BuyRewardHook is ITokenMasterBuyHook {
    address public immutable TOKENMASTER_ROUTER;
    address public immutable EXPECTED_TOKEN;
    bytes32 public immutable EXPECTED_IDENTIFIER;
    IPromotionalPool public immutable PROMOTIONAL_POOL;
    uint256 public immutable REWARD_RATE; // e.g., 100 = 1:1, 200 = 2:1

    constructor(
        address router,
        address expectedToken,
        bytes32 expectedIdentifier,
        address promotionalPool,
        uint256 rewardRate
    ) {
        TOKENMASTER_ROUTER = router;
        EXPECTED_TOKEN = expectedToken;
        EXPECTED_IDENTIFIER = expectedIdentifier;
        PROMOTIONAL_POOL = IPromotionalPool(promotionalPool);
        REWARD_RATE = rewardRate;
    }

    function tokenMasterBuyHook(
        address tokenMasterToken,
        address buyer,
        bytes32 creatorBuyIdentifier,
        uint256 amountPurchased,
        bytes calldata /* hookExtraData */
    ) external override {
        require(msg.sender == TOKENMASTER_ROUTER, "Caller must be Router");
        require(tokenMasterToken == EXPECTED_TOKEN, "Unexpected token");
        require(creatorBuyIdentifier == EXPECTED_IDENTIFIER, "Unexpected identifier");

        uint256 rewardAmount = (amountPurchased * REWARD_RATE) / 100;
        PROMOTIONAL_POOL.mint(buyer, rewardAmount);
    }
}
```

---

## Pattern 2: Sell Hook - Mint Promotional Tokens

Same pattern for sells, rewarding sellers with promotional tokens.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ITokenMasterSellHook} from "./ITokenMasterSellHook.sol";

interface IPromotionalPool {
    function mint(address to, uint256 amount) external;
}

contract SellRewardHook is ITokenMasterSellHook {
    address public immutable TOKENMASTER_ROUTER;
    address public immutable EXPECTED_TOKEN;
    bytes32 public immutable EXPECTED_IDENTIFIER;
    IPromotionalPool public immutable PROMOTIONAL_POOL;
    uint256 public immutable REWARD_RATE;

    constructor(
        address router,
        address expectedToken,
        bytes32 expectedIdentifier,
        address promotionalPool,
        uint256 rewardRate
    ) {
        TOKENMASTER_ROUTER = router;
        EXPECTED_TOKEN = expectedToken;
        EXPECTED_IDENTIFIER = expectedIdentifier;
        PROMOTIONAL_POOL = IPromotionalPool(promotionalPool);
        REWARD_RATE = rewardRate;
    }

    function tokenMasterSellHook(
        address tokenMasterToken,
        address seller,
        bytes32 creatorSellIdentifier,
        uint256 amountSold,
        bytes calldata /* hookExtraData */
    ) external override {
        require(msg.sender == TOKENMASTER_ROUTER, "Caller must be Router");
        require(tokenMasterToken == EXPECTED_TOKEN, "Unexpected token");
        require(creatorSellIdentifier == EXPECTED_IDENTIFIER, "Unexpected identifier");

        uint256 rewardAmount = (amountSold * REWARD_RATE) / 100;
        PROMOTIONAL_POOL.mint(seller, rewardAmount);
    }
}
```

---

## Pattern 3: Spend Hook - Mint ERC721C NFTs

After spending tokens, mint NFTs. The multiplier determines how many NFTs to mint. Ideal for game item shops.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ITokenMasterSpendHook} from "./ITokenMasterSpendHook.sol";

interface IERC721CMintable {
    function mint(address to, uint256 quantity) external;
}

contract SpendMintNFTHook is ITokenMasterSpendHook {
    address public immutable TOKENMASTER_ROUTER;
    address public immutable EXPECTED_TOKEN;
    bytes32 public immutable EXPECTED_IDENTIFIER;
    IERC721CMintable public immutable NFT_CONTRACT;

    constructor(
        address router,
        address expectedToken,
        bytes32 expectedIdentifier,
        address nftContract
    ) {
        TOKENMASTER_ROUTER = router;
        EXPECTED_TOKEN = expectedToken;
        EXPECTED_IDENTIFIER = expectedIdentifier;
        NFT_CONTRACT = IERC721CMintable(nftContract);
    }

    function tokenMasterSpendHook(
        address tokenMasterToken,
        address spender,
        bytes32 creatorSpendIdentifier,
        uint256 multiplier,
        bytes calldata /* hookExtraData */
    ) external override {
        require(msg.sender == TOKENMASTER_ROUTER, "Caller must be Router");
        require(tokenMasterToken == EXPECTED_TOKEN, "Unexpected token");
        require(creatorSpendIdentifier == EXPECTED_IDENTIFIER, "Unexpected identifier");

        // multiplier determines how many NFTs to mint
        NFT_CONTRACT.mint(spender, multiplier);
    }
}
```

---

## Pattern 4: Oracle Hook - Price Pegging

ITokenMasterOracle implementation that adjusts baseValue to maintain a stablecoin peg by querying an external price feed.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface ITokenMasterOracle {
    function adjustValue(
        uint256 transactionType,
        address executor,
        address tokenMasterToken,
        address baseToken,
        uint256 baseValue,
        bytes calldata oracleExtraData
    ) external view returns (uint256 tokenValue);
}

interface IPriceFeed {
    function latestRoundData() external view returns (
        uint80 roundId,
        int256 answer,
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    );
}

contract PricePegOracle is ITokenMasterOracle {
    IPriceFeed public immutable PRICE_FEED;
    uint256 public immutable TARGET_PRICE;   // target price in feed decimals
    uint256 public immutable FEED_DECIMALS;

    constructor(address priceFeed, uint256 targetPrice, uint256 feedDecimals) {
        PRICE_FEED = IPriceFeed(priceFeed);
        TARGET_PRICE = targetPrice;
        FEED_DECIMALS = feedDecimals;
    }

    function adjustValue(
        uint256 /* transactionType */,
        address /* executor */,
        address /* tokenMasterToken */,
        address /* baseToken */,
        uint256 baseValue,
        bytes calldata /* oracleExtraData */
    ) external view override returns (uint256 tokenValue) {
        (, int256 answer,,,) = PRICE_FEED.latestRoundData();
        require(answer > 0, "Invalid price feed");

        uint256 currentPrice = uint256(answer);

        // Adjust baseValue to maintain peg
        // If current price is above target, reduce token value (fewer tokens per unit)
        // If current price is below target, increase token value (more tokens per unit)
        tokenValue = (baseValue * TARGET_PRICE) / currentPrice;
    }
}
```

---

## Security Considerations

1. **Always validate `msg.sender`:** Only the Router should call hook functions. Failing to check this allows anyone to trigger hook logic.
2. **Always validate token address:** Ensure the `tokenMasterToken` matches the expected token to prevent cross-token attacks.
3. **Always validate identifier:** The `creatorIdentifier` (buy/sell/spend) must match the expected value to prevent unauthorized order reuse.
4. **Handle hookExtraData carefully:** If decoding hookExtraData, validate its structure and bounds. Malformed data should revert gracefully.
5. **Reentrancy:** Consider reentrancy guards if the hook makes external calls to untrusted contracts.
6. **Gas limits:** Hooks execute within the Router's transaction. Excessive gas consumption can cause the entire transaction to fail.
