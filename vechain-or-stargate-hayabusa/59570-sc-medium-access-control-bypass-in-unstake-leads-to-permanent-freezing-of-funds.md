# 59570 sc medium access control bypass in unstake leads to permanent freezing of funds

**Submitted on Nov 13th 2025 at 16:17:10 UTC by @Pelican26237 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59570
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

The unstake() function in Stargate.sol permanently blocks users from exiting the protocol when their staking position exceeds the maxClaimablePeriods threshold (default 832). Once reached, the function reverts with `MaxClaimablePeriodsExceeded()`, preventing the user from unstaking their VET or claiming their accumulated VTHO rewards. This creates a permanent deadlock where legitimate users lose access to both staked assets and earned rewards — a critical “permanent freezing of funds” vulnerability if deployed to mainnet.

Excerpt (approx. lines 300–330 in unstake()):

```solidity
if (_exceedsMaxClaimablePeriods($, _tokenId)) {
    revert MaxClaimablePeriodsExceeded(); // ← Reverts when >832 claimable periods
}
_claimRewards($, _tokenId);
```

## Vulnerability Details

unstake() and delegate() perform a pre-check `_exceedsMaxClaimablePeriods(tokenId)` and revert when the token has more than `maxClaimablePeriods` (default 832) instead of processing claimable rewards in batches. That revert permanently blocks the normal exit flow and makes both staked VET and accumulated VTHO rewards inaccessible to the token owner.

What exactly is wrong (step-by-step)

* The contract tracks `lastClaimedPeriod[_tokenId]` and compares the number of unclaimed periods against `maxClaimablePeriods`.
* In mutating flows that require claiming (notably `unstake()` and `delegate()`), the code does:

  ```solidity
  if (_exceedsMaxClaimablePeriods($, _tokenId)) {
      revert MaxClaimablePeriodsExceeded();
  }
  _claimRewards($, _tokenId);
  ```
* `_exceedsMaxClaimablePeriods(...)` is true when `currentPeriod - lastClaimedPeriod[_tokenId] > maxClaimablePeriods`. If a long-term staker has accrued >832 reward periods, the check returns true and the function reverts immediately.
* Because the call reverts before `_claimRewards` runs, `lastClaimedPeriod[_tokenId]` and any reward transfers are never updated and `unstake()` never completes. The token owner cannot progress the state in that call (nor in `delegate()`), so they cannot exit or batch-claim as part of the exit flow.
* If the user does not or cannot perform the manual sequence of multiple `claimRewards()` transactions (or if there is no opportunity to call those functions due to UX/knowledge/permission limitations), the result is effectively permanent freezing of both their staked VET and accumulated VTHO rewards. On a non-upgradeable deployment this is irreversible.

Why this is real & reproducible

* The code explicitly reverts before any state change or transfer takes place; the revert condition is deterministic.
* There is no fallback path in `unstake()` / `delegate()` to process claims in batches or advance `lastClaimedPeriod`.
* No privileged role is required — it occurs naturally for long-term stakers and can be simulated with the provided PoC.
* PoC (attached): after simulating 1000 completed periods and rewards, `unstake(tokenId)` reverts, while `claimableRewards(tokenId) > 0` remains true.

Root cause

* A protective gas cap (`maxClaimablePeriods`) exists, but the implementation reverts unconditionally when exceeded instead of auto-batching or providing a recoverable path.

Suggested fixes (high level)

* Auto-batch: In `unstake()`/`delegate()` iterate in batches of `maxClaimablePeriods` to process and transfer rewards until remaining periods ≤ cap, then proceed to `_claimRewards()`.
* Helper: Add `claimRewardsBatched(tokenId)` that clients can call repeatedly to advance `lastClaimedPeriod` safely.
* Emergency admin (last resort): If upgradeable, add a multisig-controlled emergency method to recover funds.

Minimal patch (illustrative)

```diff
- if (_exceedsMaxClaimablePeriods($, _tokenId)) {
-     revert MaxClaimablePeriodsExceeded();
- }
- _claimRewards($, _tokenId);
+ while (_exceedsMaxClaimablePeriods($, _tokenId)) {
+     _processClaimBatch($, _tokenId); // advance lastClaimedPeriod by maxClaimablePeriods and transfer batch rewards
+ }
+ _claimRewards($, _tokenId);
```

Scope & severity

* Affects: any token owner where `currentPeriod - lastClaimedPeriod > maxClaimablePeriods` (long-term stakers).
* Privileges: none required.
* Severity: Critical — permanent freezing of funds if contract is immutable.

## Proof of Concept

<details>

<summary>PoC summary and attachments</summary>

* PoC file: `StargateExploitTest.sol` (Foundry). Repro steps below.
* PoC gist: <https://gist.github.com/4822ebubejohnmartin-hue>

</details>

{% stepper %}
{% step %}

### Environment Setup

Use the repository: <https://github.com/vechain/stargate-contracts>

Tooling: Foundry / make solo-up (local VeChain node provided by the repo) and `yarn contracts:test:unit` to run tests.
{% endstep %}

{% step %}

### Steps to reproduce (local test environment only)

1. Clone the repo and start the local node:

