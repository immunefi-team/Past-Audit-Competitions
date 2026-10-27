# 57942 sc critical transferred slong shares are permanently unredeemable due to missing stake entry creation

**Submitted on Oct 29th 2025 at 14:39:42 UTC by @iehnnkta for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57942
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief/Intro

The Staking vault implements time-locks through deposit-specific "Stake" entries that track individual staking positions. However, the contract's unlock mechanism only recognizes shares originating from direct deposits, not shares received via ERC20 transfers. This architectural flaw breaks the fungibility principle of ERC4626 vault shares, rendering transferred tokens either permanently locked or accessible only through penalized emergency withdrawal functions.

## Vulnerability Details

### Root Cause

The vulnerability stems from a fundamental mismatch between the token transfer mechanism and the unlock tracking system:

* Deposit flow: The `_deposit` function creates a new `Stake({shares, timestamp})` entry in `stakes[to]`, establishing a time-locked position.
* Withdrawal flow: The `_withdraw` function invokes `_consumeUnlockedSharesOrRevert(owner, shares)`, which iterates through `stakes[owner]` and requires sufficient unlocked entries to fulfill the withdrawal request.
* Transfer flow: Standard ERC20 transfers update balances but do not create, modify, or migrate Stake entries. Consequently, the recipient's transferred shares exist in their balance without corresponding unlock records.

The following issues share this common root cause but manifest from different user perspectives:

BUG‑1: Transferred Shares Are Unredeemable

* Scenario: Alice transfers 50 sLONG to Bob, who subsequently deposits an additional 100 LONG (receiving 100 sLONG).
* Result: After the lock period expires, Bob's stake records total only 100 shares (from his direct deposit). Any attempt to `redeem(150)` fails with `MinStakePeriodNotMet` because the system cannot locate unlock records for the 50 transferred shares. These shares remain permanently irredeemable through standard mechanisms and can only be recovered via emergency withdrawal functions, which impose financial penalties.

BUG‑2: Partial Transfers Create Withdrawal Denial-of-Service

* Scenario: Alice deposits 100 sLONG, transfers 50 shares to another address, then attempts to withdraw her remaining 50 shares before the unlock period.
* Result: The withdrawal reverts with `MinStakePeriodNotMet` despite Alice only requesting half her balance. This occurs because her single Stake entry (100 shares) remains fully locked, and the algorithm cannot partially consume or split locked positions. The remaining 50 shares become inaccessible through standard withdrawal paths until the original lock expires, demonstrating fragile interdependency between balance and stake accounting.

## Impact Details

* Capital Lockup: Users receiving sLONG via transfer cannot access those funds through standard redemption flows; recovery requires emergency functions with mandatory financial penalties.
* Operational Disruption: Legitimate staking positions become partially or fully inaccessible via standard withdrawal mechanisms, forcing users into suboptimal emergency procedures.
* Fungibility Violation: Breaks the ERC4626 share fungibility assumption, where identical shares should have identical properties and redemption rights regardless of acquisition method.

## References

<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/periphery/Staking.sol#L258-L290>

## Proof of Concept

The following test cases (staking.test.sol) demonstrate the issue:

```javascript
describe('Vulnerability Demonstrations', () => {
    it('ISSUE-1: Recipients cannot withdraw/redeem sLONG tokens received via transfer', async () => {
      const { staking, long, admin, user1: sender, user2: recipient } = await loadFixture(fixture);

      const hundredTokens = ethers.utils.parseEther('100');
      const fiftyTokens = ethers.utils.parseEther('50');

      // Sender deposits 100 LONG and receives 100 sLONG
      await long.connect(admin).transfer(sender.address, hundredTokens);
      await long.connect(sender).approve(staking.address, hundredTokens);
      await staking.connect(sender).deposit(hundredTokens, sender.address);

      // Sender transfers 50 sLONG to Recipient
      await staking.connect(sender).transfer(recipient.address, fiftyTokens);

      // Recipient deposits 100 LONG and receives 100 sLONG
      await long.connect(admin).transfer(recipient.address, hundredTokens);
      await long.connect(recipient).approve(staking.address, hundredTokens);
      await staking.connect(recipient).deposit(hundredTokens, recipient.address);

      // Unlock all shares
      await staking.connect(admin).setMinStakePeriod(1);

      // Recipient attempts to redeem 150 sLONG (100 deposited + 50 transferred) -> fails; only deposited shares are redeemable
      await expect(
        staking.connect(recipient).redeem(hundredTokens.add(fiftyTokens), recipient.address, recipient.address),
      ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');

      // Recipient can successfully redeem only the 100 sLONG from direct deposit
      const projectedAssets = await staking.previewRedeem(hundredTokens);
      const transaction = await staking.connect(recipient).redeem(hundredTokens, recipient.address, recipient.address);
      await expect(transaction)
        .to.emit(staking, 'Withdraw')
        .withArgs(recipient.address, recipient.address, recipient.address, projectedAssets, hundredTokens);

      // The remaining 50 transferred sLONG are now locked indefinitely (cannot be redeemed/withdrawn)
      expect(await staking.balanceOf(recipient.address)).to.eq(fiftyTokens);
      await expect(staking.connect(recipient).redeem(fiftyTokens, recipient.address, recipient.address)).to.be.revertedWithCustomError(
        staking,
        'MinStakePeriodNotMet',
      );
    });

    it('ISSUE-2: transferring out partial sLONG prevents withdrawal/redemption of remaining balance', async () => {
      const { staking, long, admin, user1: depositor, user2: receiver } = await loadFixture(fixture);

      const hundredTokens = ethers.utils.parseEther('100');
      const fiftyTokens = ethers.utils.parseEther('50');

      // Depositor stakes 100 LONG and receives 100 sLONG
      await long.connect(admin).transfer(depositor.address, hundredTokens);
      await long.connect(depositor).approve(staking.address, hundredTokens);
      await staking.connect(depositor).deposit(hundredTokens, depositor.address);

      // Depositor transfers 50 sLONG to Receiver
      await staking.connect(depositor).transfer(receiver.address, fiftyTokens);

      // Depositor tries to withdraw remaining 50 LONG via withdraw/redeem -> fails due to stake entry mismatch
      await expect(
        staking.connect(depositor).withdraw(fiftyTokens, depositor.address, depositor.address),
      ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');
      await expect(
        staking.connect(depositor).redeem(fiftyTokens, depositor.address, depositor.address),
      ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');
    });
  });
```

Observed test output when running the above:

```
     ✔ ISSUE-1: Recipients cannot withdraw/redeem sLONG tokens received via transfer (762ms)
     ✔ ISSUE-2: transferring out partial sLONG prevents withdrawal/redemption of remaining balance
  2 passing (785ms)
```
