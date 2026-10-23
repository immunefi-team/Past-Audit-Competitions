# 57885 sc high dynamic share drift in royaltiesreceiverv2

**Submitted on Oct 29th 2025 at 11:31:04 UTC by @komane007 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57885
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/extensions/ReferralSystemV2.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

RoyaltiesReceiverV2 recalculates royalty shares on every payout using live values from Factory (creator/platform split and referral tier). Its accounting (PaymentSplitter-style) assumes shares are fixed over time. When shares change after any past releases (e.g., referral tier auto-increases), the newly computed entitlements implicitly apply the new shares to all historical inflows, causing over-allocation. As a result, `release` / `releaseAll` can revert and royalties can be frozen until substantial new funds arrive—sometimes indefinitely.

## Vulnerability Details

* Dynamic shares are read at payout time:

```solidity
// contracts/v2/periphery/RoyaltiesReceiverV2.sol
function shares(address account) public view returns (uint256) {
    Factory.RoyaltiesParameters memory p = factory.royaltiesParameters();
    if (account == royaltiesReceivers.creator) {
        return p.amountToCreator;                   // dynamic creator share
    } else {
        uint256 platformShare = p.amountToPlatform; // dynamic platform share
        uint256 referralShare;
        if (royaltiesReceivers.referral != address(0)) {
            referralShare = factory.getReferralRate(
                royaltiesReceivers.creator, referralCode, p.amountToPlatform
            );                                      // dynamic referral share (tier-based)
            if (referralShare > 0) {
                unchecked { platformShare -= referralShare; }
            }
        }
        return account == royaltiesReceivers.platform
            ? platformShare
            : account == royaltiesReceivers.referral ? referralShare : 0;
    }
}
```

* PaymentSplitter-style accounting assumes static shares:

```solidity
// contracts/v2/periphery/RoyaltiesReceiverV2.sol
function _pendingPayment(bool isNativeRelease, address token, address account) private view returns (uint256) {
    Releases storage releases = isNativeRelease ? nativeReleases : erc20Releases[token];
    uint256 balance = isNativeRelease ? address(this).balance : token.balanceOf(address(this));
    uint256 payment = ((balance + releases.totalReleased) * shares(account)) / TOTAL_SHARES;
    if (payment <= releases.released[account]) return 0;
    return payment - releases.released[account];
}
```

* This formula is correct only if `shares(account)` has been constant since inception. If `shares(account)` increases later, the recomputed cumulative entitlement exceeds the funds actually received/retained by the contract, producing an oversized `payment`.
* Referral tiers auto-increase with normal usage (no admin needed):

```solidity
// contracts/v2/platform/Factory.sol
_setReferralUser(referralCode, msg.sender); // increments usage whenever a referral is used

// contracts/v2/platform/extensions/ReferralSystemV2.sol
function _setReferralUser(bytes32 hashedCode, address referralUser) internal {
    ...
    uint8 used = usedCode[referralUser][hashedCode];
    if (used < MAX_TIER_INDEX) {
        usedCode[referralUser][hashedCode] = used + 1; // auto tier bump
    }
}
```

* Therefore, shares can drift organically as the referral is exercised—no upgrade or admin action required.

### Simple scenario (numbers)

* T0: 100 ETH received. Shares = Creator 80%, Platform 20%, Referral 0%.
  * `releaseAll` pays 80/20. Balance = 0, totalReleased = 100.
* T1: Referral tier rises to 5% (Platform → 15%, Referral → 5%).
  * Referral pending = (0+100) × 5% − 0 = 5 ETH > balance(=0) → `release` reverts.
* T2: 10 ETH arrive. Creator pending = (10+100) × 80% − 80 = 8 ETH (ok), Referral pending = (10+100) × 5% − 0 = 5.5 ETH. If creator withdraws first, remaining balance may be < 5.5 ETH → referral withdrawal reverts; `releaseAll` can fully revert.

## Impact Details

* Permanent / long-lived freezing of royalties
  * When shares increase after earlier payouts, newly computed entitlements can exceed the contract balance, causing `release` / `releaseAll` to revert. Future small inflows may still be insufficient to cover the recomputed arrears, keeping royalties frozen.
* Over/under-payment drift
  * If shares increase, affected payees are over-allocated relative to the actual historical split; if shares decrease, others are underpaid. The design no longer conserves balances across time, breaking accounting accuracy.
* Severity (program scope mapping)
  * High/Critical: Permanent freezing of funds; asset accuracy/accounting drift; DoS of payout flows.

## References

* Dynamic share computation: `contracts/v2/periphery/RoyaltiesReceiverV2.sol` → `shares()`
* Pending payment formula assuming static shares: `contracts/v2/periphery/RoyaltiesReceiverV2.sol` → `_pendingPayment()`
* Auto-increment referral tiers: `contracts/v2/platform/extensions/ReferralSystemV2.sol` → `_setReferralUser()`; called from `contracts/v2/platform/Factory.sol` → `produce()`

## Proof of Concept

Add this test to `/home/jo/audit-comp-belong/test/v2/platform/factory.test.ts` and run:

LEDGER\_ADDRESS=0x0000000000000000000000000000000000000001 PK=0x0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef npx hardhat test test/v2/platform/factory.test.ts --grep "Security: royalties receiver drift"

