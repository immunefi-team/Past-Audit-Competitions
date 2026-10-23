# #38634 \[SC-Medium] Insufficient validation on offchainTokenData in TokenPool.releaseOrMint allows CC

**Submitted on Jan 8th 2025 at 13:20:56 UTC by @nnez for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38634
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/bridge/adapters/TokenPool.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Permanent freezing of funds

## Description

## Vulnerabilty Details

### 2 factor cross-chain transfer

LBTC utilizes CCIP to perfrom a cross-chain token transfer. However, the protocol also adds its own security layer, namely, Consortium.\
Apart from cross-chain message being signed by Chainlink RMN, Validators of Consortium must also sign on the payload of the source chain.

This process is called 2-factor LBTC bridging.

User can initiate a cross-chain transfer by invoking `deposit` method on **Bridge** contract or directly call `ccipSend` on **CCIP Router** contract in case of needing to perform additional cross-chain action.

When request is made, the source chain bridge emits `DepositToBridge` with a payload, Validators then sign on this payload.

Once the payload is signed, the cross-chain transfer can be executed through **CCIP OffRamp** contract with payload and proof (signature) attached in `offchainTokenData` variable.

To put it simply, for every CCIP cross-chain message, there is a corresponding signed payload from Consortium. (Note that signed payload is not part of CCIP message, so it is not blessed by RMN)

That is, the legitimacy of cross-chain transaction is verified by CCIP and the legitimacy of token transfer is verified by Consortium.

Hence, the name: 2-factor.

Cross-chain transfer flow on destination is as follows:

1. Consortium signed the payload and CCIP's RMN blesses the message
2. CCIP's DON or user executes the message and attach signatures from Consortium in `offchainTokenData` through **OffRamp** contract
3. **OffRamp** contract then calls LBTC `TokenPool.releaseOrMint` on destination chain
4. **TokenPool** then calls to **CLAdapter** which takes care of the proof verificaiton and minting of token to receiver (specified in payload)
5. After tokens are minted, if the receiver's address is CCIP compatible, **OffRamp** proceed to call `ccipReceive` method on receiver address.

The simplified flow:

```
OffRamp.manuallyExecute(transmit)  
     -> TokenPool.releaseOrMint  
          -> CLAdapter.initiateWithdrawal  
               -> Bridge.receivePayload  
               -> Bridge.authNotary (verify signatures)  
                    -> Consortium.checkProof  
               -> Bridge.witdraw (mint token to payload.receiver)  
     -> receiver.ccipReceive  
```

### Vulnerability

Let's say Alice is requesting two cross-chain transfers of LBTC and let's name each transfer A and B respectively.

Transfer A: amount=1, receiver=Bob\
Transfer B: amount=2, receiver=Bob

Ideally, Transfer A's payload should only work with Transfer A's CCIP message. However, since `offchainTokenData` (containing the payload and proof) is not part of CCIP's message, it is possible to use Transfer B's payload with Transfer A's CCIP message and pass CCIP verification.

See the comment explaning why `offchainTokenData` is user-control and untrusted: <https://github.com/smartcontractkit/ccip/blob/ccip-develop/contracts/src/v0.8/ccip/pools/USDC/USDCTokenPool.sol#L127-L155>

Also, if we look further into `TokenPool.releaseOrMint` implementation:

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

*TokenPool.\_validateReleaseOrMint*: <https://github.com/smartcontractkit/ccip/blob/ccip-develop/contracts/src/v0.8/ccip/pools/TokenPool.sol#L222-L235>

We can see that it relies solely on `offchainTokenData` to verify and mint tokens to receiver. It doesn't check that CCIP message is corresponding with input `offChainTokenData`.

This means that we can execute CCIP message of Transfer A while using payload of Transfer B successfully, minting tokens as per Transfer B payload.

Fortunately, **CCIP OffRamp** verifies receiver's balance (using receiver from CCIP's message) after every call to `TokenPool.releaseOrMint`. That is, the balance of receiver must increase to the same amount return by **TokenPool**. This creates a constraint where one can only switch payload if the receiver of CCIP's message is the same as the payload.

Initially, this might not appear problematic due to the constraint of having the same receiver. After all, once all messages are executed, the receiver's total balance amount is unchanged.

However, there exists some use cases that this switch of payload could cause a problem. Consider the following scenario:

