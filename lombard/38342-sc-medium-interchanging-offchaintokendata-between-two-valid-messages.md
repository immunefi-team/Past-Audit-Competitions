# #38342 \[SC-Medium] Interchanging \`offchainTokenData\` between two valid messages

**Submitted on Dec 31st 2024 at 13:14:22 UTC by @security for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38342
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/bridge/adapters/TokenPool.sol>
* **Impacts:**
  * Protocol insolvency
  * Temporary freezing of funds for at least 30 days
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol (not lower than $1K))

## Description

## Brief/Intro

A vulnerability exists in the bridge message processing flow where `offchainTokenData` is not strictly tied to the corresponding bridged message. This allows a malicious actor to interchange `offchainTokenData` between two valid messages (e.g., using `offchainTokenData` from message1 for message2), leading to unintended behaviour in the token release or minting process.

## Vulnerability Details

During the bridging, the hash of required payload is attached to the transferred data. <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/TokenPool.sol#L44>

On the destination chain, the `OffRamp` contract in Chainlink handles message delivery by invoking either the `manuallyExecute` or `execute` function. <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L274> <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L331>

This triggers a series of internal calls to validate and process the message. The full flow of message delivery on the destination is as follows:

```
OffRamp::manuallyExecute/execute ==> OffRamp::_batchExecute ==> OffRamp::_executeSingleReport ==> OffRamp::_trialExecute ==> OffRamp::executeSingleMessage ==> OffRamp::_releaseOrMintTokens ==> OffRamp::_releaseOrMintSingleToken ==> TokenPool::releaseOrMint ==> CLAdapter::initiateWithdrawal ==> Bridge::receivePayload/authNotary/withdraw
```

<https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L274> <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L331> <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L345> <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L367> <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L540> <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L562> <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L744> <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L637> <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/TokenPool.sol#L57> <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/CLAdapter.sol#L200> <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/Bridge.sol#L178> <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/Bridge.sol#L220> <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/Bridge.sol#L263>

Both of these functions `manuallyExecute` or `execute` will end in verification phase to be sure that the bridged message is valid: <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L389-L424>

The important thing is that the verification is done on `report.messages`, while the `report` has also another important piece of data `offchainTokenData`. <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L389-L424>

```solidity
  struct ExecutionReport {
    uint64 sourceChainSelector; // Source chain selector for which the report is submitted.
    Any2EVMRampMessage[] messages;
    // Contains a bytes array for each message, each inner bytes array contains bytes per transferred token.
    bytes[][] offchainTokenData;
    bytes32[] proofs;
    uint256 proofFlagBits;
  }
```

<https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/libraries/Internal.sol#L68> <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L368>

This data `offchainTokenData` carries off-chain data to process the release or mint in the `TokenPool`. As you see, when `releaseOrMint` is called in the `TokenPool`, the `offchainTokenData` is forwarded as a field in the struct `ReleaseOrMintInV1`. <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L664-L683>

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

<https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/TokenPool.sol#L57>

```solidity
  struct ReleaseOrMintInV1 {
    bytes originalSender; //          The original sender of the tx on the source chain.
    uint64 remoteChainSelector; // ─╮ The chain ID of the source chain.
    address receiver; // ───────────╯ The recipient of the tokens on the destination chain.
    uint256 amount; //                The amount of tokens to release or mint, denominated in the source token's decimals.
    address localToken; //            The address on this chain of the token to release or mint.
    /// @dev WARNING: sourcePoolAddress should be checked prior to any processing of funds. Make sure it matches the
    /// expected pool address for the given remoteChainSelector.
    bytes sourcePoolAddress; //       The address of the source pool, abi encoded in the case of EVM chains.
    bytes sourcePoolData; //          The data received from the source pool to process the release or mint.
    /// @dev WARNING: offchainTokenData is untrusted data.
    bytes offchainTokenData; //       The offchain data to process the release or mint.
  }
```

<https://github.com/smartcontractkit/chainlink/blob/498b0b8579ad52a8c394fe3cbf55d3a86a8e29a0/contracts/src/v0.8/ccip/libraries/Pool.sol#L48>

Please note the comment: "offchainTokenData is untrusted data", showing that this data is not verified through Chainlink verification. So, it is the responsibility of the receiver protocol to validate it.

Its validation is done in the adapter through help of authority notarization by calling `authNotary`. <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/TokenPool.sol#L62>

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

<https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/CLAdapter.sol#L200-L212>

### Issue

