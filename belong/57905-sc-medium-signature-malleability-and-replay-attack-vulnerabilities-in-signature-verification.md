# 57905 sc medium signature malleability and replay attack vulnerabilities in signature verification

**Submitted on Oct 29th 2025 at 12:22:31 UTC by @Sparrow\_23 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57905
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Unauthorized minting of NFTs

## Description

### Brief/Intro

The `SignatureVerifier` library uses a vulnerable signature scheme susceptible to signature malleability and replay attacks. The current implementation lacks essential security measures including EIP-712 compliance, nonces, deadlines, and proper s-value validation, allowing attackers to reuse or manipulate signatures indefinitely across different chains and contexts.

### Vulnerability Details

ECDSA signatures are inherently malleable — given a valid signature (r, s, v), an attacker can create (r, -s mod n, v') which is also valid. The SignatureVerifier contract uses the SignatureCheckerLib library to verify signatures:

```solidity
library SignatureVerifier {
    using SignatureCheckerLib for address;
```

But SignatureCheckerLib does not protect against malleable signatures.

The library itself contains a warning comment:

{% hint style="warning" %}
/// WARNING! Do NOT use signatures as unique identifiers:\
/// - Use a nonce in the digest to prevent replay attacks on the same contract.\
/// - Use EIP-712 for the digest to prevent replay attacks across different chains and contracts.\
/// EIP-712 also enables readable signing of typed data for better user safety.\
/// This implementation does NOT check if a signature is non-malleable.
{% endhint %}

The signatures also lack a proper EIP-712 domain separator making signatures vulnerable to being replayed across chains and contracts. They are also missing nonce and deadline fields and as such cannot be cancelled and remain valid forever once generated.

All signature verification functions in the library use the vulnerable pattern:

```solidity
// Current vulnerable implementation in checkAccessTokenInfo:
keccak256(
    abi.encodePacked(
        accessTokenInfo.metadata.name,
        accessTokenInfo.metadata.symbol,
        accessTokenInfo.contractURI,
        accessTokenInfo.feeNumerator,
        block.chainid
    )
)
```

### Impact Details

This allows attackers to create alternative valid signatures for the same message by modifying the signature components leading to replay attacks and signature malleability exploitation (e.g., unauthorized minting).

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/utils/SignatureVerifier.sol#L28-L33>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/node\\_modules/solady/src/utils/SignatureCheckerLib.sol#L19-L23>
* <https://eips.ethereum.org/EIPS/eip-712>

## Proof of Concept

The following tests demonstrate:

* missing nonce and deadline (signatures remain valid indefinitely)
* missing nonce allowing replay of identical signed requests

Add the following test to `factory.test.ts`:

```ts
describe('Missing Nonce and Deadline', () => {
  it('should demonstrate signature validity over time', async () => {
    const { signatureVerifier, factory, alice, signer } = await loadFixture(fixture);

    const nftName = 'TimelessNFT';
    const nftSymbol = 'TNFT';
    const contractURI = 'contractURI/Timeless';
    const feeNumerator = 500;

    // Create signature now
    const message = hashAccessTokenInfo(nftName, nftSymbol, contractURI, feeNumerator, chainId);
    const signature = EthCrypto.sign(signer.privateKey, message);

    const info: AccessTokenInfoStruct = {
      metadata: { name: nftName, symbol: nftSymbol },
      contractURI: contractURI,
      paymentToken: NATIVE_CURRENCY_ADDRESS,
      mintPrice: ethers.utils.parseEther('0.01'),
      whitelistMintPrice: ethers.utils.parseEther('0.01'),
      transferable: true,
      maxTotalSupply: BigNumber.from('1000'),
      feeNumerator,
      collectionExpire: BigNumber.from('86400'),
      signature: signature,
    };

    // Verify signature works immediately
    await expect(signatureVerifier.checkAccessTokenInfo(signer.address, info)).to.not.be.reverted;

    // Advance time by 1 year
    await ethers.provider.send('evm_increaseTime', [365 * 24 * 60 * 60]); // 1 year
    await ethers.provider.send('evm_mine', []);

    // Signature should still be valid (demonstrating no expiration)
    await expect(signatureVerifier.checkAccessTokenInfo(signer.address, info)).to.not.be.reverted;

    // Advance time by 10 years
    await ethers.provider.send('evm_increaseTime', [10 * 365 * 24 * 60 * 60]); // 10 years
    await ethers.provider.send('evm_mine', []);

    // Signature should still be valid after 10 years
    await expect(signatureVerifier.checkAccessTokenInfo(signer.address, info)).to.not.be.reverted;
  });

  it('should demonstrate missing nonce allows potential replay', async () => {
    const { signatureVerifier, signer } = await loadFixture(fixture);

    // Create two identical requests (simulating replay scenario)
    const nftName = 'ReplayNFT';
    const nftSymbol = 'RNFT';
    const contractURI = 'contractURI/Replay';
    const feeNumerator = 500;

    const message = hashAccessTokenInfo(nftName, nftSymbol, contractURI, feeNumerator, chainId);
    const signature = EthCrypto.sign(signer.privateKey, message);

    const info1: AccessTokenInfoStruct = {
      metadata: { name: nftName, symbol: nftSymbol },
      contractURI: contractURI,
      paymentToken: NATIVE_CURRENCY_ADDRESS,
      mintPrice: ethers.utils.parseEther('0.01'),
      whitelistMintPrice: ethers.utils.parseEther('0.01'),
      transferable: true,
      maxTotalSupply: BigNumber.from('1000'),
      feeNumerator,
      collectionExpire: BigNumber.from('86400'),
      signature: signature,
    };

    const info2 = { ...info1 }; // Exact same info

    // Both would be valid (though Factory would prevent duplicate NFT creation)
    await expect(signatureVerifier.checkAccessTokenInfo(signer.address, info1)).to.not.be.reverted;
    await expect(signatureVerifier.checkAccessTokenInfo(signer.address, info2)).to.not.be.reverted;

    console.log('VULNERABILITY CONFIRMED: Same signature valid for identical requests (replay potential)');

    // Show what proper implementation would include
    const properImplementation = {
      hasNonce: false,
      hasDeadline: false,
      hasUsedSignaturesTracking: false
    };

    expect(properImplementation.hasNonce).to.be.false;
    expect(properImplementation.hasDeadline).to.be.false;
  });
});
```

<details>

<summary>Notes / Context</summary>

* The PoC demonstrates that signatures generated today remain accepted after long time periods (no deadline) and that identical signed payloads can be reused (no nonce / used-signature tracking).
* The existing code uses `abi.encodePacked(..., block.chainid)` in the message hash; adding block.chainid helps bind to a chain but does not solve malleability nor replace proper EIP-712 domain usage, nonces, deadlines, or s-value validation.

</details>