```js
 describe('Security: royalties receiver drift', () => {
    it('freezes payouts when shares increase after prior releases', async () => {
      const { factory, owner, alice, signer } = await loadFixture(fixture);

      const nftName = 'Drift1';
      const nftSymbol = 'DR1';
      const contractURI = 'contractURI/drift1';
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

      // Deploy a collection with royalties receiver
      await factory.connect(alice).produce(info, ethers.constants.HashZero);
      const instance = await factory.nftInstanceInfo(nftName, nftSymbol);
      const rr: RoyaltiesReceiverV2 = await ethers.getContractAt('RoyaltiesReceiverV2', instance.royaltiesReceiver);
      const payees = await rr.royaltiesReceivers();

      // Fund the receiver and settle once at 80/20
      const deposit = ethers.utils.parseEther('1');
      await owner.sendTransaction({ to: rr.address, value: deposit });
      await rr.releaseAll(NATIVE_CURRENCY_ADDRESS);

      expect(await rr.totalReleased(NATIVE_CURRENCY_ADDRESS)).to.eq(deposit);
      const releasedCreator = await rr.released(NATIVE_CURRENCY_ADDRESS, payees.creator);
      const releasedPlatform = await rr.released(NATIVE_CURRENCY_ADDRESS, payees.platform);
      expect(releasedCreator).to.eq(deposit.mul(8000).div(10000));
      expect(releasedPlatform).to.eq(deposit.mul(2000).div(10000));

      // Increase creator share to 90/10 (dynamic drift)
      await factory.upgradeToV2({ amountToCreator: 9000, amountToPlatform: 1000 }, implementations);

      // No new funds present: next creator release tries to claim additional 10% of historical inflows → revert
      await expect(rr.release(NATIVE_CURRENCY_ADDRESS, payees.creator)).to.be.reverted;
    });

    it('freezes payouts after organic referral tier bump (no admin)', async () => {
      const { factory, owner, alice, charlie, signer } = await loadFixture(fixture);

      // Referrer creates a referral code (sets referrals[code].creator)
      await factory.connect(charlie).createReferralCode();
      const referralCode = await factory.getReferralCodeByCreator(charlie.address);

      // Deploy a collection with referral code; _setReferralUser bumps alice's used count to 1
      const nftName = 'DriftOrg1';
      const nftSymbol = 'DRO1';
      const contractURI = 'contractURI/drift-org-1';
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

      await factory.connect(alice).produce(info, referralCode);
      const instance = await factory.nftInstanceInfo(nftName, nftSymbol);
      const rr: RoyaltiesReceiverV2 = await ethers.getContractAt('RoyaltiesReceiverV2', instance.royaltiesReceiver);
      const payees = await rr.royaltiesReceivers();
      expect(payees.referral).to.eq(charlie.address);

      // Fund the receiver and settle once at used=1 → creator 80%, platform 10%, referral 10%
      const deposit = ethers.utils.parseEther('1');
      await owner.sendTransaction({ to: rr.address, value: deposit });
      await rr.releaseAll(NATIVE_CURRENCY_ADDRESS);

      expect(await rr.totalReleased(NATIVE_CURRENCY_ADDRESS)).to.eq(deposit);
      const releasedCreator = await rr.released(NATIVE_CURRENCY_ADDRESS, payees.creator);
      const releasedPlatform = await rr.released(NATIVE_CURRENCY_ADDRESS, payees.platform);
      const releasedReferral = await rr.released(NATIVE_CURRENCY_ADDRESS, payees.referral);
      expect(releasedCreator).to.eq(deposit.mul(8000).div(10000));
      expect(releasedPlatform).to.eq(deposit.mul(1000).div(10000));
      expect(releasedReferral).to.eq(deposit.mul(1000).div(10000));

      // Organically bump referral tier for alice by producing another collection with the same referral code
      const nftName2 = 'DriftOrg2';
      const nftSymbol2 = 'DRO2';
      const contractURI2 = 'contractURI/drift-org-2';
      const feeNumerator2 = 0; // no need to deploy another RR
      const message2 = hashAccessTokenInfo(nftName2, nftSymbol2, contractURI2, feeNumerator2, chainId);
      const signature2 = EthCrypto.sign(signer.privateKey, message2);
      const info2: AccessTokenInfoStruct = {
        metadata: { name: nftName2, symbol: nftSymbol2 },
        contractURI: contractURI2,
        paymentToken: NATIVE_CURRENCY_ADDRESS,
        mintPrice: price,
        whitelistMintPrice: price,
        transferable: true,
        maxTotalSupply: BigNumber.from('1000'),
        feeNumerator: feeNumerator2,
        collectionExpire: BigNumber.from('86400'),
        signature: signature2,
      };
      await factory.connect(alice).produce(info2, referralCode); // used bumps from 1 -> 2

      // With used=2, referral share decreases and platform share increases (to 14%). Balance is 0 → release for platform reverts.
      await expect(rr.release(NATIVE_CURRENCY_ADDRESS, payees.platform)).to.be.reverted;
    });
  });
```

***

If you want, I can:

* Produce suggested remediations (e.g., snapshotting fixed shares per release, storing historical totals per-account, or converting to per-inflow accounting), or
* Draft a minimal patch suggestion to make shares static for accounting purposes while preserving referral mechanics.
