# 90086 sc high inclusive period boundary checkpoint substitution in firelightvault withdraw lets a withdrawing shareholder retain funds owed to an approved claimant

**Submitted on Aug 22nd 2026 at 22:04:42 UTC by @ErenYeagerMax for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90086
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

An ordinary Firelight vault shareholder can request a delayed withdrawal at the exact timestamp a capture period begins, causing an inclusive checkpoint lookup to treat the reduced post-withdrawal active balance as that period's opening exposure even though the withdrawn assets remain held and slashable as pending withdrawals. If an incident from that period is approved, the corrupted opening checkpoint caps the claimant's payout below the assets actually available, allowing part of the withdrawing shareholder's pending collateral to survive and later be claimed.

## Vulnerability Details

I reproduced the issue against the `v1_audit_ready` competition source at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`. A local Foundry 1.7.1 run compiled the target with Solidity 0.8.28 and passed both the exploit test and a matched control.

Calling `withdraw()` burns shares and schedules a later claim; it does not immediately transfer the underlying asset out of the vault. Pending withdrawal assets remain in the contract and remain part of the incident-payment waterfall.

For a covered incident, Firelight combines:

* the assets recorded when the capture period opened; and
* the active-plus-pending assets currently available for payment.

Those calculations should describe the same economic exposure. Moving assets from active accounting into a still-slashable pending bucket at the boundary should not remove them from the opening cap while retaining them in the current-payment calculation.

In `contracts/core/FirelightVault.sol`, `withdraw()` queues the delayed withdrawal and then records the reduced active totals:

```solidity
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) =
    _previewTotals(assets, true, Math.Rounding.Ceil);

uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);

_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

`_requestWithdraw()` leaves the assets inside the vault and adds them to pending-withdrawal accounting:

```solidity
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
```

`_logTrace()` writes the reduced active balance using the current timestamp:

```solidity
uint48 ts = Time.timestamp();
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

`totalAssetsAt()` uses an inclusive checkpoint lookup:

```solidity
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

The helper in `contracts/core/lib/Checkpoints.sol` returns the newest checkpoint whose key is less than or equal to the requested key. A withdrawal checkpoint written exactly at `capturePeriodStart` is therefore treated as the opening-period value, even though the queued assets remain payable.

The mismatch reaches the payout sink in `FirelightVault.payout()`:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

uint256 activePayableAmount =
    currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;