The issue is that the `offchainTokenData` is not enforced to be related to the bridged message. In other words, suppose two valid messages are bridged and two corresponding `offchainTokenData` are generated and validated by the authority. On the destination chain, if `offchainTokenData` related to the **first** message is attached to the **second** message, and `offchainTokenData` related to the **second** message is attached to the **first** message, both of these messages will be delivered and validated successfully. The key issue is that when the **first** message is processed on the destination chain, the `offchainTokenData` associated with the **second** message is forwarded to the `TokenPool` and subsequently processed on the Bridge. As a result, processing the first message ends up minting LBTC associated with the second message, while processing the second message mints LBTC associated with the first message.

Please note that, the action of swapping `offchainTokenData` can be done easily by calling `OffRamp::manuallyExecute`. In other words, one can call this function with the following parameters:

```solidity
  struct ExecutionReport {
    uint64 sourceChainSelector;
    Any2EVMRampMessage[] messages; // the first message data
    bytes[][] offchainTokenData; // the data related to the second message
    bytes32[] proofs; // valid proof related to the first message
    uint256 proofFlagBits;
  }
```

So that, the first message would be validated on Chainlink with provided proof. So, the status of the first message would be set as successful on Chainlink.

```solidity
      _setExecutionState(sourceChainSelector, message.header.sequenceNumber, Internal.MessageExecutionState.IN_PROGRESS);
      (Internal.MessageExecutionState newState, bytes memory returnData) =
        _trialExecute(message, offchainTokenData, tokenGasOverrides);
      _setExecutionState(sourceChainSelector, message.header.sequenceNumber, newState);
```

<https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L499>

But, since `offchainTokenData` associated the second message is forwarded to `TokenPool`, the intended amount of LBTC will be minted to the receiver address set in the `offchainTokenData` associated with the second message.

It means that swapping `offchainTokenData` does not change the intended amount sent to the receiver, but it changes the protocol procedure significantly in handling the messages, impacting on the user’s intended actions.

For better understand, please consider the following example:

Suppose Alice calls `brdige::deposit` to bridge 1 LBTC to chainX. Bob calls `brdige::deposit` to bridge 100 LBTC to chainX. The bridge is using `CLAdapter` to handle the bridging mechanism.

* Alice deposits 1 LBTC to be bridged
* Bob deposits 100 LBTC to be bridged

The payload generated for Alice and Bob would be:

* Alice's payload:

```solidity
        bytes memory payloadAlice = abi.encodeWithSelector(
            Actions.DEPOSIT_BRIDGE_ACTION,
            bytes32(block.chainid),
            bytes32(uint256(uint160(address(this)))),
            chainX,
            config.bridgeContract,
            Alice,
            1 LBTC,
            $.crossChainOperationsNonce++
        );
```

* Bob's payload:

```solidity
        bytes memory payloadBob = abi.encodeWithSelector(
            Actions.DEPOSIT_BRIDGE_ACTION,
            bytes32(block.chainid),
            bytes32(uint256(uint160(address(this)))),
            chainX,
            config.bridgeContract,
            Bob,
            100 LBTC,
            $.crossChainOperationsNonce++
        );
```

On chainX, Alice calls the function `OffRamp::manuallyExecute` with the following parameters:

* `reports: [ ExecutionReport{ sourceChainSelector: // source chain messages: // the message related to Bob offchainTokenData: // the valid off-chain data related to Alice's message proofs: // the valid proof for Bob's message so that Chainlink can verify that Bob's message is delivered and processed securely proofFlagBits: // related to the provided proof for the Bob's message } ]`
* `gasLimitOverrides`: // enough gas for executing the message delivery

Then, in the function `OffRamp::_executeSingleReport`, the Bob's message will be verified with the provided proof, so that Chainlink ensures that the bridging is done securely. <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L424>

Then, the function `TokenPool::releaseOrMint` is called with the forwarded `offchainTokenData` related to Alice's message. <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L676> <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/TokenPool.sol#L64>

Since the provided `offchainTokenData` is valid, it will pass all the checks during the calls to functions `Bridge::receivePayload`, `Bridge::authNotary`, and `Bridge::withdraw`. Finally, 1 LBTC would be minted to Alice on chainX, and Alice's message would be flagged as `withdrawn`, so that it can not be retired on chainX. <https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/Bridge.sol#L312-L314>

On `OffRamp`, since the execution was successful, it sets the status of Bob's message as successful, so that it can not be later retried. <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L499>

The final status would be:

