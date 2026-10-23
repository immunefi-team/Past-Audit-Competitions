# Attackathon \_ Fuel Network 33181 - \[Smart Contract - Insight] users messages might encode incorrect

Submitted on Sat Jul 13 2024 14:54:16 GMT-0400 (Atlantic Standard Time) by @zeroK for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33181

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-bridge/tree/e3e673e31f9e72d757d68979bb6796a0b7f9c8bc/packages/fungible-token>

Impacts:

* Permanent freezing of funds
* Permanent freezing of unclaimed yield
* Contract fails to deliver promised returns, but doesn't lose value
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Block stuffing

## Description

## Brief/Intro

the fuelERC20gateway.sol allow users to deposit tokens with/without data to L2 by calling the functions deposit and depositWithData, however we recognized that the assetIssuerID play a crusial rule in encoding the data that later used to generate message on L2, the problem arises when users call deposit or depositWthData before the admin make call to the `setAssetIssuerId`, this lead to encode incorrect data and pass it to L2 sequencer which may lead to return incorrect message data input.

we set this report to high because on the sway L2 part the inputs cames from the bytes that we encoded and incorrect bytes mean incorrect value on L2 sides and its possible to happen only in time gap between deployed gateway and calling the setAssetIssuerId.

## Vulnerability Details

the function deposit and DepositWithData and even sendMetadata use assetIssuerID as one of the params to encode valid message data:

```solidity
    function deposit(bytes32 to, address tokenAddress, uint256 amount) external payable virtual whenNotPaused {
        uint8 decimals = _getTokenDecimals(tokenAddress);
        uint256 l2MintedAmount = _adjustDepositDecimals(decimals, amount);

        bytes memory depositMessage = abi.encodePacked(
            assetIssuerId,
            uint256(MessageType.DEPOSIT_TO_ADDRESS),
            bytes32(uint256(uint160(tokenAddress))),
            uint256(0), // token_id = 0 for all erc20 deposits
            bytes32(uint256(uint160(msg.sender))),
            to,
            l2MintedAmount,
            uint256(decimals)
        );
        _deposit(tokenAddress, amount, l2MintedAmount, depositMessage);
    }

    /// @notice Deposits the given tokens to a contract on Fuel with optional data
    /// @param to Fuel account or contract to deposit tokens to
    /// @param tokenAddress Address of the token being transferred to Fuel
    /// @param amount Amount of tokens to deposit
    /// @param data Optional data to send with the deposit
    /// @dev Made payable to reduce gas costs
    function depositWithData(
        bytes32 to,
        address tokenAddress,
        uint256 amount,
        bytes calldata data
    ) external payable virtual whenNotPaused {
        uint8 decimals = _getTokenDecimals(tokenAddress);
        uint256 l2MintedAmount = _adjustDepositDecimals(decimals, amount);

        bytes memory depositMessage = abi.encodePacked(
            assetIssuerId,
            uint256(data.length == 0 ? MessageType.DEPOSIT_TO_CONTRACT : MessageType.DEPOSIT_WITH_DATA),
            bytes32(uint256(uint160(tokenAddress))),
            uint256(0), // token_id = 0 for all erc20 deposits
            bytes32(uint256(uint160(msg.sender))),
            to,
            l2MintedAmount,
            uint256(decimals),
            data
        );
        _deposit(tokenAddress, amount, l2MintedAmount, depositMessage);
    }

    function sendMetadata(address tokenAddress) external payable virtual whenNotPaused {
        bytes memory metadataMessage = abi.encodePacked(
            assetIssuerId,
            uint256(MessageType.METADATA),
            abi.encode(
                tokenAddress,
                uint256(0), // token_id = 0 for all erc20 deposits
                IERC20MetadataUpgradeable(tokenAddress).symbol(),
                IERC20MetadataUpgradeable(tokenAddress).name()
            )
        );
        sendMessage(CommonPredicates.CONTRACT_MESSAGE_PREDICATE, metadataMessage);
    }

```

