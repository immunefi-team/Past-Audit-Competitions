# 89511 sc medium mutable period start checkpoint lets a covernft callback grief payouts for already settled cover

**Submitted on Aug 19th 2026 at 17:12:39 UTC by @ChadiSebbar for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89511
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

Summary

An attacker can deposit temporary collateral through a contract that is both a vault shareholder and a cover buyer. If the allocator commits and starts batch settlement at exact `start(P)`, minting the attacker's Cover NFT invokes the buyer's ERC-721 receiver callback. The callback withdraws the temporary collateral after the allocator has stored the higher capacity. This withdrawal writes a lower vault checkpoint at `start(P)`. The later incident payout uses that lower checkpoint even though settlement retained the higher committed cover. In the PoC, 1,000 units of cover remain settled, an honest order has 999 units of approved loss, the vault transfers zero to the configured aggregate `payoutReceiver`, and the incident closes with `vaultPaidAmount == 0`. The attacker later claims its full 1,000-unit delayed withdrawal.

The attacker recovers its temporary principal while the payout pipeline closes an incident with a 999-unit vault payout shortfall for honestly settled cover. The PoC does not measure attacker profit or theft.

## Affected asset

Primary affected asset:

* `contracts/core/FirelightVault.sol`

Relevant functions and components:

* `FirelightVault.withdraw`, `_requestWithdraw`, `_logTrace`, `totalAssetsAt`, and `payout`;
* `CoverOrderAllocator.commitAllocation`, `_computeAvailableCapacity`, and `_settleCoverOrder`;
* `CoverNFT.safeMint` and the standard ERC-721 receiver callback;
* `Checkpoints.Trace256.push`;
* `IncidentManager.approveCurrentAssessment` and `_executePayout`.

The authoritative source is `v1_audit_ready@42f9ea5e43d88b35197b901acc2a8c24b314fa62`. This submission does not rely on a live deployment or live configuration.

## Vulnerability details

### Broken invariant

Given that the allocator accepts a commitment for period `P` using vault collateral at `start(P)`, the settled cover and the later payout cap for `P` must use the same economic exposure.

The allocator and vault read the same historical value at different moments. The allocator stores capacity from the first read. A callback then writes a lower checkpoint at the same timestamp. The payout performs a second read and uses the lower value.

### The allocator stores the higher capacity

`commitAllocation` validates the submitted `matchingCapacity` against live capacity and stores it in the period commitment:

```solidity
_validateMatchingCapacity($, currentPeriod, matchingCapacity, totalAllocated);

commit.totalAvailableCapacity = matchingCapacity;
commit.totalDeclaredAllocated = totalAllocated;
```

`_computeAvailableCapacity` reads `vault.totalAssetsAt(vault.currentPeriodStart())`. At exact `start(P)`, before any checkpoint has been written with that key, the historical lookup returns the most recent earlier value. That value includes the attacker's pre-period deposit.

### The callback replaces the period-start exposure

Settlement updates the order and commitment before calling `CoverNFT.safeMint(order.buyer, orderId)`. A contract buyer receives `onERC721Received` before batch settlement returns.

The attacker contract owns its vault shares. Its receiver callback calls the public `FirelightVault.withdraw` function for those shares. The withdrawal is delayed, but `_logTrace` immediately writes the reduced active assets using the current timestamp:

```solidity
uint48 ts = Time.timestamp();
if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

Because the timestamp equals `start(P)`, this write becomes the historical value for the beginning of the period. The allocator commitment remains unchanged. Settlement continues against `commit.totalAvailableCapacity` and does not recompute vault capacity after the callback.

This produces incompatible state for the same period:

```
capacity accepted and settled for P = 1,000
totalAssetsAt(start(P))             =     0
```

### Payout trusts the lower value

For an approved incident, `FirelightVault.payout` calculates:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 activePayableAmount =
    currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

The vault still physically holds the delayed withdrawal assets. However, the zero `assetsAtCapturePeriod` value caps `paidAmount` at zero. The pending assets are therefore not charged for this incident.

`IncidentManager.approveCurrentAssessment` accepts the zero result, records `vaultPaidAmount == 0`, and sets the incident status to `CLOSED`. A second approval attempt reverts because the incident is no longer under evaluation. The frozen contracts do not record the unpaid amount as debt or expose a beneficiary claim function for it.

## Attack scenario

1. Before `start(P)`, the attacker deposits 1,000 vault-asset units for an attacker-controlled contract.
2. The attacker contract is the buyer of a 1-unit cover order. A separate honest buyer has a 999-unit order.
3. The matcher includes both orders in a 1,000-unit allocation.
4. At exact `start(P)`, the authorized allocator performs the ordinary commitment and batch-settlement flow.
5. The commitment accepts the 1,000-unit capacity derived from the attacker's deposit.
6. Settlement safely mints the attacker's Cover NFT. The receiver callback withdraws all 1,000 vault-asset units.
7. The withdrawal writes zero to the exact period-start checkpoint and creates a delayed withdrawal entitlement.
8. Settlement retains 1,000 units of cover, including 999 units for the honest order.
9. Honest protocol roles approve a valid 999-unit loss against the honest order.
10. The vault reads the zero checkpoint and transfers zero to the configured aggregate `payoutReceiver`.
11. The incident closes. Retrying the normal approval path reverts.
12. In `P+2`, the attacker claims its full 1,000-unit withdrawal entitlement.

The attacker controls only its deposit, contract buyer, NFT callback, and withdrawal. The allocator, curator, and assessment approver perform ordinary protocol actions.

## Impact

The PoC measures:

| Measurement                                       |                           Result |
| ------------------------------------------------- | -------------------------------: |
| Capacity committed and settled                    |                      1,000 units |
| Honest order's settled cover                      |                        999 units |
| Approved loss against the honest order            |                        999 units |
| Final `totalAssetsAt(start(P))`                   |                          0 units |
| Vault transfer to aggregate `payoutReceiver`      |                          0 units |
| Vault payout shortfall for honestly settled cover |                        999 units |
| Attacker deposit and later claim                  |              1,000 / 1,000 units |
| Incident state                                    | `CLOSED`, `vaultPaidAmount == 0` |

All vault and First-Loss Buffer payments go to the configured aggregate `IncidentManager.payoutReceiver`. The honest order's stored `payoutRecipient` is a separate accounting identity. Final distribution to individual beneficiaries is off-chain and was not measured.

The normal manual distribution process cannot distribute vault funds that never reached the aggregate receiver. Firelight could choose exceptional off-chain compensation, but the frozen contracts neither guarantee nor record such remediation.

## Limitations and nonclaims

* The exploit requires honest commitment and first settlement while the block timestamp equals exact `start(P)`.
* Public evidence does not establish how often the honest runner lands both operations at that exact timestamp.
* The PoC does not measure final off-chain transfers to individual `payoutRecipient` addresses.
* The result is a vault payout shortfall, not proof that a beneficiary receives zero from every possible external source.
* The report does not claim attacker profit, theft, insolvency, live deployment, guaranteed repetition, or protocol-wide shutdown.
* Private known-issue status is unavailable to the submitter.

## Known-issue boundary

The closest public families concern collateral drawdown, capacity validation, historical checkpointing, and payout accounting. This mechanism requires a different ordering:

```
high capacity committed
→ CoverNFT receiver callback
→ lower exact-start checkpoint written
→ high cover remains settled
→ payout reads lower historical exposure
```

The public material reviewed for this competition does not describe this allocator-to-payout snapshot split or the surviving full withdrawal entitlement. Private known issues remain unknown.

## Recommended fix

Materialize one immutable vault-exposure value for period `P`. Bind the allocation commitment, settlement limits, and incident payout cap to that same value. A later checkpoint written at the same timestamp must not change the exposure already accepted for the period.

Add regression coverage for both same-block orders:

```
commit → withdraw
withdraw → commit
```

Include a contract buyer that calls `withdraw` from `onERC721Received`. The first ordering must not leave settled cover above payout exposure. The second ordering must validate capacity against the reduced collateral.

## Source references

* [`FirelightVault.withdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L755-L777)
* [`FirelightVault.payout`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892-L965)
* [`FirelightVault._logTrace`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1055)
* [`CoverOrderAllocator.commitAllocation`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L332-L356)
* [`CoverOrderAllocator._settleCoverOrder`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L457-L539)
* [`CoverOrderAllocator._computeAvailableCapacity`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L696-L724)
* [`IncidentManager.approveCurrentAssessment`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L403-L438)
* [`IncidentManager._executePayout`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L621-L705)

