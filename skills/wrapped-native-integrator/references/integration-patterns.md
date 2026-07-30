# Wrapped Native Integration Patterns

Annotated Solidity examples for common WNATIVE integrations.

## Table of Contents

1. [Basic Wrapping and Unwrapping](#basic-wrapping-and-unwrapping)
2. [Payable ERC-20 Patterns](#payable-erc-20-patterns)
3. [Protocol Refund Pattern](#protocol-refund-pattern)
4. [Batch Withdrawal Pattern](#batch-withdrawal-pattern)
5. [Permit Transfer Pattern](#permit-transfer-pattern)
6. [Gas Sponsorship Pattern](#gas-sponsorship-pattern)
7. [EIP-712 Signature Construction](#eip-712-signature-construction)

---

## Basic Wrapping and Unwrapping

```solidity
import {IWrappedNative} from "@limitbreak/wrapped-native/src/interfaces/IWrappedNative.sol";

contract BasicWrapper {
    IWrappedNative public constant WNATIVE =
        IWrappedNative(0x6000030000842044000077551D00cfc6b4005900);

    /// @notice Wrap ETH into WNATIVE for the caller
    function wrap() external payable {
        WNATIVE.deposit{value: msg.value}();
        // WNATIVE is now in this contract's balance
        // Transfer to the caller
        WNATIVE.transfer(msg.sender, msg.value);
    }

    /// @notice Unwrap WNATIVE back to ETH
    /// @dev Caller must approve this contract first
    function unwrap(uint256 amount) external {
        WNATIVE.transferFrom(msg.sender, address(this), amount);
        WNATIVE.withdraw(amount);
        // Send native tokens to caller
        (bool success,) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }

    receive() external payable {}
}
```

---

## Payable ERC-20 Patterns

WNATIVE's payable `approve`, `transfer`, and `transferFrom` combine deposit + action into one call, saving gas and improving UX.

### Deposit + Approve in One Transaction

```solidity
import {IWrappedNative} from "@limitbreak/wrapped-native/src/interfaces/IWrappedNative.sol";

contract PayableApproveExample {
    IWrappedNative public constant WNATIVE =
        IWrappedNative(0x6000030000842044000077551D00cfc6b4005900);

    /// @notice Wrap ETH and approve a spender in a single call
    /// @dev The user sends ETH with this call. WNATIVE.approve is payable,
    ///      so it deposits msg.value before setting the approval.
    function wrapAndApprove(address spender, uint256 approvalAmount) external payable {
        // This single call:
        // 1. Deposits msg.value as WNATIVE (credited to msg.sender since we're the caller)
        // 2. Approves spender for approvalAmount
        //
        // IMPORTANT: Since this contract is calling approve, the deposit credits
        // THIS CONTRACT, not the end user. For direct user interactions, they
        // should call WNATIVE.approve{value: ...}() directly.
        WNATIVE.approve{value: msg.value}(spender, approvalAmount);
    }
}
```

### Deposit + Transfer in One Transaction

```solidity
/// @notice Wrap ETH and transfer WNATIVE to a recipient in one call
function wrapAndSend(address to, uint256 amount) external payable {
    // Deposits msg.value as WNATIVE for msg.sender, then transfers amount to `to`
    // msg.value can be >= amount (excess stays as WNATIVE balance)
    WNATIVE.transfer{value: msg.value}(to, amount);
}
```

---

## Protocol Refund Pattern

When a protocol needs to refund native tokens, `depositTo` is more gas-efficient than `deposit` + `transfer` and reduces re-entrancy risk by avoiding a native transfer to the refundee.

```solidity
import {IWrappedNativeExtended} from "@limitbreak/wrapped-native/src/interfaces/IWrappedNativeExtended.sol";

contract RefundExample {
    IWrappedNativeExtended public constant WNATIVE =
        IWrappedNativeExtended(0x6000030000842044000077551D00cfc6b4005900);

    /// @notice Process a payment, refunding excess as WNATIVE
    function processPayment(uint256 price) external payable {
        require(msg.value >= price, "Insufficient payment");

        uint256 refund = msg.value - price;
        if (refund > 0) {
            // Credits the user's WNATIVE balance directly -- no re-entrancy risk
            WNATIVE.depositTo{value: refund}(msg.sender);
        }

        // ... process the payment with `price` amount of native tokens
    }
}
```

---

## Batch Withdrawal Pattern

`withdrawSplit` distributes native tokens to multiple recipients in one transaction.

```solidity
import {IWrappedNativeExtended} from "@limitbreak/wrapped-native/src/interfaces/IWrappedNativeExtended.sol";

contract PayrollExample {
    IWrappedNativeExtended public constant WNATIVE =
        IWrappedNativeExtended(0x6000030000842044000077551D00cfc6b4005900);

    /// @notice Pay multiple recipients from WNATIVE balance
    /// @dev Caller must have sufficient WNATIVE balance
    function distributePayments(
        address[] calldata recipients,
        uint256[] calldata amounts
    ) external {
        // Transfer WNATIVE from caller to this contract first
        uint256 total;
        for (uint256 i; i < amounts.length; ++i) {
            total += amounts[i];
        }
        WNATIVE.transferFrom(msg.sender, address(this), total);

        // Split-withdraw to all recipients in one call
        WNATIVE.withdrawSplit(recipients, amounts);
    }

    receive() external payable {}
}
```

---

## Permit Transfer Pattern

Gasless token transfers using EIP-712 signed permits.

```solidity
import {IWrappedNativeExtended} from "@limitbreak/wrapped-native/src/interfaces/IWrappedNativeExtended.sol";

contract PermitRelayer {
    IWrappedNativeExtended public constant WNATIVE =
        IWrappedNativeExtended(0x6000030000842044000077551D00cfc6b4005900);

    /// @notice Execute a pre-signed gasless WNATIVE transfer
    /// @dev The `from` account signed an EIP-712 permit authorizing this transfer.
    ///      This contract (msg.sender) must be the operator specified in the permit.
    function executePermitTransfer(
        address from,
        address to,
        uint256 transferAmount,
        uint256 permitAmount,
        uint256 nonce,
        uint256 expiration,
        bytes calldata signedPermit
    ) external {
        // Check that the nonce hasn't been used or revoked
        require(!WNATIVE.isNonceUsed(from, nonce), "Nonce already used");

        WNATIVE.permitTransfer(
            from,
            to,
            transferAmount,
            permitAmount,
            nonce,
            expiration,
            signedPermit
        );
    }
}
```

---

## Gas Sponsorship Pattern

A service that unwraps WNATIVE for users who lack native tokens for gas, charging a convenience fee.

```solidity
import {IWrappedNativeExtended} from "@limitbreak/wrapped-native/src/interfaces/IWrappedNativeExtended.sol";

contract GasSponsor {
    IWrappedNativeExtended public constant WNATIVE =
        IWrappedNativeExtended(0x6000030000842044000077551D00cfc6b4005900);

    address public feeReceiver;
    uint256 public feeBps; // e.g., 25 = 0.25%

    constructor(address _feeReceiver, uint256 _feeBps) {
        feeReceiver = _feeReceiver;
        feeBps = _feeBps;
    }

    /// @notice Execute a permitted withdrawal on behalf of a user
    /// @dev The user signed an EIP-712 withdrawal permit acknowledging the fee.
    ///      The user receives native tokens minus the convenience fee.
    ///      The fee receiver gets WNATIVE (stays wrapped).
    ///      10% of the convenience fee goes to infrastructure tax.
    ///
    /// Example with 0.25% fee on 1 ETH:
    ///   User receives: 0.9975 ETH (native)
    ///   Fee receiver:  0.00225 ETH (WNATIVE)
    ///   Infrastructure: 0.00025 ETH (WNATIVE)
    function sponsorWithdrawal(
        address from,
        address to,
        uint256 amount,
        uint256 nonce,
        uint256 expiration,
        bytes calldata signedPermit
    ) external {
        WNATIVE.doPermittedWithdraw(
            from,
            to,
            amount,
            nonce,
            expiration,
            feeReceiver,
            feeBps,
            signedPermit
        );
    }
}
```

---

## EIP-712 Signature Construction

### Transfer Permit (off-chain signing)

```javascript
// ethers.js v6 example
const domain = {
    name: "Wrapped Native",
    version: "1",
    chainId: 1, // or target chain
    verifyingContract: "0x6000030000842044000077551D00cfc6b4005900"
};

const transferTypes = {
    PermitTransfer: [
        { name: "operator", type: "address" },
        { name: "amount", type: "uint256" },
        { name: "nonce", type: "uint256" },
        { name: "expiration", type: "uint256" },
        { name: "masterNonce", type: "uint256" }
    ]
};

// Get the signer's current master nonce from the contract
const masterNonce = await wnative.masterNonces(signer.address);

const transferValue = {
    operator: relayerAddress,
    amount: parseEther("1.0"),
    nonce: 0, // choose an unused nonce
    expiration: Math.floor(Date.now() / 1000) + 3600, // 1 hour
    masterNonce: masterNonce
};

const signature = await signer.signTypedData(domain, transferTypes, transferValue);
```

### Withdrawal Permit (off-chain signing)

```javascript
const withdrawalTypes = {
    PermitWithdrawal: [
        { name: "operator", type: "address" },
        { name: "amount", type: "uint256" },
        { name: "nonce", type: "uint256" },
        { name: "expiration", type: "uint256" },
        { name: "masterNonce", type: "uint256" },
        { name: "to", type: "address" },
        { name: "convenienceFeeReceiver", type: "address" },
        { name: "convenienceFeeBps", type: "uint256" }
    ]
};

const withdrawalValue = {
    operator: sponsorAddress,
    amount: parseEther("1.0"),
    nonce: 1,
    expiration: Math.floor(Date.now() / 1000) + 3600,
    masterNonce: masterNonce,
    to: userAddress, // where the native tokens go
    convenienceFeeReceiver: feeReceiverAddress,
    convenienceFeeBps: 25 // 0.25%
};

const signature = await signer.signTypedData(domain, withdrawalTypes, withdrawalValue);
```

### Foundry Test Signing

```solidity
// In Foundry tests, use vm.signTypedData or manual hash construction
bytes32 PERMIT_TRANSFER_TYPEHASH = keccak256(
    "PermitTransfer(address operator,uint256 amount,uint256 nonce,uint256 expiration,uint256 masterNonce)"
);

bytes32 structHash = keccak256(abi.encode(
    PERMIT_TRANSFER_TYPEHASH,
    operator,
    amount,
    nonce,
    expiration,
    masterNonce
));

bytes32 domainSeparator = wnative.domainSeparatorV4();
bytes32 digest = keccak256(abi.encodePacked("\x19\x01", domainSeparator, structHash));

(uint8 v, bytes32 r, bytes32 s) = vm.sign(signerPrivateKey, digest);
bytes memory signature = abi.encodePacked(r, s, v);
```