1. Alice performs a cross-chain swap of 1 LBTC using SwapRouter as the receiver with specific swap data.
2. Eve observes Alice’s transaction and initiates a zero-data, zero-gas cross-chain transfer of 0.001 LBTC to SwapRouter. (This forces CCIP to not call a SwapRouter)
3. Due to network congestion, both transactions are delayed by the DON.
4. Eve manually executes Eve's CCIP message using Alice's payload.
5. The TokenPool mints Alice’s 1 LBTC to SwapRouter, but Eve’s zero-data and zero-gas message ensures the SwapRouter is not invoked.
6. Alice’s 1 LBTC remains stuck in the SwapRouter.
7. When Alice attempts to execute her own message, it fails because the payload has already been withdrawn.
8. Even though Alice manages to execute their message with Eve's payload, the transaction would either fail from insufficient token or invalid data.

Alice loses 1 LBTC\
Eve pays 0.001 LBTC

## Impact

This scenario, while potentially occurring in specific use cases of LBTC cross-chain transfers, can result in the loss or freezing of funds. Although the impact may be limited to certain situations, this is a feasibility issue which should not affect the severity of its consequence (loss of funds -> Critical).

Ref: <https://immunefisupport.zendesk.com/hc/en-us/articles/16913132495377-Feasibility-Limitation-Standards>

## Recommended Mitigations

Add a check that `descPoolData` (part of CCIP's message) is corresponding with input `offchainTokenData.payload`.

## References

1. Conditions where OffRamp skip calling ccipReceiver: <https://github.com/smartcontractkit/ccip/blob/ccip-develop/contracts/src/v0.8/ccip/offRamp/EVM2EVMOffRamp.sol#L518-L530>

## Proof of Concept

## Proof-of-Concept

Due to limitation in test suite, I couldn't construct a complete flow of PoC.\
Instead, as the root cause is insufficient validation on `offchainTokenData`, I wrote a PoC to demonstrate that `TokenPool.releaseOrMint` can be invoked with CCIP message with a mismatched payload (same receiver).

### Steps

1. Add the following test case in `test/Bridge.ts`, after **should route message** case.

```javascript
it('Should demonstrate that TokenPool.releaseOrMint does not verify specfic source message and its corresponding payload', async function(){
    /**
    The following test attempts to demonstrate that TokenPool.releaseOrMint can be invoked with a CCIP message containing a payload that is inconsistent with the intended transfer (but same receiver)  
    Specifically, the test constructs a releaseOrMint argument that includes a CCIP message designed to transfer 1e8 LBTC from signer3 to signer2. However, the actual payload within this CCIP message is crafted to transfer only 1,000 LBTC from signer1 to signer2, creating a mismatch between the intended transfer and the data carried within the CCIP message.
    */
    
    // beforeEach set this adapter to incorrect address (aCLAdapter)
    await bridgeDestination.changeAdapter(
        CHAIN_ID,
        await bCLAdapter.getAddress()
    );

    // retrieve payload and its signature  
    // a payload to transfer 1_000 of LBTC to signer2 from signer1
    let data = await signDepositBridgePayload(
        [signer1],
        [true],
        CHAIN_ID,
        await bridgeSource.getAddress(),
        CHAIN_ID,
        await bridgeDestination.getAddress(),
        signer2.address,
        1_000n
    );

    // Simulating a CCIP message to transfer 1e8 of LBTC to signer2 from signer3
    // Note that we put above payload in offchainTokenData
    const coder = new AbiCoder();
    const args = {
        originalSender: ethers.zeroPadValue(signer3.address, 32),
        remoteChainSelector: aChainSelector,
        receiver: signer2.address,
        amount: 1_0000_0000n,
        localToken: await lbtcDestination.getAddress(),
        sourcePoolAddress: ethers.zeroPadValue(await aTokenPool.getAddress(), 32),
        sourcePoolData: data.payload,
        offchainTokenData: coder.encode(["bytes", "bytes"], [data.payload, data.proof])
    }

    // Asserts that initial balance of signer2 is zero
    expect(await lbtcDestination.balanceOf(signer2.address)).to.equal(0);

    // MockCCIPRouter already allows arbitrary call on tokenPool
    await bTokenPool.connect(signer1).releaseOrMint(args);
    
    // Asserts that initial balance of signer2 is 1000n (equal to that of crafted payload)
    expect(await lbtcDestination.balanceOf(signer2.address)).to.equal(1000n);
});
```

2. Run `yarn hardhat test test/Bridge.ts --grep "Should demonstrate that"`
3. Observe that the test passes, indicate that 1\_000 LBTC is minted to signer2 despite CCIP message specifying 1e8.\
   That is, `TokenPool.releaseOrMint` can be invoked with CCIP message with mismatched payload and that it blindly trust `offchainTokenData`.