## Link to Proof of Concept

<https://gist.github.com/Dar-Own/a25bf01456d5cf8f0c70d55251c91f60>

## Proof of Concept

Proof of Concept

This PoC reproduces the period-start checkpoint split against the frozen Firelight competition source. It demonstrates that a CoverNFT receiver callback can withdraw temporary collateral after higher cover capacity is committed. The callback changes the historical payout cap without changing the stored commitment or settled cover.

The test runs entirely on the local Hardhat network. It does not require an RPC, a `.env` file, live contracts, or privileged attacker credentials.

## Frozen source and requirements

* Repository: `https://github.com/immunefi-team/audit-comp-firelight.git`
* Commit: `42f9ea5e43d88b35197b901acc2a8c24b314fa62`
* Node.js: 20 or 22
* Package manager: npm

The PoC package contains three files:

```
README.md
period-start-checkpoint.js
WithdrawOnCoverReceipt.sol
```

Place the executable files at these paths in the frozen repository:

```
period-start-checkpoint.js
  → test/clock-differential/period-start-checkpoint.js

WithdrawOnCoverReceipt.sol
  → contracts/test/WithdrawOnCoverReceipt.sol
```

`WithdrawOnCoverReceipt.sol` is the attacker-controlled contract buyer. It owns its vault shares, authenticates the CoverNFT callback, and calls the production vault's public `withdraw` function once.

## Setup

Clone and install the frozen source:

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm ci
```

Create the destination directories:

```bash
mkdir -p test/clock-differential contracts/test
```

Then place the downloaded Gist files at these exact paths:

```
test/clock-differential/period-start-checkpoint.js
contracts/test/WithdrawOnCoverReceipt.sol
```

Confirm the frozen commit:

```bash
git rev-parse HEAD
```

Expected value:

```
42f9ea5e43d88b35197b901acc2a8c24b314fa62
```

## Run the primary exploit

```bash
npm test -- test/clock-differential/period-start-checkpoint.js --grep "RL-001 callback exploit"
```

The test should finish with `1 passing` and print:

```
RL001_CALLBACK_RESULT={"committedCover":"1000","honestSettledCover":"999","approvedHonestLoss":"999","payoutReceiverReceived":"0","vaultPayoutShortfall":"999","attackerDeposit":"1000","attackerClaim":"1000","incidentClosed":true}
```

The vault asset uses 6 decimals. Cover and assessed losses use 18 canonical decimals. The receipt displays human units after conversion.

## Step-by-step explanation

### 1. Fund the attacker-controlled contract buyer

The attacker receives 1,000 vault-asset units and deposits them into `FirelightVault`. The vault shares are minted to `WithdrawOnCoverReceipt.sol`. The helper therefore owns the shares it later withdraws.

The attacker arms the helper for a one-time withdrawal of all 1,000 units.

### 2. Create attacker and honest orders

The ordinary curator flow creates two orders:

```
attacker contract order:   1 unit of cover
separate honest order:   999 units of cover
total allocation:      1,000 units of cover
```

The attacker needs positive allocated cover so settlement mints a Cover NFT to the helper and invokes its ERC-721 receiver callback. The 999-unit order keeps the measured loss separate from the attacker's own cover.

### 3. Commit and begin settlement at exact `start(P)`

The test places the authorized allocator's `commitAllocation` and `batchSettleCoverOrder` calls in one block whose timestamp equals exact `start(P)`.

Before the callback, `totalAssetsAt(start(P))` returns the earlier 1,000-unit deposit. `commitAllocation` accepts and stores 1,000 units of capacity.

### 4. Withdraw from the CoverNFT callback

Settlement records the attacker's allocation and calls `CoverNFT.safeMint`. The safe mint invokes `WithdrawOnCoverReceipt.onERC721Received`.

The callback calls:

```solidity
vault.withdraw(1_000 units, attacker, helper)
```

The withdrawal burns the helper's shares and creates a delayed entitlement for the attacker. It also writes zero active assets at the exact `start(P)` checkpoint. The helper clears its armed state before making the external vault call, so the callback is one-shot.

### 5. Verify that settled cover still uses the higher commitment

After the callback, the test verifies:

```
committed capacity:             1,000
total settled cover:            1,000
honest order settled cover:       999
totalAssetsAt(start(P)):             0
attacker delayed entitlement:    1,000
```

The honest order's stored `payoutRecipient` is also checked separately from the configured aggregate `IncidentManager.payoutReceiver`.

### 6. Approve an honest loss and execute payout

Honest protocol roles create and confirm an incident. They add a valid 999-unit assessment loss against the separate honest order and submit it for approval.

The assessment approver performs the normal approval action. The First-Loss Buffer balance is zero in this test, so the remaining approved loss is passed to the vault.

The vault calculates:

```
requested payout                  = 999
totalAssetsAt(capturePeriodStart) =   0
currently payable assets          = 1000

