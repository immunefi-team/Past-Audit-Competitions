# #38363 \[SC-Medium] LBTC cross-chain transfer can be DOSed

**Submitted on Jan 1st 2025 at 15:16:53 UTC by @jasonxiale for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38363
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/bridge/adapters/TokenPool.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 30 days

## Description

## Brief/Intro

While transferring LBTC cross chains using CCIP, `LombardTokenPool.releaseOrMint` will be called on the dest chain. While `EVM2EVMOffRamp._releaseOrMintToken` calls `LombardTokenPool.releaseOrMint` to mint LBTC for the recipient, the function `LombardTokenPool.releaseOrMint`'s parameter `releaseOrMintIn` includes the [token's information](https://github.com/smartcontractkit/ccip/blob/c279cbb4ab57436b9c59c9321492e25f0aa30e80/contracts/src/v0.8/ccip/libraries/Pool.sol#L37-L49)

```solidity
 37   struct ReleaseOrMintInV1 {
 38     bytes originalSender; //          The original sender of the tx on the source chain
 39     uint64 remoteChainSelector; // ─╮ The chain ID of the source chain
 40     address receiver; // ───────────╯ The recipient of the tokens on the destination chain.
 41     uint256 amount; //                The amount of tokens to release or mint, denominated in the source token's decimals
 42     address localToken; //            The address on this chain of the token to release or mint
 43     /// @dev WARNING: sourcePoolAddress should be checked prior to any processing of funds. Make sure it matches the
 44     /// expected pool address for the given remoteChainSelector.
 45     bytes sourcePoolAddress; //       The address of the source pool, abi encoded in the case of EVM chains
 46     bytes sourcePoolData; //          The data received from the source pool to process the release or mint
 47     /// @dev WARNING: offchainTokenData is untrusted data.
 48     bytes offchainTokenData; //       The offchain data to process the release or mint
 49   }
```

As above code shows, there are a few import things:

1. `receiver` address and `amount` are included directly in the [parameter](https://github.com/smartcontractkit/ccip/blob/c279cbb4ab57436b9c59c9321492e25f0aa30e80/contracts/src/v0.8/ccip/libraries/Pool.sol#L40-L41)

> 40 address receiver; // ───────────╯ The recipient of the tokens on the destination chain.

> 41 uint256 amount; // The amount of tokens to release or mint, denominated in the source token's decimals

2. `offchainTokenData` is untrusted data, which means it isn't verified by CCIP's [i\_commitStore](https://github.com/smartcontractkit/ccip/blob/c279cbb4ab57436b9c59c9321492e25f0aa30e80/contracts/src/v0.8/ccip/offRamp/EVM2EVMOffRamp.sol#L316)

> 47 /// @dev WARNING: offchainTokenData is untrusted data.

> 48 bytes offchainTokenData; // The offchain data to process the release or mint

And in `LombardTokenPool.releaseOrMint`, the function doesn't verify recipient info stored in `releaseOrMintIn.offchainTokenData` matches `releaseOrMintIn.receiver` and `releaseOrMintIn.amount`.

**So an attacker can abuse the mismatch to block other users' cross-chain LBTC transfer.**

## Vulnerability Details

As shown in [TokenPool.releaseOrMint](https://github.com/lombard-finance/evm-smart-contracts/blob/e46c8c9f1e94e422e6082d01bc97d331b98c9d07/contracts/bridge/adapters/TokenPool.sol#L57-L70), the function doesn't use the recipient info included in `releaseOrMintIn`, and also doesn't verify if the `releaseOrMintIn.offchainTokenData` matches the recipient info stored in `releaseOrMintIn`

```
 57     function releaseOrMint(
 58         Pool.ReleaseOrMintInV1 calldata releaseOrMintIn
 59     ) external virtual override returns (Pool.ReleaseOrMintOutV1 memory) {
 60         _validateReleaseOrMint(releaseOrMintIn);
 61 

>>>>>> offchainTokenData is untrusted, and used directly by the function
 62         uint64 amount = adapter.initiateWithdrawal(
 63             releaseOrMintIn.remoteChainSelector,
 64             releaseOrMintIn.offchainTokenData
 65         );
 66 
 67         emit Minted(msg.sender, releaseOrMintIn.receiver, uint256(amount));
 68 
 69         return Pool.ReleaseOrMintOutV1({destinationAmount: uint256(amount)});
 70     }
```

Then in [Bridge.authNotary](https://github.com/lombard-finance/evm-smart-contracts/blob/e46c8c9f1e94e422e6082d01bc97d331b98c9d07/contracts/bridge/Bridge.sol#L220-L258), `$.consortium.checkProof` can only make sure the `payload` is signed by correct validators.

And also in [EVM2EVMOffRamp.\_execute](https://github.com/smartcontractkit/ccip/blob/c279cbb4ab57436b9c59c9321492e25f0aa30e80/contracts/src/v0.8/ccip/offRamp/EVM2EVMOffRamp.sol#L294-L448), while cross-chain messages are verified, the `report.offchainTokenData` isn't verified.

```solidity
294   function _execute(Internal.ExecutionReport memory report, GasLimitOverride[] memory manualExecGasOverrides) internal {
...

>>>>>> as the following code shows, report.offchainTokenData isn't verified by `i_commitStore`
299     if (numMsgs != report.offchainTokenData.length) revert UnexpectedTokenData();
300 
301     bytes32[] memory hashedLeaves = new bytes32[](numMsgs);
302 
303     for (uint256 i = 0; i < numMsgs; ++i) {
304       Internal.EVM2EVMMessage memory message = report.messages[i];
305       // We do this hash here instead of in _verifyMessages to avoid two separate loops
306       // over the same data, which increases gas cost
307       hashedLeaves[i] = Internal._hash(message, i_metadataHash);
308       // For EVM2EVM offramps, the messageID is the leaf hash.
309       // Asserting that this is true ensures we don't accidentally commit and then execute
310       // a message with an unexpected hash.
311       if (hashedLeaves[i] != message.messageId) revert InvalidMessageId();
312     }
313     bool manualExecution = manualExecGasOverrides.length != 0;
314 
315     // SECURITY CRITICAL CHECK
316     uint256 timestampCommitted = ICommitStore(i_commitStore).verify(hashedLeaves, report.proofs, report.proofFlagBits);
317     if (timestampCommitted == 0) revert RootNotCommitted();
...

448   }
```

## Impact Details

Please consider the following case:

1. Alice send a cross-chain tx to transfer 10e8 LBTC from ETH to BNB
2. Bob wants to block Alice's tx on BNB chain, so he send a cross-chain tx to transfer 1 unit LBTC.(Please note this step isn't necessary, Bob can use other's tx as well).
3. Both Alice and Bob's tx will be signed by Consortium's validators.
4. If everything goes well, [EVM2EVMOffRamp.manuallyExecute](https://github.com/smartcontractkit/ccip/blob/c279cbb4ab57436b9c59c9321492e25f0aa30e80/contracts/src/v0.8/ccip/offRamp/EVM2EVMOffRamp.sol#L234-L278) will be called with:

   ```
   1) EVM2EVMMessage for Alice(it'll be called `message_alice`) and Alice's payload(it'll be called `payload_alice`)
   2) EVM2EVMMessage for Bob(it'll be called `message_bob`)  and Bob's payloab(it'll be called `payload_bob`)
   ```

   And after the EVM2EVMMessage been executed, the message will be marked as `Internal.MessageExecutionState.SUCCESS` in [EVM2EVMOffRamp.sol#L411](https://github.com/smartcontractkit/ccip/blob/c279cbb4ab57436b9c59c9321492e25f0aa30e80/contracts/src/v0.8/ccip/offRamp/EVM2EVMOffRamp.sol#L411)
5. Because Bob wants to block Alice's tx on BNB chain, he will front-run the `EVM2EVMOffRamp.manuallyExecute` with **message\_alice plus payload\_bob** as parameters. In such case, because [TokenPool.releaseOrMint](https://github.com/lombard-finance/evm-smart-contracts/blob/e46c8c9f1e94e422e6082d01bc97d331b98c9d07/contracts/bridge/adapters/TokenPool.sol#L57-L70) handles LBTC transfer based on `releaseOrMintIn.offchainTokenData`, which means Bob's payload will be executed. **But because CCIP marks the executed message based on `message.sequenceNumber`, `message_alice` will be marked as executed**
6. When Alice executes her tx, because her `message.sequenceNumber` is marked as executed, her LBTC can't be transferred.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

Please put the following code in `test/Bridge.ts` and run

```bash
yarn hardhat test test/Bridge.ts 
yarn run v1.22.22
$ /in/evm-smart-contracts/node_modules/.bin/hardhat test test/Bridge.ts
duplicate definition - ZeroAddress()
duplicate definition - LengthMismatch()
duplicate definition - LengthMismatch()


  Bridge
    Actions/Flows
      ✔ Transfer Based On payload (145ms)
```

As the POC demostrates, the amount of token trasferred based on payload, instead of `releaseOrMintIn.amount`

```diff
diff --git a/test/Bridge.ts b/test/Bridge.ts
index 8b62c3d..24114bc 100644
--- a/test/Bridge.ts
+++ b/test/Bridge.ts
@@ -334,6 +334,93 @@ describe('Bridge', function () {
                 );
         });
 
+        it.only('Transfer Based On payload', async () => {
+            let amount = AMOUNT;
+            let fee = amount / 10n;
+
+            let amountWithoutFee = amount - fee;
+            let receiver = signer2.address;
+
+            let payload = getPayloadForAction(
+                [
+                    CHAIN_ID,
+                    encode(['address'], [await bridgeSource.getAddress()]),
+                    CHAIN_ID,
+                    encode(['address'], [await bridgeDestination.getAddress()]),
+                    encode(['address'], [receiver]),
+                    amountWithoutFee,
+                    ethers.AbiCoder.defaultAbiCoder().encode(['uint256'], [0]),
+                ],
+                DEPOSIT_BRIDGE_ACTION
+            );
+
+            await lbtcSource
+                .connect(signer1)
+                .approve(await bridgeSource.getAddress(), amount);
+            await expect(
+                bridgeSource
+                    .connect(signer1)
+                    .deposit(CHAIN_ID, encode(['address'], [receiver]), amount)
+            )
+                .to.emit(bridgeSource, 'DepositToBridge')
+                .withArgs(
+                    signer1.address,
+                    encode(['address'], [receiver]),
+                    ethers.sha256(payload),
+                    payload
+                );
+
+            expect(await lbtcSource.balanceOf(signer1.address)).to.be.equal(0);
+            expect(
+                await lbtcSource.balanceOf(treasurySource.address)
+            ).to.be.equal(fee);
+            expect((await lbtcSource.totalSupply()).toString()).to.be.equal(
+                fee
+            );
+
+            expect(
+                await lbtcDestination.balanceOf(signer2.address)
+            ).to.be.equal(0);
+            expect(await lbtcDestination.totalSupply()).to.be.equal(0);
+
+            const data1 = await signDepositBridgePayload(
+                [signer1],
+                [true],
+                CHAIN_ID,
+                await bridgeSource.getAddress(),
+                CHAIN_ID,
+                await bridgeDestination.getAddress(),
+                receiver,
+                amountWithoutFee
+            );
+
+            await expect(
+                bridgeDestination
+                    .connect(signer2)
+                    .authNotary(data1.payload, data1.proof)
+            )
+                .to.emit(bridgeDestination, 'PayloadNotarized')
+                .withArgs(receiver, ethers.sha256(data1.payload));
+
+            await expect(
+                bridgeDestination.connect(signer2).withdraw(data1.payload)
+            )
+                .to.emit(bridgeDestination, 'WithdrawFromBridge')
+                .withArgs(
+                    receiver,
+                    ethers.sha256(data1.payload),
+                    data1.payload,
+                    amountWithoutFee
+                );
+
+            expect(
+                (await lbtcDestination.totalSupply()).toString()
+            ).to.be.equal(amount - fee);
+            expect(
+                (await lbtcDestination.balanceOf(signer2.address)).toString()
+            ).to.be.equal(amountWithoutFee);
+
+        });
         describe('With failing rate limits', function () {
             it('should fail to deposit if rate limit is exceeded', async function () {
                 await lbtcSource.mintTo(signer1.address, 1);
```