* Alice receives her 1 LBTC on chainX successfully.
* `offchainTokenData` associated with the Alice's message is consumed on the `Bridge`.
* Alice's message is not consumed on `OffRamp`.
* `offchainTokenData` associated with the Bob's message is not consumed on the `Bridge`.
* Bob's message is consumed on `OffRamp`.
* Bob has not received his 100 LBTC.

Bob does not lose his 100 LBTC, but if `OffRamp::manuallyExecute` or `OffRamp::execute` are invoked with Bob's message and `offchainTokenData` associated with Bob's message as parameters, its execution would be skipped with the event `SkippedAlreadyExecutedMessage` as Bob's proof and message is already consumed in Chainlink by Alice. <https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/ccip/offRamp/OffRamp.sol#L437-L448>

The root cause of this issue is that the `offchainTokenData` is not enforced to be associated with the bridged message. The following check is missing:

```diff
    function releaseOrMint(
        Pool.ReleaseOrMintInV1 calldata releaseOrMintIn
    ) external virtual override returns (Pool.ReleaseOrMintOutV1 memory) {
        _validateReleaseOrMint(releaseOrMintIn);

        uint64 amount = adapter.initiateWithdrawal(
            releaseOrMintIn.remoteChainSelector,
            releaseOrMintIn.offchainTokenData,
+           releaseOrMintIn.sourcePoolData
        );

        emit Minted(msg.sender, releaseOrMintIn.receiver, uint256(amount));

        return Pool.ReleaseOrMintOutV1({destinationAmount: uint256(amount)});
    }
```

<https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/TokenPool.sol#L57>

```diff
    function initiateWithdrawal(
        uint64 remoteSelector,
        bytes calldata offChainData,
+       bytes calldata sourcePoolData
    ) external onlyTokenPool returns (uint64) {
        (bytes memory payload, bytes memory proof) = abi.decode(
            offChainData,
            (bytes, bytes)
        );

+       require(keccak256(sourcePoolData) == keccak256(abi.encode(sha256(payload))), "offchain data is not related to the bridged message");

        _receive(getChain[remoteSelector], payload);
        bridge.authNotary(payload, proof);
        return bridge.withdraw(payload);
    }
```

<https://github.com/lombard-finance/evm-smart-contracts/blob/edd557006050ee5b847fa1cc67c1c4e19079437e/contracts/bridge/adapters/TokenPool.sol#L57>

## Impact Details

Alice has consumed Bob's proof and message in Chainlink while supplying `offchainTokenData` associated with her own message. As a result, Alice successfully receives her intended amount on the destination chain. However, Bob is now unable to use his own proof and message in Chainlink, as they have already been consumed by Alice. To recover his amounts, Bob would need to use Alice's proof and message in Chainlink while providing `offchainTokenData` associated with his own message. This attack disrupts the intended functionality, particularly if Bob's transaction involved using CCIP to transfer tokens and interact with a contract on the destination chain (calling `ccipReceive`), preventing him from executing the intended operation.

* Protocol Insolvency
* Temporary freezing of funds for at least 30 days
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol (not lower than $1K))

## References

It is worth noting that a similar issue was previously identified in an older audit conducted by Veridise on 17.12.2024, titled `4.1.5 V-CSC-VUL-005: Attacker can DoS CCIP messages that include LBTC transfers due to missing offchainTokenData validation`. This issue was resolved in commit `080220c`. However, due to extensive refactoring in the current protocol version, a similar vulnerability has re-emerged, and the risk is available.

## Proof of Concept

## PoC

Running the test `offchain data not related to the payload` shows that `offchainTokenData` is not enforced to be associated with the payload (i.e. the bridged message). Thus, it is possible to use `offchainTokenData` for an unrelated message, to consume the message in `OffRamp`.