actual payout = min(999, 0, 1000) = 0
```

The test verifies:

```
approved honest loss:                     999
aggregate payoutReceiver balance increase: 0
vault payout shortfall:                    999
incident status:                        CLOSED
incident.vaultPaidAmount:                   0
```

It then attempts `approveCurrentAssessment(incidentId)` a second time. The call reverts with `InvalidIncidentStatus(incidentId, CLOSED)`. This proves that the normal approval path cannot retry the unpaid vault amount after the incident closes.

### 7. Claim the attacker withdrawal

The test advances to `P+2` and calls `claimWithdraw` for the attacker. The attacker receives the full 1,000-unit delayed entitlement.

The final measured result is:

```
honestly settled cover with approved loss: 999
vault contribution to payout:                0
vault payout shortfall:                     999
attacker deposit recovered:               1,000
```

## Primary assertions

The test requires all of the following:

* 1,000 units of capacity are committed and settled;
* the honest order retains 999 units of allocated cover;
* the honest order stores its own `payoutRecipient`;
* the aggregate `payoutReceiver` is a different address;
* the First-Loss Buffer balance is zero;
* the approved assessment loss is 999 units;
* the aggregate receiver's vault-asset balance increases by zero;
* the vault payout shortfall is 999 units;
* the incident is `CLOSED` with `vaultPaidAmount == 0`;
* retrying approval reverts with `InvalidIncidentStatus`;
* the attacker retains a 1,000-unit withdrawal entitlement;
* the attacker later claims all 1,000 units.

## Causal controls

The same file contains three controls. No additional files are required.

### Exact start versus one second later

```bash
npm test -- test/clock-differential/period-start-checkpoint.js --grep "like-for-like start"
```

This control holds 2,000 units of cover and approved loss constant. Withdrawing at exact `start(P)` produces a 1,000-unit payout shortfall. Withdrawing at `start(P)+1` preserves the 2,000-unit period-start checkpoint and produces no shortfall.

### Full-principal-recovery control

```bash
npm test -- test/clock-differential/period-start-checkpoint.js --grep "full-principal-recovery grief"
```

This control holds 1,000 units of cover constant. At exact `start(P)`, the period-start checkpoint and payout are zero while the attacker later claims 1,000 units. At `start(P)+1`, the vault pays the full 1,000 units and the attacker has no remaining claim.

### Same-timestamp transaction-order control

```bash
npm test -- test/clock-differential/period-start-checkpoint.js --grep "commit snapshot split"
```

This control compares `commit → withdraw` with `withdraw → commit` at the same timestamp. The high commitment succeeds only when it is stored before the checkpoint falls.

Run the complete file with:

```bash
npm test -- test/clock-differential/period-start-checkpoint.js
```

The complete file should finish with `4 passing`.

## What the PoC proves

The PoC proves that the frozen production contracts can reach this state under the tested exact-boundary ordering:

```
settled cover for P > exposure later used to cap P's vault payout
```

It also proves that the zero-paid incident closes, the normal approval path cannot retry it, and the attacker's full delayed withdrawal remains claimable in the primary scenario.

The PoC measures the transfer to the configured aggregate `payoutReceiver`. It does not measure final off-chain distribution to individual beneficiary addresses. It does not prove that exceptional manual compensation is impossible.

## Scheduling limitation

The exploit requires honest `commitAllocation` and first settlement while the block timestamp equals exact `start(P)`. Public evidence supports the protocol performing matching at period boundaries and the chain permitting that timestamp. Public evidence does not establish how often the honest runner lands both operations at that exact timestamp.