the problem is that the assetIssuerId is not set when the implementation initialize function get called which allow the scenario below to be possible:

* the fuelERC20Gateway.sol deployed on ethereum and since its impl, the proxy made call to the initialize function which shown as below:

```solidity

    function initialize(FuelMessagePortal fuelMessagePortal) public initializer {
        __FuelMessagesEnabled_init(fuelMessagePortal);
        __Pausable_init();
        __AccessControl_init();
        __UUPSUpgradeable_init();

        //grant initial roles
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(PAUSER_ROLE, msg.sender);
    }


```

* as shown the assetIssuerID not set in the initialize function and it depends on the admin to call the `setAssetIssuerId`

```solidity
    function setAssetIssuerId(bytes32 id) external payable virtual onlyRole(DEFAULT_ADMIN_ROLE) {
        assetIssuerId = id;
    }
```

* between the deploy time and calling setAssetIssuerId, any user can make deposit call or send metadata call with the assetIssuerID equal to zero bytes which this lead to create incorrect messageData which in its turn lead to two cases:

case A : the sequencers on L2 fuel not create the messageID or set it to invalid transaction since the assetIssuer not equal to the bridge address and this lead to loss of users funds.

case B: creating incorrect messageData which later used in the l2 bridge contract `main.sw` to get the message data value and return incorrect data.

in both cases user face lose of funds and creating incorrect message data.

## Impact Details

not setting the assetIssuerID inside the initialize function might lead to lose of funds or return incorrect message data.

## References

its highly recommended to set the assetIssuerID inside the Initialize function when the fuelERC20Gateway get deployed.

## Proof of concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: BSD 3-Clause License

pragma solidity ^0.8.15;

import "forge-std/Test.sol";
import "./interfaces/IPortalMessage.sol";
import "./interfaces/IERC20.sol";

// run this test in this repo : https://github.com/SunWeb3Sec/defilabs  run forge test --contracts src/test/test.sol -vvvv

contract ContractTest is Test {
    event LogBytes(bytes data);

    address public alice;
    address public bob;

    bytes32 public assetIssuerID;

    FuelMessagePortalV3 fuelPortal =
        FuelMessagePortalV3(0x01855B78C1f8868DE70e84507ec735983bf262dA);

    address usdt = 0xdAC17F958D2ee523a2206206994597C13D831ec7;

    enum MessageType {
        DEPOSIT_TO_ADDRESS,
        DEPOSIT_TO_CONTRACT,
        DEPOSIT_WITH_DATA,
        METADATA
    }

    function setUp() public {
        vm.createSelectFork("sepolia");
        alice = vm.addr(1);
        bob = vm.addr(2);
        deal(address(alice), 1 ether);
    }

    function testExploitHere() public {
        uint256 l2MintedAmount = 100 * 10 ** 9;

        bytes memory depositMessage1 = abi.encodePacked(
            assetIssuerID, // this will be 0000
            uint256(MessageType.DEPOSIT_TO_ADDRESS),
            bytes32(uint256(uint160(usdt))),
            uint256(0), // token_id = 0 for all erc20 deposits
            bytes32(uint256(uint160(msg.sender))),
            alice,
            l2MintedAmount,
            uint256(18)
        );

        emit LogBytes(depositMessage1);

        bytes32 id = bytes32(uint256(uint160(alice))); // any address because the issuerID address is unknown currently which we believe its l2 bridge address

        setAssetIssuerID(id);

        bytes memory depositMessage2 = abi.encodePacked(
            assetIssuerID, // this will be 0000
            uint256(MessageType.DEPOSIT_TO_ADDRESS),
            bytes32(uint256(uint160(usdt))),
            uint256(0), // token_id = 0 for all erc20 deposits
            bytes32(uint256(uint160(msg.sender))),
            alice,
            l2MintedAmount,
            uint256(18)
        );

        emit LogBytes(depositMessage2);
    }

    function setAssetIssuerID(bytes32 id) public {
        assetIssuerID = id;
    }
}


```