paidAmount = Math.min(
    amount,
    Math.min(assetsAtCapturePeriod, activePayableAmount)
);
```

After the boundary withdrawal, the fixture contains 780,000 active units and 420,000 pending units, so `activePayableAmount` correctly equals 1,200,000. However, `assetsAtCapturePeriod` is only 780,000. The corrupted opening checkpoint becomes the binding cap.

The complete attack path is:

1. Mallory and Alice are ordinary shareholders with deposits already in the vault.
2. A curator creates a legitimate 1,200,000-unit cover order for the following period.
3. Mallory calls the public `withdraw()` entry point for 420,000 units at the exact next-period boundary.
4. The assets remain in the vault as pending withdrawals, but the boundary checkpoint records only Alice's 780,000 active units.
5. Under the fixture's contract-permitted 2x capacity ratio, the real allocator computes 1,560,000 units of capacity from that checkpoint and settles the 1,200,000-unit order using a StandardMerkleTree-compatible single-leaf commitment. It collects premium, marks the order matched, and mints the real `CoverNFT`.
6. A covered incident is captured later in the same period and approved through the ordinary `IncidentManager` path. Mallory holds no incident, allocator, or payout role.
7. The approved 1,200,000-unit request reaches the vault. The vault counts all 1,200,000 active-plus-pending units as payable but caps the transfer at the 780,000 opening checkpoint.
8. The payout consumes 507,000 active units and 273,000 of Mallory's pending units. Alice retains 273,000 active units, while Mallory's pending claim retains 147,000 units.
9. After the normal withdrawal delay, Mallory calls `claimWithdraw()` and receives the surviving 147,000 units.

Only the withdrawal requires exact-boundary execution. Cover settlement and incident capture happen afterward, and Mallory needs no incident, allocator, or payout privilege. Exact-boundary inclusion is nevertheless probabilistic for a public sender: Flare permits boundary-timestamp blocks, but propagation, congestion, fees, and block construction remain outside the sender's control. The PoC establishes the state transition after exact-boundary inclusion; it does not claim reliable inclusion or historical exploitation.

The local proof uses a zero-balance first-loss buffer only to isolate the vault leg. A funded buffer reduces the amount requested from the vault but does not repair the mismatch whenever the remaining approved request exceeds the corrupted opening cap. The fixture values are representative test values and are not claims about live balances or configuration.

Define period-opening exposure as the state strictly before the boundary and use the same definition in both underwriting capacity and payout calculations. A deposit, withdrawal, or reward checkpoint written at the boundary must not retroactively replace the exposure carried into the new period.

A source-compatible approach is to add a strict-before lookup in `contracts/core/lib/Checkpoints.sol` that returns the latest checkpoint whose timestamp is less than the requested boundary. Both `FirelightVault.payout()` and `CoverOrderAllocator._computeAvailableCapacity()` should use that same opening value. An immutable opening snapshot recorded before boundary-time accounting changes would also work.

Regression tests should cover deposits, withdrawals, and reward checkpoints at `periodStart - 1`, `periodStart`, and `periodStart + 1`; allocator capacity and incident payout must agree on the same opening exposure. Tests should also include a full-payment control, a partially funded buffer, and a request below the opening cap.

## Impact Details

**Selected impact:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.

In the reproduced scenario, Mallory deposits 420,000 units and Alice deposits 780,000 units, for 1,200,000 total vault assets. A contract-valid allocation supports an approved 1,200,000-unit incident payment. The matched control pays the claimant the full 1,200,000 units. With Mallory's boundary withdrawal, the claimant instead receives 780,000 units, a 420,000-unit shortfall.

Mallory's 420,000-unit pending withdrawal absorbs 273,000 units of the payout and retains 147,000 units that she later claims. The accounting is therefore `420,000 - 273,000 = 147,000` for Mallory and `1,200,000 - 780,000 = 420,000` for the claimant's shortfall. The exploit case conserves the initial assets: 780,000 paid to the claimant, 273,000 left active for Alice, and 147,000 later claimed by Mallory equals 1,200,000.

Mallory does not obtain a net gain: relative to her 420,000-unit deposit, she suffers an absolute loss of 273,000 units. The direct-theft classification rests on her retaining 147,000 units that the matched control consumes while the approved claimant is underpaid by 420,000 units. I assess the issue as **High** conservatively because inclusion at the exact boundary is timing-dependent and the demonstrated attacker incurs a loss. If the retained counterfactual amount is not accepted as direct theft, **Medium griefing** is the conservative fallback classification; griefing is not selected as a separate form impact.

## References

* [`FirelightVault.sol` in the scoped `v1_audit_ready` branch](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol)
* [`Checkpoints.sol` in the scoped `v1_audit_ready` branch](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/lib/Checkpoints.sol)
* [`CoverOrderAllocator.sol` in the scoped `v1_audit_ready` branch](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/CoverOrderAllocator.sol)
* [Firelight Audit Competition scope](https://immunefi.com/audit-competition/audit-comp-firelight-1/scope/)
* [Flare network timing and transaction ordering](https://dev.flare.network/network/overview)
* [Assessed repository revision `42f9ea5e`](https://github.com/immunefi-team/audit-comp-firelight/tree/42f9ea5e43d88b35197b901acc2a8c24b314fa62)

## Link to Proof of Concept

<https://gist.github.com/erenyeager1483/951f740743f81a78dbb0d4c20e852580>

## Proof of Concept

## Proof of Concept

The PoC runs entirely on Foundry's disposable local EVM. It uses no RPC endpoint, fork, private key, broadcast transaction, or live assets. It deploys the real upgradeable `FirelightVault`, `CoverOrderAllocator`, `CoverNFT`, and `IncidentManager` behind transparent proxies. Only the ERC-20 assets and Chainlink-shaped price feed are local mocks.

## Files

* `foundry.toml`
* `test-foundry/C02BoundaryWithdrawal.t.sol`

## Clean-checkout reproduction

1. Clone the competition repository and pin the assessed revision:

```sh
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
```

2. Install the repository's locked dependencies:

```sh
npm ci
```

3. Save the provided `foundry.toml` at the repository root and save the provided test as `test-foundry/C02BoundaryWithdrawal.t.sol`.
4. Run only the PoC contract:

```sh
forge test --match-contract C02BoundaryWithdrawalTest -vv
```

## What the tests establish

`test_control_fullApprovedPayout()` executes the real order, allocation, premium, NFT, incident-assessment, and payout path without the boundary withdrawal. The receiver obtains the full 1,200,000 units.

`test_exploit_boundaryWithdrawalUnderpaysAndPreservesClaim()` repeats that flow with Mallory's 420,000-unit withdrawal at the boundary. It asserts:

* the boundary checkpoint falls from 1,200,000 to 780,000;
* the full 420,000 withdrawal remains queued and held by the vault;
* the approved receiver obtains 780,000 rather than 1,200,000;
* 273,000 active units remain attributable to Alice;
* Mallory's pending withdrawal retains 147,000 units; and
* Mallory can claim those 147,000 units after the ordinary delay.

Conservation in the exploit case is:

```
780,000 paid to the claimant
+ 273,000 remaining active assets
+ 147,000 later claimed by Mallory
= 1,200,000 initial vault assets
```

## Observed output

Verified with Foundry 1.7.1 and Solidity 0.8.28:

```
Ran 2 tests for test-foundry/C02BoundaryWithdrawal.t.sol:C02BoundaryWithdrawalTest
[PASS] test_control_fullApprovedPayout() (gas: 22091138)
[PASS] test_exploit_boundaryWithdrawalUnderpaysAndPreservesClaim() (gas: 22385821)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

No cleanup is required beyond terminating the local test process.
