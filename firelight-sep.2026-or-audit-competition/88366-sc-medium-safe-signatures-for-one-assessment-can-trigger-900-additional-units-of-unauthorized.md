# 88366 sc medium safe signatures for one assessment can trigger 900 additional units of unauthorized depletion

## #88366 \[SC-Medium] Safe signatures for one assessment can trigger 900 additional units of unauthorized depletion

**Submitted on Aug 12th 2026 at 21:28:51 UTC by @anbug for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88366
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/IncidentManager.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

### Brief / Intro

`approveCurrentAssessment` authorizes an incident id, not an assessment round. A curator can submit round 1, allow the independent three-of-five Risk Consortium Safe to review and sign its approval, cancel round 1, and submit materially different round 2 losses. The already-signed Safe transaction is unchanged and approves whichever round is current when it executes.

The PoC uses the frozen Firelight vault, allocator, CoverNFT, and IncidentManager plus an official Safe 1.4.1 singleton/proxy. Three Safe owners sign while round 1 contains a 100-unit loss. The curator replaces it with a valid but unreviewed 1,000-unit round. A distinct non-owner relayer executes the original signatures, moving 250 funded and approved first-loss-buffer units and 750 vault units. The control pays only the reviewed 100-unit round and leaves the vault untouched.

This is a constraint bypass, not a claim that the curator may simply publish bad data. The documented curator proposes a loss schedule; the independent Risk Consortium Safe is the primary on-chain check that validates that schedule. Here no Safe owner approves round 2, yet round 2 receives the Safe's authority and closes the incident.

### Vulnerability Details

The assessment lifecycle deliberately supports correcting a submitted upload:

1. `cancelCurrentAssessment(incidentId)` lets the curator cancel a `DRAFT` or `UNDER_EVALUATION` round.
2. The next `addAssessmentLosses` call sees a canceled round and increments `incident.currentAssessmentRoundId`.
3. `submitCurrentAssessment(incidentId)` submits the replacement.

That workflow is safe only if approval binds the particular replacement. It does not. The approval ABI is:

```solidity
function approveCurrentAssessment(uint256 incidentId) external;
```

At execution, `approveCurrentAssessment` calls `_setCurrentAssessmentStatus`, which then reads the mutable `incident.currentAssessmentRoundId`. Neither the expected round id nor a hash of its losses is supplied by the approver.

The Safe transaction hash correctly binds the Safe, chain, target, calldata, gas fields, and Safe nonce. Its calldata remains `approveCurrentAssessment(1)` before and after replacement. Curator transactions do not consume the Safe nonce, so three signatures collected for round 1 remain executable after round 2 becomes current. This is not a replay: the Safe transaction executes once. The defect is that the callee gives the signed transaction a different meaning.

1. Two legitimate cover orders settle for the same period: 100 units for the reviewed recipient and 1,000 units for a different recipient. Total settled cover, 1,100, is below the demonstrated 1,250 capacity.
2. A distinct staker supplies 1,000 vault units. The first-loss-buffer wallet holds and approves 250 units.
3. The curator creates one genuine incident, confirms it, adds the 100-unit loss to assessment round 1, and submits it.
4. Three of five Safe owners sign the official Safe transaction hash for `approveCurrentAssessment(1)`. At this point round 1 is the only submitted round, totals 100, and attributes zero loss to the second recipient.
5. The curator cancels round 1, opens round 2, adds a 1,000-unit loss against the separate settled cover, and submits it.
6. The Safe nonce is still zero and its transaction hash is identical.
7. Any relayer can submit the three valid signatures to `Safe.execTransaction`. `IncidentManager` sees the Safe as `msg.sender`, resolves current round 2, approves 1,000, and closes the incident.
8. The payout receiver receives 250 buffer units and 750 vault units. The staker's withdrawable assets fall from 1,000 to 250.

Relative to the reviewed round, 900 additional units leave backing: 150 extra buffer units and 750 vault units.

The curator performs calls explicitly available to it, but obtains an effect explicitly reserved for a separate authority. The public trust model says the curator publishes proposed losses, while the independent Risk Consortium Safe validates coverage and payout accuracy and is the primary on-chain check. The competition rules keep privileged findings in scope when a role exceeds its attributed authority or bypasses an intended protocol constraint.

No Safe collusion, compromised Safe key, incorrect consortium vote, governance action, oracle manipulation, or unprivileged access to the approver role is assumed. Three owners authorize the reviewed 100-unit schedule. The lower curator role changes what that authorization approves afterward.

The PoC does not use aggregate duplicate claims: there is one incident, round 1 is canceled and unpaid, and only round 2 closes. Each loss is individually at or below its distinct settled cover-market allocation. Total settled cover remains within demonstrated capacity.

The first-loss buffer is not empty or unapproved. It contains 250 units and has approved exactly 250 for the IncidentManager. The PoC does not rely on a depeg or a nonstandard token.