```javascript
import {
    LBTCMock,
    Bascule,
    Consortium,
    Bridge,
    MockCCIPRouter,
    MockRMN,
    LombardTokenPool,
    CLAdapter,
    EndpointV2Mock,
} from '../typechain-types';
import {
    takeSnapshot,
    SnapshotRestorer,
} from '@nomicfoundation/hardhat-toolbox/network-helpers';
import {
    getSignersWithPrivateKeys,
    deployContract,
    CHAIN_ID,
    getPayloadForAction,
    NEW_VALSET,
    DEPOSIT_BRIDGE_ACTION,
    encode,
    signDepositBridgePayload,
    Signer,
} from './helpers';
import { ethers } from 'hardhat';
import { expect } from 'chai';
import { bridge } from '../typechain-types/contracts';
import { ZeroAddress } from 'ethers';
import { randomBytes } from 'crypto';

const aChainSelector = 1;
const bChainSelector = 2;

describe('Bridge', function () {
    let deployer: Signer,
        signer1: Signer,
        signer2: Signer,
        signer3: Signer,
        Alice: Signer,
        Bob: Signer,
        treasurySource: Signer,
        treasuryDestination: Signer,
        reporter: Signer,
        admin: Signer,
        pauser: Signer;
    let lbtcSource: LBTCMock;
    let lbtcDestination: LBTCMock;
    let consortium: Consortium;
    let bascule: Bascule;
    let bridgeSource: Bridge;
    let bridgeDestination: Bridge;
    let snapshot: SnapshotRestorer;
    const absoluteFee = 100n;

    before(async function () {
        [
            deployer,
            signer1,
            signer2,
            signer3,
            Alice,
            Bob,
            treasurySource,
            treasuryDestination,
            admin,
            pauser,
            reporter,
        ] = await getSignersWithPrivateKeys();

        // for both chains
        consortium = await deployContract<Consortium>('Consortium', [
            deployer.address,
        ]);
        await consortium.setInitialValidatorSet(
            getPayloadForAction([1, [signer1.publicKey], [1], 1, 1], NEW_VALSET)
        );

        // chain 1
        lbtcSource = await deployContract<LBTCMock>('LBTCMock', [
            await consortium.getAddress(),
            100,
            treasurySource.address,
            deployer.address,
        ]);
        bridgeSource = await deployContract<Bridge>('Bridge', [
            await lbtcSource.getAddress(),
            treasurySource.address,
            deployer.address,
        ]);
        bascule = await deployContract<Bascule>(
            'Bascule',
            [
                admin.address,
                pauser.address,
                reporter.address,
                await lbtcSource.getAddress(),
                100,
            ],
            false
        );

        // chain 2
        lbtcDestination = await deployContract<LBTCMock>('LBTCMock', [
            await consortium.getAddress(),
            100,
            treasuryDestination.address,
            deployer.address,
        ]);
        bridgeDestination = await deployContract<Bridge>('Bridge', [
            await lbtcDestination.getAddress(),
            treasuryDestination.address,
            deployer.address,
        ]);

        await lbtcSource.addMinter(await bridgeSource.getAddress());
        await lbtcDestination.addMinter(await bridgeDestination.getAddress());

        await bridgeSource.changeConsortium(await consortium.getAddress());
        await bridgeDestination.changeConsortium(await consortium.getAddress());

        // set rate limits
        const oo = {
            chainId: CHAIN_ID,
            limit: 1_0000_0000n, // 1 LBTC
            window: 100,
        };
        await bridgeSource.addDestination(
            CHAIN_ID,
            encode(['address'], [await bridgeDestination.getAddress()]),
            1000, // 10%
            0,
            ethers.ZeroAddress,
            true
        );
        await bridgeDestination.addDestination(
            CHAIN_ID,
            encode(['address'], [await bridgeSource.getAddress()]),
            0, // 0%
            absoluteFee,
            ethers.ZeroAddress,
            true
        );
        await bridgeSource.setRateLimits([oo], [oo]);
        await bridgeDestination.setRateLimits([oo], [oo]);

        snapshot = await takeSnapshot();
    });

    afterEach(async function () {
        await snapshot.restore();
    });

    describe('Actions/Flows', function () {
        const AMOUNT = 1_0000_0000n; // 1 LBTC

        beforeEach(async function () {
            await lbtcSource.mintTo(signer1.address, AMOUNT);
        });

        describe('With Chainlink Adapter', function () {
            let CCIPRouter: MockCCIPRouter,
                CCIPRMN: MockRMN,
                aTokenPool: LombardTokenPool,
                bTokenPool: LombardTokenPool,
                aCLAdapter: CLAdapter,
                bCLAdapter: CLAdapter;
            const aCCIPFee = 1_0000_0000n; // 1 gwei
            const bCCIPFee = 10_0000_0000n; // 10 gwei

            beforeEach(async function () {
                // configure CCIP
                CCIPRouter = await deployContract<MockCCIPRouter>(
                    'MockCCIPRouter',
                    [], // [aChainSelector, bChainSelector],
                    false
                );
                await CCIPRouter.setFee(aCCIPFee);

                CCIPRMN = await deployContract<MockRMN>('MockRMN', [], false);

                aCLAdapter = await deployContract<CLAdapter>(
                    'CLAdapter',
                    [
                        await bridgeSource.getAddress(),
                        300_000,
                        //
                        await CCIPRouter.getAddress(),
                        [], // no allowlist
                        await CCIPRMN.getAddress(), // will do work of rmn as well,
                    ],
                    false
                );

                aTokenPool = await ethers.getContractAt(
                    'LombardTokenPool',
                    await aCLAdapter.tokenPool()
                );
                await aTokenPool.acceptOwnership();
                await aCLAdapter.setRemoteChainSelector(
                    CHAIN_ID,
                    bChainSelector
                );

                bCLAdapter = await deployContract<CLAdapter>(
                    'CLAdapter',
                    [
                        await bridgeDestination.getAddress(),
                        300_000,
                        //
                        await CCIPRouter.getAddress(),
                        [], // no allowlist
                        await CCIPRMN.getAddress(), // will do work of rmn as well
                    ],
                    false
                );
                bTokenPool = await ethers.getContractAt(
                    'LombardTokenPool',
                    await bCLAdapter.tokenPool()
                );
                await bTokenPool.acceptOwnership();
                await bCLAdapter.setRemoteChainSelector(
                    CHAIN_ID,
                    aChainSelector
                );

                /// configure bridges
                await bridgeSource.changeAdapter(
                    CHAIN_ID,
                    await aCLAdapter.getAddress()
                );
                await bridgeDestination.changeAdapter(
                    CHAIN_ID,
                    await bCLAdapter.getAddress()
                );

                /// set token pools
                await aTokenPool.applyChainUpdates([
                    {
                        remoteChainSelector: bChainSelector,
                        allowed: true,
                        remotePoolAddress: await bTokenPool.getAddress(),
                        remoteTokenAddress: await lbtcDestination.getAddress(),
                        inboundRateLimiterConfig: {
                            isEnabled: false,
                            rate: 0,
                            capacity: 0,
                        },
                        outboundRateLimiterConfig: {
                            isEnabled: false,
                            rate: 0,
                            capacity: 0,
                        },
                    },
                ]);

                await bTokenPool.applyChainUpdates([
                    {
                        remoteChainSelector: aChainSelector,
                        allowed: true,
                        remotePoolAddress: await aTokenPool.getAddress(),
                        remoteTokenAddress: await lbtcSource.getAddress(),
                        inboundRateLimiterConfig: {
                            isEnabled: false,
                            rate: 0,
                            capacity: 0,
                        },
                        outboundRateLimiterConfig: {
                            isEnabled: false,
                            rate: 0,
                            capacity: 0,
                        },
                    },
                ]);

                await aTokenPool.setRemotePool(
                    bChainSelector,
                    ethers.zeroPadValue(await bTokenPool.getAddress(), 32)
                );
                await bTokenPool.setRemotePool(
                    aChainSelector,
                    ethers.zeroPadValue(await aTokenPool.getAddress(), 32)
                );
            });

            it('offchain data not related to the payload', async function () {
                let AliceAmount = 1;
                let BobAmount = 100;

                const data = await signDepositBridgePayload(
                    [signer1],
                    [true],
                    CHAIN_ID,
                    await bridgeSource.getAddress(),
                    CHAIN_ID,
                    await bridgeDestination.getAddress(),
                    Alice.address,
                    AliceAmount
                );

                const AliceOffChainData = encode(['bytes', 'bytes'], [data.payload, data.proof]);

                // using Bob message data but with Alice Offchain data

                const snapshot = await takeSnapshot();
                await bTokenPool.releaseOrMint({
                    originalSender: ethers.zeroPadValue(await bridgeSource.getAddress(), 32),
                    remoteChainSelector: aChainSelector,
                    receiver: Bob.address,
                    amount: BobAmount,
                    localToken: await lbtcDestination.getAddress(),
                    sourcePoolAddress: ethers.zeroPadValue(await aTokenPool.getAddress(), 32),
                    sourcePoolData: "0x11223344",
                    offchainTokenData: AliceOffChainData
                })
                expect(await lbtcDestination.balanceOf(Alice.address)).to.eq(1);
                await snapshot.restore();

                // using Alice message data but with Alice Offchain data

                await bTokenPool.releaseOrMint({
                    originalSender: ethers.zeroPadValue(await bridgeSource.getAddress(), 32),
                    remoteChainSelector: aChainSelector,
                    receiver: Alice.address,
                    amount: AliceAmount,
                    localToken: await lbtcDestination.getAddress(),
                    sourcePoolAddress: ethers.zeroPadValue(await aTokenPool.getAddress(), 32),
                    sourcePoolData: "0xaabbccdd",
                    offchainTokenData: AliceOffChainData
                })
                expect(await lbtcDestination.balanceOf(Alice.address)).to.eq(1);

            });

        });
    });
});

```