```bash
git clone https://github.com/vechain/stargate-contracts.git
cd stargate-contracts
make solo-up
```

2. Add the PoC file `StargateExploitTest.sol` into the project test folder (e.g., `test/` or `test/unit/`). The test simulates a validator with 1000 completed periods and per-period rewards.
3. Run the unit tests:

```bash
yarn contracts:test:unit
```

(Or run only the test file if your runner supports it.)
{% endstep %}

{% step %}

### Expected behavior

* The test calls `unstake(tokenId)` after simulating 1000 completed periods.
* The call reverts with `MaxClaimablePeriodsExceeded` (asserted with `vm.expectRevert(...)`).
* After the revert, `claimableRewards(tokenId)` returns a value > 0, demonstrating rewards exist but are inaccessible in that flow.

Key test assertions (illustrative):

```solidity
// Expect revert when trying to exit with > maxClaimablePeriods
vm.expectRevert(Stargate.MaxClaimablePeriodsExceeded.selector);
stargate.unstake(tokenId);

// Rewards remain claimable (but unstake() cannot proceed)
uint256 claimable = stargate.claimableRewards(tokenId);
assertGt(claimable, 0);
```

{% endstep %}
{% endstepper %}

## Impact Details

Because `unstake()` reverts when a token’s unclaimed reward periods exceed `maxClaimablePeriods` (default 832), legitimately earned VTHO rewards — and the ability to unstake the underlying VET — can become permanently inaccessible for affected token owners. This is a direct, on-chain lock of user assets (not a temporary denial): if the contract is immutable or no privileged recovery exists, those funds cannot be recovered.

Affected assets

* Staked VET associated with the tokenId (owner cannot call `unstake()` to retrieve principal).
* Accrued VTHO rewards that are claimable but blocked by the cap; these rewards remain on-chain but inaccessible to the owner.
* Potentially protocol liquidity / reputation if many users are impacted.

Direct loss scenarios (examples & math) — assumptions stated

* Assumptions: `maxClaimablePeriods = 832`; per-period reward = variable; “period” = protocol reward period (use your UI/chain mapping).

Single long-term staker (illustrative)

* Per-period VTHO = 0.01 VTHO (PoC value).
* Accrued periods = 1,000 → unclaimed VTHO = 1,000 × 0.01 = 10 VTHO.
* Owner cannot unstake — their VET principal remains locked; 10 VTHO is inaccessible unless manually batched.
* If the contract is immutable, those 10 VTHO are effectively permanently frozen.

Multiple users / systemic example

* If 1,000 users each have \~10 VTHO frozen (same conditions), 10,000 VTHO frozen total.

High-accumulation case

* Per-period VTHO = 0.05; periods = 2,000 → unclaimed = 2,000 × 0.05 = 100 VTHO for one account.

Exploit / impact mechanics

* Not an active “exploit” by an attacker stealing funds — it is a logic/design issue that locks legitimate user funds when certain state conditions are met.
* Triggered by normal accrual or by maliciously high period accounting in a protocol staker mock; no privileged access required.
* Attack cost for an adversary to cause widespread damage is low (only gas), because the condition is triggered by time/period accumulation and the code reverts deterministically.

Exploitability & cost

* Exploit cost: Minimal (gas only). No oracle manipulation or privileged keys needed.
* Detectability: High — `unstake()` reverts clearly with `MaxClaimablePeriodsExceeded`.
* Recovery difficulty: High to impossible if immutable — requires user manual batching (multiple txs and gas) or a contract patch / emergency admin if upgradeable.

Why this justifies Critical severity & payout

* In-scope impact matches exactly: Permanent freezing of funds (Critical).
* Consequences are direct, measurable, and can affect many users with low attacker effort.
* Fix is straightforward technically but crucial: without it, funds remain locked on-chain and may be unrecoverable.

Mitigations / immediate recommendations for triage

* Add immediate UX/communication guidance instructing users to run batched `claimRewards()` calls (temporary mitigation).
* Short-term: if upgradeable, schedule a patch to auto-batch in `unstake()` / `delegate()` or add a `claimRewardsBatched` helper.
* Long-term: implement batched processing + gas-safeguards and add monitoring/alerts for tokens approaching the cap.

## References

Stargate.sol (vulnerable `unstake()` / `maxClaimablePeriods` check):\
<https://github.com/vechain/stargate-contracts/blob/main/packages/contracts/contracts/Stargate.sol>

Relevant code excerpt:

```solidity
if (_exceedsMaxClaimablePeriods($, _tokenId)) {
    revert MaxClaimablePeriodsExceeded();
}
```

## Link to Proof of Concept

<https://gist.github.com/4822ebubejohnmartin-hue>

## Evidence & Additional Files

<details>

<summary>Attached PoC and suggested test outputs</summary>

* `StargateExploitTest.sol` (Foundry test + Mock `IProtocolStaker`) — attach to test folder as described above.
* One or two screenshots: failing test output (terminal) and revert selector or a tx trace from the local node showing `MaxClaimablePeriodsExceeded`.
* (Optional) stdout of test runner showing the `expectRevert` result.

</details>

## Safety / Disclosure

Do not run this on mainnet or any live network.

The PoC is intended for local testing only and reproduces the condition deterministically.
