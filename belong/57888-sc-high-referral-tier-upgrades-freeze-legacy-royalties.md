# 57888 sc high referral tier upgrades freeze legacy royalties

**Submitted on Oct 29th 2025 at 11:36:11 UTC by @spongebob for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57888
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/RoyaltiesReceiverV2.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

Referral tiers silently mutate historic splits. `shares()` recomputes BPS from the factory every time funds are released:

<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/RoyaltiesReceiverV2.sol#L107-L131>

Each new collection a creator mints through the same referral code bumps `usedCode` in `_setReferralUser`, which can jump the referral percentage for all of that creator’s legacy receivers:

<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/Factory.sol#L254>

<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/extensions/ReferralSystemV2.sol#L136-L154>

After a first payout (e.g., 100 sale → referral paid 6% on tier 1), a later tier upgrade might raise the referral share to 30%. `_pendingPayment` then demands the additional 24% that was never retained (<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/RoyaltiesReceiverV2.sol#L227-L237>). Because only 0 wei remain, `safeTransferETH` reverts. Even after a new 10 wei sale the pending amount is 27 wei, so every release attempt keeps reverting.

As a result, royalties stay stuck until enough fresh volume accumulates to cover the historical shortfall, which may never happen and may result in permanent freeze of unclaimed royalties.

## Impact

This falls under **High – Permanent freezing of unclaimed royalties**. The referral-tier mutation can leave legacy receivers with unsatisfied payout deltas that keep reverting every release call until enough new sales occur, so overdue royalties can stay locked indefinitely.

Likelihood is high. Every new `Factory.produce` call run through the same referral code unconditionally increments `usedCode[creator][code]` (<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/extensions/ReferralSystemV2.sol#L136-L162>), so an active creator who keeps crediting the same referrer will inevitably climb tiers.

If that creator has already released royalties for an older collection at a lower tier, the next referral release on that legacy receiver will recompute shares from the upgraded tier (<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/extensions/ReferralSystemV2.sol#L107-L131>) and immediately demand the historical shortfall (<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/RoyaltiesReceiverV2.sol#L227-L237>).

Because the receiver only holds whatever fresh sales have come in, any shortfall larger than the current balance causes `safeTransferETH` to revert, so in practice the first release attempt after a tier bump fails unless the contract happens to be sitting on enough new volume to cover the back-pay. That combination (multiple collections per creator plus routine royalty releases) is an expected, common workflow, so the freeze condition is not an edge case.

## Recommendation

Snapshot the split when the receiver is created and pay out against that immutable schedule. The simplest fix is to record the referral tier (or the full creator/platform/referral BPS breakdown) inside `RoyaltiesReceiverV2.initialize` and have `shares()` read from that local copy instead of re-querying the factory. That way past payouts never get repriced when `usedCode` advances.

## Proof of Concept

The test funds the receiver with 100 wei, settles at the 6% tier, bumps `usedCode` to 2, adds another 10 wei, verifies the referral share jump to 600 BPS, and asserts that `releaseAll` now reverts with `ETHTransferFailed` because the legacy balance can’t cover the back-pay.

Run this diff:

```diff
diff --git a/test/v2/platform/royalties-freeze.poc.test.ts b/test/v2/platform/royalties-freeze.poc.test.ts
new file mode 100644
index 0000000..efb95b8
--- /dev/null
+++ b/test/v2/platform/royalties-freeze.poc.test.ts
@@ -0,0 +1,116 @@
+import { expect } from 'chai';
+import { ethers } from 'hardhat';
+import EthCrypto from 'eth-crypto';
+import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
+import {
+  AccessToken,
+  CreditToken,
+  Factory,
+  MockTransferValidatorV2,
+  RoyaltiesReceiverV2,
+  SignatureVerifier,
+  VestingWalletExtended,
+} from '../../../typechain-types';
+import {
+  deployAccessToken,
+  deployAccessTokenImplementation,
+  deployCreditTokenImplementation,
+  deployFactory,
+  deployRoyaltiesReceiverV2Implementation,
+  deployVestingWalletImplementation,
+} from '../../../helpers/deployFixtures';
+import { deploySignatureVerifier } from '../../../helpers/deployLibraries';
+import { deployMockTransferValidatorV2 } from '../../../helpers/deployMockFixtures';
+
+describe('RoyaltiesReceiverV2 :: Referral tier escalation PoC', () => {
+  const NATIVE_CURRENCY_ADDRESS = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE';
+  const REFERRAL_PERCENTAGES = [0, 600, 3000, 3000, 3000]; // 6% → 30% jump on second use.
+  const safeTransferInterface = new ethers.utils.Interface(['error ETHTransferFailed()']);
+
+  async function fixture() {
+    const [platform, creator, referrer] = await ethers.getSigners();
+    const signer = EthCrypto.createIdentity();
+
+    const signatureVerifier: SignatureVerifier = await deploySignatureVerifier();
+    const validator: MockTransferValidatorV2 = await deployMockTransferValidatorV2();
+    const accessTokenImpl: AccessToken = await deployAccessTokenImplementation(signatureVerifier.address);
+    const royaltiesReceiverImpl: RoyaltiesReceiverV2 = await deployRoyaltiesReceiverV2Implementation();
+    const creditTokenImpl: CreditToken = await deployCreditTokenImplementation();
+    const vestingWalletImpl: VestingWalletExtended = await deployVestingWalletImplementation();
+
+    const implementations: Factory.ImplementationsStruct = {
+      accessToken: accessTokenImpl.address,
+      creditToken: creditTokenImpl.address,
+      royaltiesReceiver: royaltiesReceiverImpl.address,
+      vestingWallet: vestingWalletImpl.address,
+    };
+
+    const factory: Factory = await deployFactory(
+      platform.address,
+      signer.address,
+      signatureVerifier.address,
+      validator.address,
+      implementations,
+      undefined,
+      100,
+      NATIVE_CURRENCY_ADDRESS,
+      10,
+      {
+        amountToCreator: 8000,
+        amountToPlatform: 2000,
+      },
+      REFERRAL_PERCENTAGES,
+    );
+
+    const referralCode = await factory.connect(referrer).callStatic.createReferralCode();
+    await factory.connect(referrer).createReferralCode();
+
+    return {
+      factory,
+      creator,
+      platform,
+      referrer,
+      signer,
+      referralCode,
+    };
+  }
+
+  it('locks legacy royalties after referral tier bump', async () => {
+    const { factory, creator, platform, referrer, signer, referralCode } = await loadFixture(fixture);
+
+    const { royaltiesReceiver } = await deployAccessToken(
+      { name: 'First', symbol: 'FIRST', uri: 'first-uri' },
+      0,
+      0,
+      signer,
+      creator,
+      factory,
+      referralCode,
+    );
+
+    const firstSale = ethers.BigNumber.from(100);
+    await platform.sendTransaction({ to: royaltiesReceiver.address, value: firstSale });
+    await royaltiesReceiver.releaseAll(NATIVE_CURRENCY_ADDRESS);
+
+    await deployAccessToken(
+      { name: 'Second', symbol: 'SECOND', uri: 'second-uri' },
+      0,
+      0,
+      signer,
+      creator,
+      factory,
+      referralCode,
+    );
+
+    const secondSale = ethers.BigNumber.from(10);
+    await platform.sendTransaction({ to: royaltiesReceiver.address, value: secondSale });
+
+    expect(await factory.usedCode(creator.address, referralCode)).to.equal(2);
+    expect(await royaltiesReceiver.shares(referrer.address)).to.equal(600); // 30% of platform share (2000 BPS * 0.3)
+
+    await expect(royaltiesReceiver.releaseAll(NATIVE_CURRENCY_ADDRESS)).to.be.revertedWithCustomError(
+      { interface: safeTransferInterface } as any,
+      'ETHTransferFailed',
+    );
+  });
+});
```