The on-chain proceeds go to the configured protocol payout receiver. The report therefore uses the listed Medium griefing impact rather than claiming Critical direct theft, governance manipulation, or insolvency. The demonstrated harm is unauthorized, irreversible depletion of first-loss-buffer and staker backing despite the independent approver reviewing a different schedule.

A compromised or malicious curator can turn a pending Safe authorization for a small reviewed assessment into approval of a larger replacement assessment. This defeats the separation of duties described as the most important assumption in the claim flow.

The PoC demonstrates 900 units of incremental unauthorized depletion and a 75% reduction in a distinct staker's withdrawable principal. The attack does not require profit for the curator; it damages the protocol and its stakers, matching Medium `Griefing`.

The recommended repair is to make approval and rejection bind an expected immutable round:

```solidity
function approveAssessment(uint256 incidentId, uint256 expectedRoundId) external;
```

Revert unless `expectedRoundId == incident.currentAssessmentRoundId`, then load that exact round. Submitted rounds cannot be edited, so the round id is a sufficient binding. Including the submitted total and/or a canonical hash of the loss list in the signed calldata would make the reviewed object explicit to Safe tooling as defense in depth.

Apply the same binding to `rejectCurrentAssessment`; stale rejection is less damaging but has the same root cause.

### References

* Frozen `IncidentManager`: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol>
* Frozen `IIncidentManager`: <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/interfaces/IIncidentManager.sol>
* Official Safe 1.4.1 source: <https://github.com/safe-fndn/safe-smart-account/blob/v1.4.1/contracts/Safe.sol>
* Safe transaction concepts: <https://docs.safe.global/advanced/smart-account-concepts>
* Firelight documentation: <https://docs.firelight.finance/>

### Link to Proof of Concept

<https://gist.github.com/AnrokX/856afc13847aa70c8c06c20c5f3cb362>

### Proof of Concept

## Proof of Concept

gist: <https://gist.github.com/AnrokX/856afc13847aa70c8c06c20c5f3cb362>

### Target

* Repository: `https://github.com/immunefi-team/audit-comp-firelight`
* Checkout commit: `42f9ea5e43d88b35197b901acc2a8c24b314fa62`
* Scoped path: `contracts/core/IncidentManager.sol`

### Overview

The patch adds a Hardhat exploit/control test. It deploys the frozen FirelightVault, CoverOrderAllocator, CoverNFT, and IncidentManager, then wires an official Safe 1.4.1 singleton/proxy as the assessment approver, rejecter, and invalidator. The Safe has five distinct owners and a threshold of three.

Three owners sign `approveCurrentAssessment(1)` while assessment round 1 contains a reviewed 100-unit loss. The curator cancels it and submits round 2 with a distinct 1,000-unit loss. The Safe transaction hash and nonce remain unchanged. A relayer that is neither a Safe owner nor an IncidentManager approver executes the three signatures; round 2 is approved, the funded buffer pays 250, and the real vault pays 750. The control executes without replacement and pays only the reviewed 100.

### Run

Prerequisites: Bash, Git, Node.js, npm, and an internet connection for repository/dependency download. No RPC URL is needed.

Download all four Gist files into the same local directory, then run:

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
git -C audit-comp-firelight checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
bash /path/to/downloaded/run.sh /path/to/audit-comp-firelight
```

The runner rejects the wrong revision, any target `.env`, or unexpected checkout changes; verifies both patch and resulting PoC hashes; performs a lockfile install including development dependencies; installs exact Safe version `1.4.1-2`; pins every RPC, fork, chain-id, deployment-key, and execution-key variable consumed by the Hardhat configuration to empty; explicitly selects the repository's local config and `hardhat` network; and runs the one test file on an ephemeral local chain.

### Component And Mock Boundary

Core behavior is not mocked:

* frozen `FirelightVault`;
* frozen `CoverOrderAllocator`;
* frozen `CoverNFT`;
* frozen `IncidentManager` and its real payout path; and
* official Safe 1.4.1 singleton/proxy with three real owner signatures.

Only external setup dependencies are local substitutes. The repository's contextual FAsset implementation is used as the vault asset; its test fixture uses a local `MockContract` asset-manager address and Hardhat impersonation solely to mint setup balances. A plain local ERC20 represents the premium/first-loss-buffer token, and a fixed local aggregator supplies a fresh $1 price. None implements, intercepts, or overrides the vulnerable assessment lifecycle, Safe authorization, allocator settlement, or vault payout.

### Safety

* No mainnet or public-testnet transaction.
* No fork and no RPC call.
* No broadcast mode.
* No live or externally supplied private key or privileged account. The Safe-owner keys are disposable local keys generated solely inside the isolated test.
* No third-party contract or service is tested; network access is only for downloading source and npm packages.
* All state changes occur inside an ephemeral local Hardhat network.
