# 57298 sc critical state sync omission in staking transfers forces transferred slong holders into penalized emergency exits

* **Submitted on:** Oct 25th 2025 at 04:23:25 UTC by @InquisitorScythe for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)
* **Report ID:** #57298
* **Report Type:** Smart Contract
* **Severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impact category:** Griefing (no profit motive required; harms users/protocol)

## Description

### Brief / Intro

The staking vault mints and allows free transfer of sLONG (ERC4626-style shares), but it does not synchronize the per-account stake ledger on transfers. A recipient who obtains sLONG via ERC20 transfer receives shares (ERC20 balance) but has no corresponding unlocked stake entries in the contract's per-account ledger. Such recipients cannot pass the normal withdrawal checks and are forced to use the emergency withdrawal path, which burns shares and applies a permanent penalty routed to the treasury.

### Root cause

The per-account stake ledger:

* Is declared as `mapping(address staker => Stake[] times) public stakes;`
* Is appended to only on deposits (via `_deposit`)
* Is never reconciled or updated on ERC20 transfers (no transfer hooks or overrides)

Thus, transferred-in sLONG holders have token balances but zero stake entries, so they cannot consume "unlocked" stakes via the normal withdrawal flow and are blocked into emergency withdrawals with penalties.

## Vulnerability Details (selected snippets)

* File: `contracts/v2/periphery/Staking.sol`
* `stakes` declaration:

```solidity
	/// @notice User stake entries stored as arrays per staker.
	/// @dev Public getter: `stakes(user, i)` → `(shares, timestamp)`.
	mapping(address staker => Stake[] times) public stakes;
```

* `_deposit` — locks freshly minted shares on deposit:

```solidity
	function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
		super._deposit(by, to, assets, shares);
		// lock freshly minted shares
		stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
	}
```

* `_withdraw` — requires consumption of unlocked stake entries:

```solidity
	function _withdraw(address by, address to, address _owner, uint256 assets, uint256 shares) internal override {
		_consumeUnlockedSharesOrRevert(_owner, shares);
		super._withdraw(by, to, _owner, assets, shares);
	}
```

* `_consumeUnlockedSharesOrRevert` — iterates per-account stake entries and reverts if not enough unlocked shares:

```solidity
	function _consumeUnlockedSharesOrRevert(address staker, uint256 need) internal {
		Stake[] storage userStakes = stakes[staker];
		uint256 _min = minStakePeriod;
		uint256 nowTs = block.timestamp;
		uint256 remaining = need;

		for (uint256 i; i < userStakes.length && remaining > 0;) {
			Stake memory s = userStakes[i];
			if (nowTs >= s.timestamp + _min) {
				uint256 take = s.shares <= remaining ? s.shares : remaining;
				if (take == s.shares) {
					// full consume → swap and pop
					remaining -= take;
					userStakes[i] = userStakes[userStakes.length - 1];
					userStakes.pop();
					// don't ++i: a new element is now at index i
				} else {
					// partial consume
					userStakes[i].shares = s.shares - take;
					remaining = 0;
					unchecked {
						++i;
					}
				}
			} else {
				unchecked {
					++i;
				}
			}
		}

		if (remaining != 0) revert MinStakePeriodNotMet();
	}
```

* `_emergencyWithdraw` — burns shares regardless of lock status and applies penalty:

```solidity
	function _emergencyWithdraw(address by, address to, address _owner, uint256 assets, uint256 shares) internal {
		require(shares > 0, SharesEqZero());

		uint256 penalty = FixedPointMathLib.fullMulDiv(assets, penaltyPercentage, SCALING_FACTOR);
		uint256 payout;
		unchecked {
			payout = assets - penalty;
		}

		if (by != _owner) _spendAllowance(_owner, by, shares);

		_removeAnySharesFor(_owner, shares);
		_burn(_owner, shares);

		LONG.safeTransfer(to, payout);
		LONG.safeTransfer(treasury, penalty);

		emit EmergencyWithdraw(by, to, _owner, assets, shares);
		// also emit standard ERC4626 Withdraw for indexers/analytics
		emit Withdraw(by, to, _owner, assets, shares);
	}
```

Note: line ranges cited in the report correspond to the repository's current main branch at the time of reporting.

## Impact (concise)

* Transfers only move ERC20 balances and do not update `stakes[...]`. Recipients have shares but no stake entries.
* Attempting a standard `withdraw`/`redeem` calls `_consumeUnlockedSharesOrRevert` and reverts with `MinStakePeriodNotMet` because the recipient has no unlocked stake entries.
* Recipients are forced to use `emergencyWithdraw`/`emergencyRedeem`, which burns shares, applies the configured penalty (default 10%), and sends the penalty to `treasury`.
* Monetary impact: transferred-in holders permanently lose a fraction of their assets proportional to `penaltyPercentage`. This can be arbitrarily large depending on transferred amount.
* Attack surface: any integration or flow that transfers sLONG balances (e.g., auto-stake, rewards routing, third-party transfers) can inadvertently subject recipients to penalized exits.

## References

* Vulnerable file in related repo: <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/periphery/Staking.sol>

## Proof of Concept

Below is the Hardhat test PoC used to reproduce the issue: depositor stakes LONG, transfers the minted sLONG to a recipient, and the recipient cannot perform normal redeem — only emergency redeem succeeds (with penalty).

File: `test/v2/platform/staking-transfer-poc.test.ts`

