# 88597 sc high same boundary withdrawal mutates committed collateral and causes protocol insolvency

**Submitted on Aug 14th 2026 at 02:30:23 UTC by @Kopi for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88597
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

At an exact vault-period boundary, an allocator commitment can read the vault's pre-withdrawal collateral before an ordinary shareholder's withdrawal writes a checkpoint at the same timestamp. The withdrawal retroactively lowers the historical value later used to cap incident payouts, while the allocator retains the larger committed capacity. The PoC sells 110 units of cover backed by 100 vault units and a 10-unit first-loss buffer, but a valid 110-unit assessment receives only the 10-unit buffer while the shareholder later claims all 100 vault units, leaving the cover obligation insolvent.

## Vulnerability Details

### Root cause

`CoverOrderAllocator.commitAllocation` calculates commitment capacity using `vault.totalAssetsAt(vault.currentPeriodStart())` and stores the result. Settlement later trusts that stored commitment capacity.

`FirelightVault.withdraw` and `redeem`, however, checkpoint the reduced active-asset balance at the current timestamp. `Checkpoints` lookups are inclusive: a checkpoint whose key equals the query timestamp is returned. Therefore, the following sequence is possible at `T == currentPeriodStart()`:

1. `commitAllocation` reads the prior 100-asset checkpoint through `totalAssetsAt(T)` and stores capacity of 110 after including the 10-unit first-loss buffer.
2. A shareholder withdraws 100 assets later at the same timestamp `T`.
3. The withdrawal writes the zero-active-assets checkpoint at `T`, so subsequent `totalAssetsAt(T)` calls return zero.
4. Settlement still accepts the stored 110-unit commitment.
5. `FirelightVault.payout` independently reads `totalAssetsAt(T)` and uses the now-zero value as the outer vault-payment cap.

All transactions in one EVM block share a timestamp. Flare also permits a child block to have the same timestamp as its parent, so the vulnerable ordering can span consecutive accepted blocks. The shareholder requires no privileged role and withdraws only its own shares.

The source also supports a stronger zero-settlement-grace variant. `_settleCoverOrder` writes the matched status and `totalSettledCover` before `CoverNFT.safeMint` invokes the buyer's `onERC721Received`. A buyer contract that owns the vault shares can withdraw in that callback. When the callback returns, settlement completes with the snapshot reduced and the commitment already nonreplaceable because settlement has begun.

### Why existing checks do not prevent the issue

* Settlement validates the Merkle proof and stored commitment but does not recompute the collateral snapshot.
* Payout includes pending withdrawals only inside a cap derived from `totalAssetsAt(periodStart)`; once that outer cap is zero, pending-withdrawal accounting cannot restore the vault leg.
* Withdrawal-before-commit is safe because the reduced capacity is visible. The bug requires commit-before-withdraw at the same timestamp.
* Before any settlement, administrators can cancel or recommit. After settlement begins, `_replaceableCommitment` reverts with `SettlementsAlreadyStarted`.

### Recommended remediation

Freeze a vault-owned, set-once collateral exposure for each period when the first commitment consumes it, and require both commitment capacity and payout to use that same immutable value. Track initialization separately because zero is a valid exposure. Only the configured allocator contract should initialize the value. An alternative is to make these specific consumers use an exclusive pre-boundary lookup such as `T - 1`, after auditing first-period and zero-history behavior. Add regression tests for commit/withdraw orderings in one block and consecutive equal-timestamp blocks.

## Impact Details

The exact in-scope impact is **Protocol insolvency**. In the reproduced state:

* Vault collateral before the boundary withdrawal: `100` units.
* First-loss-buffer collateral: `10` units.
* Cover accepted and settled: `110` units.
* Valid allocation-bounded incident loss: `110` units.
* Buffer payment: `10` units.
* Vault payment: `0` units.
* Shareholder's later withdrawal claim: `100` units.

The protocol has issued 110 units of real cover but can satisfy only 10 units through the contractual payout path. The shareholder preserves the entire 100-unit vault leg. Depositing tokens later does not repair the historical zero cap, and ordinary commitment replacement is unavailable after settlement. Recovery then requires exceptional buffer funding, manual beneficiary compensation, or an upgrade/accounting backfill.

Likelihood is low because exploitation requires exact-boundary timing and favorable ordering. That affects exploitability, but not the demonstrated Critical impact once the sequence occurs.

## References

* [`CoverOrderAllocator.commitAllocation`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L332-L403)
* [`CoverOrderAllocator` settlement path](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L457-L537)
* [`FirelightVault.withdraw` and `redeem`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L717-L775)
* [`FirelightVault.payout`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L878-L966)
* [`FirelightVault._logTrace`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1055)
* [`Checkpoints` inclusive lookup and push](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L198-L223)
* [Flare timestamp validation](https://github.com/flare-foundation/go-flare/blob/53f8b033fd6822b6831d66d5acc682a5f13b22d4/coreth/consensus/dummy/consensus.go#L217-L225)
* [Flare block timestamp construction](https://github.com/flare-foundation/go-flare/blob/53f8b033fd6822b6831d66d5acc682a5f13b22d4/coreth/miner/worker.go#L139-L147)
* [Immunefi Firelight competition scope](https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/)

## Link to Proof of Concept

<https://gist.github.com/sebastian-osec/f62f39092c2f47c3ef8f6891c30470b1>

## Proof of Concept

## Proof of Concept

The PoC runs only on a local Hardhat network and uses the real proxy Vault, Allocator, CoverNFT, and IncidentManager contracts. Only standard ERC-20 tokens and a fixed `$1` price feed are mocked.

### Steps

1. Clone `https://github.com/immunefi-team/audit-comp-firelight.git` and check out revision `42f9ea5e43d88b35197b901acc2a8c24b314fa62`.
2. Run `npm ci`.
3. Create `test/security/`.
4. Attach/copy [`poc.test.js`](https://gist.githubusercontent.com/sebastian-osec/f62f39092c2f47c3ef8f6891c30470b1/raw/1fd79c0561693bbc64c329ba916728d9466c15ba/poc.test.js) into `test/security/poc.test.js`.
5. Run:

```sh
npx hardhat test test/security/poc.test.js
```

The preserved test has SHA-256 `f293b092d30e74b585510e7f9bdbf74f81e13a2855df957e51e147335a243c09` and passed at the pinned revision with Node `v24.16.0`, npm `11.13.0`, and Hardhat `2.22.19`:

```
FL-008: a boundary withdrawal mutates committed period-start capacity
  1 passing (620ms)
```

The assertions show that commit and withdrawal share timestamp `T`, commitment capacity remains 110, the later withdrawal changes `totalAssetsAt(T)` from 100 to zero, the incident pays 10 from the buffer and zero from the vault, and the shareholder later claims 100.