```typescript
import { ethers } from 'hardhat';
import { expect } from 'chai';
import { loadFixture, time } from '@nomicfoundation/hardhat-network-helpers';
import { deployLONG, deployStaking } from '../../../helpers/deployFixtures';
import { LONG, Staking } from '../../../typechain-types';

/**
 * ============================================================================
 * POC: Transferred sLONG Holders Are Locked Out Of Standard Withdrawals
 * ============================================================================
 *
 * This Hardhat test recreates the scenario where a depositor stakes LONG,
 * transfers the resulting sLONG shares to a recipient, and that recipient can
 * no longer perform a normal (non-emergency) withdrawal. Even after the lock
 * period elapses, `_consumeUnlockedSharesOrRevert` finds no stake entries for
 * the recipient and reverts with `MinStakePeriodNotMet`, forcing the user into
 * the emergency withdrawal path which burns shares and charges the penalty.
 */
describe('Staking - Transferred Holder Forced Emergency PoC', function () {
  async function fixture() {
    const [owner, treasury, depositor, recipient] = await ethers.getSigners();

    // Deploy proxy LONG token and staking vault.
    const long: LONG = await deployLONG(depositor.address, owner.address, owner.address);
    const staking: Staking = await deployStaking(owner.address, treasury.address, long.address);

    // Shorten the lock period to simplify the reproduction.
    await staking.connect(owner).setMinStakePeriod(1);

    const depositAmount = ethers.utils.parseEther('100');

    // Depositor stakes LONG and receives sLONG along with a stake entry.
    await long.connect(depositor).approve(staking.address, depositAmount);
    await staking.connect(depositor).deposit(depositAmount, depositor.address);
    const mintedShares = await staking.balanceOf(depositor.address);
    const recordedStake = await staking.stakes(depositor.address, 0);

    console.log(`\n[Setup] Depositor minted ${ethers.utils.formatEther(mintedShares)} sLONG shares`);
    console.log(`[Setup] Stake entry recorded with timestamp ${recordedStake.timestamp.toString()}`);

    return {
      owner,
      treasury,
      depositor,
      recipient,
      long,
      staking,
      depositAmount,
      mintedShares,
    };
  }

  it('proves transferred holders must use penalized emergency exits', async () => {
    const { treasury, depositor, recipient, long, staking, depositAmount, mintedShares } = await loadFixture(fixture);

    // Depositor transfers the entire sLONG balance to the recipient.
    await staking.connect(depositor).transfer(recipient.address, mintedShares);

    await expect(staking.callStatic.stakes(recipient.address, 0)).to.be.reverted;
    console.log('[Transfer] Recipient received shares but has zero stake entries');

    // Advance time beyond the (shortened) minimum stake period.
    await time.increase(2);
    await time.advanceBlock();

    // Recipient attempts a standard redeem and gets blocked by MinStakePeriodNotMet.
    await expect(
      staking.connect(recipient).redeem(mintedShares, recipient.address, recipient.address),
    ).to.be.revertedWithCustomError(staking, 'MinStakePeriodNotMet');
    console.log('[Redeem Attempt] Normal redeem reverts with MinStakePeriodNotMet');

    // Emergency redeem succeeds but pays penalty to treasury.
    const scalingFactor = await staking.SCALING_FACTOR();
    const penaltyPercentage = await staking.penaltyPercentage();
    const expectedAssets = await staking.convertToAssets(mintedShares);
    const expectedPenalty = expectedAssets.mul(penaltyPercentage).div(scalingFactor);

    const recipientBefore = await long.balanceOf(recipient.address);
    const treasuryBefore = await long.balanceOf(treasury.address);

    const emergencyTx = await staking
      .connect(recipient)
      .emergencyRedeem(mintedShares, recipient.address, recipient.address);
    await emergencyTx.wait();

    const recipientAfter = await long.balanceOf(recipient.address);
    const treasuryAfter = await long.balanceOf(treasury.address);

    const recipientGain = recipientAfter.sub(recipientBefore);
    const treasuryGain = treasuryAfter.sub(treasuryBefore);

    console.log(
      `[Emergency] Recipient forced to emergencyRedeem, receiving ${ethers.utils.formatEther(recipientGain)} LONG`,
    );
    console.log(`[Emergency] Treasury collects penalty of ${ethers.utils.formatEther(treasuryGain)} LONG`);

    expect(recipientGain).to.eq(expectedAssets.sub(expectedPenalty));
    expect(treasuryGain).to.eq(expectedPenalty);
    expect(await staking.balanceOf(recipient.address)).to.eq(0);
  });
});
```

Example run:

```
yarn hardhat test test/v2/platform/staking-transfer-poc.test.ts
...
[Setup] Depositor minted 100.0 sLONG shares
[Setup] Stake entry recorded with timestamp 1761366088
[Transfer] Recipient received shares but has zero stake entries
[Redeem Attempt] Normal redeem reverts with MinStakePeriodNotMet
[Emergency] Recipient forced to emergencyRedeem, receiving 90.0 LONG
[Emergency] Treasury collects penalty of 10.0 LONG
✔ proves transferred holders must use penalized emergency exits
```

(Full console output and test artifacts omitted for brevity.)

***

If you want, I can:

* Suggest minimal code fixes or design approaches to keep per-account stake consistency on transfers (e.g., hooks updating `stakes` on transfer, or migrating stakes on transfer/delegation).
* Draft a patch/PR against the referenced repository demonstrating a fix.
