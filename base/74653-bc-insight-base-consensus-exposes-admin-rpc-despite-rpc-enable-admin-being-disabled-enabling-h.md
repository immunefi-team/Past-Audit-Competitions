# 74653 bc insight base consensus exposes admin rpc despite rpc enable admin being disabled enabling hidden unsafe branch partition via admin postunsafepayload

Submitted on Apr 24th 2026 at 03:04:10 UTC by @cgdusek for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74653
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

### Vulnerability Report

Selected fields:

* program: `Base Azul`
* asset: `Offchain Components`
* impact: `High: Unintended chain split (network partition)`
* title: `base-consensus exposes admin RPC despite --rpc.enable-admin being disabled, enabling hidden unsafe-branch partition via admin_postUnsafePayload`

### Executive summary of issue

`base-consensus` parses `--rpc.enable-admin` / `BASE_NODE_RPC_ENABLE_ADMIN`, but the normal node RPC actor does not enforce that flag before merging the `admin_*` namespace. A caller who can reach the normal sequencer RPC socket can therefore use `admin_postUnsafePayload` even when the operator did not enable admin RPC. The combined PoC proves the end-to-end consequence on the pinned release: a hidden valid unsafe payload can be injected onto a poisoned subset, the active sequencer then builds descendants on that hidden branch, a second poisoned peer follows it, and a clean validator remains on the public head because it never observed the hidden parent. That is a proved unintended unsafe-head network partition.

### Classification and scope justification

#### Exact program criteria match

* asset name(s) exactly as listed on scope page: `Offchain Components`
* specific scoped asset(s) supporting this severity claim: Base-native `base-consensus` offchain code in `base/base` under `crates/client`, `crates/consensus/rpc`, `crates/consensus/service`, `crates/consensus/engine`, and `crates/consensus/gossip`
* exact current Immunefi scope picker URL(s) supporting this severity claim: the live scope page did not expose separate per-asset picker URLs; use [Base Azul scope](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/)
* online form asset picker selection: `Offchain Components`
* exact current Immunefi scope picker URL for selected form asset: the live scope page does not expose a separate per-asset picker URL; use the [Base Azul scope page, checked 2026-04-23](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/)
* selected severity from scope page: `High`
* selected impact title from scope page: `Unintended chain split (network partition)`
* exact scope wording: the live scope lists `Offchain Components` as an asset and says only Base-native code such as CL and EL components is in scope; it also lists `actions`, `devnet`, `baseup`, and `etc` as out of scope, so this report keeps the bug claim on Base-native `crates/` code only
* scope page last checked: `2026-04-23`
* why this matches the demonstrated consequence: the combined PoC proves that the disabled-admin bypass exposes `admin_postUnsafePayload`, that a valid hidden payload can be injected into a poisoned subset, that the active sequencer builds descendants on that hidden branch, and that a clean validator remains on a different branch because the hidden parent never reached it. That is the listed High network-partition lane.

#### Why higher or adjacent criteria do not fit

* closest higher severity criterion considered: `Unintended permanent chain split requiring hard fork (network partition requiring hard fork)`
* why the report does not satisfy it: the PoC proves unsafe-head divergence on a poisoned subset. It does not prove permanence, safe/finalized divergence, hard-fork necessity, or that manual restart/reconfiguration cannot restore convergence. The live program also says reports that assume a service will not be manually restarted with possibly different configurations may be downgraded.
* closest adjacent same/lower criteria considered: `Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments`
* why the selected criterion fits better: the selected proved consequence is not just a stalled sequencer. The combined artifact shows two validator views diverging onto different unsafe branches.

#### Feasibility boundary

* proven effect: unauthorized access to the disabled admin namespace plus `admin_postUnsafePayload` can create a local unsafe-head network partition between a poisoned subset and a clean validator
* proven technical requirements inside the PoC: the attacker supplies a fully valid hidden payload extending the shared public head, injects it through the exposed admin route, and the active sequencer then builds descendants on that hidden branch
* bounded-inference deployment precondition: the attacker can reach the normal sequencer RPC socket in deployment. This report does not claim a probed live public endpoint.
* unsupported / not claimed escalation path: this packet does not claim permanent or finalized chain split, hard-fork-required partition, invalid L1 finalization, bridge loss, or direct theft

#### Out-of-scope exclusion check

* relevant out-of-scope exclusions reviewed: the live scope excludes `actions`, `devnet`, `baseup`, and `etc`; it excludes impacts requiring leaked keys, privileged addresses, mainnet or public testnet testing, phishing, and denial-of-service attacks executed against project assets
* excluded assumptions not used by this exploit: no leaked key, privileged address, governance action, TEE/ZK proof compromise, invalid state-root proof, `etc` deployment file, mainnet testing, public testnet testing, live endpoint probing, or attack traffic against project assets is required
* PoC harness boundary: the reviewer test is copied into `actions/harness` only as local scaffolding inside a fresh pinned clone; the bug claim and the asserted runtime paths remain in in-scope `crates/` code

## Finding details

### Public documentation invariant

Public evidence checked on `2026-04-23`:

* The live [Base Azul scope page, checked 2026-04-23](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/) lists `Offchain Components` as an in-scope asset and identifies Base's migration to `base-reth-node` and `base-consensus` as a main Azul change.
* The official [Base Azul upgrade guide, checked 2026-04-23](https://docs.base.org/base-chain/node-operators/base-v1-upgrade) says most `base-consensus` configuration is preconfigured in `base/node` `.env.mainnet` and `.env.sepolia`, and it maps `OP_NODE_RPC_ENABLE_ADMIN` to `BASE_NODE_RPC_ENABLE_ADMIN`.
* In release-pinned source, [`RpcArgs` labels `--rpc.enable-admin` / `BASE_NODE_RPC_ENABLE_ADMIN` as "Enable the admin API"`](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/client/cli/src/rpc.rs#L24-L32), and [`RpcArgs::build`carries`enable\_admin`into`RpcBuilder\`](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/client/cli/src/rpc.rs#L64-L78).
* The stock [`base/node` Sepolia template at commit `ca91ddb0683d6ebde184e96d3515863e80bfb918` lines 24-95](https://github.com/base/node/blob/ca91ddb0683d6ebde184e96d3515863e80bfb918/.env.sepolia#L24-L95) preconfigures `base-consensus` RPC host/port settings but does not explicitly set `BASE_NODE_RPC_ENABLE_ADMIN`.
* The analogous [OP Stack node guide, checked 2026-04-23](https://docs.optimism.io/node-operators/tutorials/run-node-from-source) warns that the rollup-node RPC should not be exposed publicly because it could expose admin controls.
* The release-pinned admin namespace includes operational methods such as [`postUnsafePayload`, `startSequencer`, `stopSequencer`, `setRecoverMode`, `overrideLeader`, and `resetDerivationPipeline` in `AdminApi` lines 162-204](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/rpc/src/jsonrpsee.rs#L162-L204).

The resulting invariant is inference-backed rather than stated in one sentence: the normal `base-consensus` RPC listener should expose the `admin_*` namespace only when the operator explicitly enables the admin API. This bug breaks that boundary because [`RpcActor::start`](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/rpc/actor.rs#L101-L123) merges `AdminRpc` whenever normal startup supplies `network_admin`, even when `--rpc.enable-admin` is unset. The current PoC then proves that this unauthorized admin surface is strong enough to create an unsafe-head network partition through `admin_postUnsafePayload`, not just to expose dormant admin methods in theory.

### Root cause

`RpcBuilder` contains an `enable_admin` field, as shown in [`crates/consensus/rpc/src/config.rs` lines 7-24](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/rpc/src/config.rs#L7-L24), but the registration path ignores it. In [`crates/consensus/service/src/actors/rpc/actor.rs` lines 101-123](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/rpc/actor.rs#L101-L123), `RpcActor::start` merges `AdminRpc` whenever `network_admin` is `Some`. It checks other optional modules later in the same function, but it never checks `enable_admin` before merging the admin namespace.

Normal node startup supplies that `network_admin` channel in [`crates/consensus/service/src/service/node.rs` lines 534-555](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/service/node.rs#L534-L555). The exposed `admin_postUnsafePayload` route forwards an unsigned [`BaseExecutionPayloadEnvelope` that contains no signature field in lines 17-25](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/common/rpc-types-engine/src/envelope.rs#L17-L25) through [`AdminRpc::admin_post_unsafe_payload` lines 70-80](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/rpc/src/admin.rs#L70-L80) into [`NetworkActor` forwarding lines 203-207](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/network/actor.rs#L203-L207). The normal gossip path, by contrast, requires a signed [`NetworkPayloadEnvelope` in lines 239-249](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/common/rpc-types-engine/src/envelope.rs#L239-L249) and validates the unsafe block signer in [`BlockHandler` lines 218-229](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/gossip/src/block_validity.rs#L218-L229).

### Strongest demonstrated exploit path

{% stepper %} {% step %}

## Attacker reaches the RPC socket

The attacker reaches the normal sequencer node HTTP RPC socket. The release CLI defaults that RPC socket to `0.0.0.0:9545` in [`crates/client/cli/src/rpc.rs` lines 24-32](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/client/cli/src/rpc.rs#L24-L32), but this report keeps live reachability as an exploit precondition rather than a proved public exposure. {% endstep %}

{% step %}

## The disabled-admin namespace is exposed

The attacker uses the exposed disabled-admin namespace to invoke `admin_postUnsafePayload` on nodes that should not have admin RPC at all. The route is exposed because [`RpcActor::start` merges `AdminRpc` without checking `enable_admin`](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/rpc/actor.rs#L101-L123). {% endstep %}

{% step %}

## A hidden valid payload is injected

The attacker injects a fully valid hidden payload extending the shared public head into the target sequencer and one peer through the admin route. The PoC proves the method accepts that hidden payload and advances the poisoned subset to the hidden head. {% endstep %}

{% step %}

## The sequencer builds descendants on the hidden head

The active sequencer then builds the next block from the hidden head. That behavior follows the current unsafe head in [`PayloadBuilder::build` lines 56-63](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/sequencer/build.rs#L56-L63). {% endstep %}

{% step %}

## The branches diverge

A second poisoned peer that observed the hidden ancestor follows the descendant, while a clean validator that only saw the public head rejects the descendant because the hidden parent was never observed. The combined PoC proves the poisoned subset and clean validator end on different branches. {% endstep %} {% endstepper %}

### PoC-backed proof chain

{% stepper %} {% step %}

## `verify_admin_rpc_gate.py`

`verify_admin_rpc_gate.py` proves that the release-pinned code parses `--rpc.enable-admin` but still exposes the admin namespace without checking that flag. {% endstep %}

{% step %}

## `admin_post_unsafe_payload_partition.rs`

`admin_post_unsafe_payload_partition.rs` is injected into a fresh pinned clone and run through `cargo test` to prove the partition consequence. {% endstep %} {% endstepper %}

Observed high-signal markers from the combined artifact:

* `POC_MARKER_FLAG_PARSED`
* `POC_MARKER_ADMIN_MERGED_WITHOUT_ENABLE_ADMIN`
* `POC_MARKER_PUBLIC_HEAD_ESTABLISHED`
* `POC_MARKER_ADMIN_POST_UNSAFE_PAYLOAD_POISONS_SUBSET`
* `POC_MARKER_POISONED_SEQUENCER_BUILDS_DESCENDANT`
* `POC_MARKER_CLEAN_VALIDATOR_REJECTS_MISSING_PARENT_DESCENDANT`
* `POC_MARKER_NETWORK_PARTITION`
* `POC_RESULT: PASS`

The live secret gist [`ceb8c60b5af8e2e486d4299d8a4c0b5f`](https://gist.github.com/cgdusek/ceb8c60b5af8e2e486d4299d8a4c0b5f) was rechecked against the local mirror on `2026-04-23`. The exact fresh AWS Spot rerun used gist revision `cb6f5e8b0cb3f519e6b3ae0d211e00fd33453931` with the reviewer command `bash run_poc.sh`. The current live gist revision at the last congruence check is `37ba32dfeea10c18f12f8738a4834940efd1742e`, which only narrows the README wording to the validated Ubuntu reviewer environment and leaves the runnable files unchanged.

### PoC setup conditions

The PoC is local-only with respect to blockchain state. It clones the pinned `base/base` release, boots local harness actors, and never connects to Base mainnet, Base Sepolia, Ethereum, or any live RPC endpoint. On a stock root Ubuntu host, `run_poc.sh` bootstraps the Rust toolchain, Linux build packages, and Foundry when missing; on other environments, those requirements must be preinstalled as described in the gist README.

### Exploit preconditions

* The normal sequencer node's HTTP RPC socket is reachable by the attacker.
* The node is running the affected `base-consensus` code path.
* The attacker can construct a fully valid payload on top of the current public unsafe head. The PoC proves this technically within the pinned local environment and does not require a privileged signer because the admin route bypasses the normal gossip signature gate.

No leaked key, privileged address, governance vote, TEE signer compromise, ZK soundness failure, invalid proof, public-network transaction, or public testnet/mainnet testing is required.

### Deployment and in-scope surface

The bug-relevant surface is Base-native `base-consensus` code under `crates/`, which falls under the live `Offchain Components` asset. The PoC pins `base/base` commit `de349fc9e8bf61531ce36ca57572345b03b2b097`, tagged locally as `v0.8.0-rc.15`. This report does not claim that a specific public Base Sepolia RPC endpoint is internet-exposed; it claims the release-pinned scoped code exposes admin RPC on any normal sequencer RPC socket reachable by an attacker.

That reachability precondition is deployment-specific and is presented as a bounded inference rather than a proved live exposure. The release-pinned CLI defaults the `base-consensus` RPC to `0.0.0.0:9545` in [`crates/client/cli/src/rpc.rs` lines 24-32](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/client/cli/src/rpc.rs#L24-L32). Base's official migration guide tells node operators to use the preconfigured `base/node` setup and start it with `docker compose up` on the [Base Azul upgrade guide, checked 2026-04-23](https://docs.base.org/base-chain/node-operators/base-v1-upgrade). The analogous OP Stack `op-node` guide explicitly warns that the rollup-node RPC should not be exposed publicly because it could expose admin controls on the [OP node run guide, checked 2026-04-23](https://docs.optimism.io/node-operators/tutorials/run-node-from-source).

### Public-known / duplicate boundary

The public known-issues document linked from the live scope lists SP1, ZK service, Nitro/TEE, registrar, and proposer issues, but it does not describe this base-consensus admin RPC gate mismatch or an exposed-`admin_postUnsafePayload` partition path. Targeted GitHub issue/PR searches for `admin_stopSequencer`, `admin_postUnsafePayload`, `BASE_NODE_RPC_ENABLE_ADMIN`, `AdminRpc::new`, and `enable_admin` in `base/base` found no exact public issue or pull request describing this bug family.

## Repository, file, and line of code where finding is found

* [`crates/client/cli/src/rpc.rs` lines 24-32](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/client/cli/src/rpc.rs#L24-L32)
* [`crates/client/cli/src/rpc.rs` lines 64-78](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/client/cli/src/rpc.rs#L64-L78)
* [`crates/consensus/rpc/src/config.rs` lines 7-24](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/rpc/src/config.rs#L7-L24)
* [`crates/consensus/service/src/actors/rpc/actor.rs` lines 101-123](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/rpc/actor.rs#L101-L123)
* [`crates/consensus/service/src/service/node.rs` lines 534-555](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/service/node.rs#L534-L555)
* [`crates/consensus/rpc/src/jsonrpsee.rs` lines 162-204](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/rpc/src/jsonrpsee.rs#L162-L204)
* [`crates/consensus/rpc/src/admin.rs` lines 70-80](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/rpc/src/admin.rs#L70-L80)
* [`crates/consensus/service/src/actors/network/actor.rs` lines 203-207](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/network/actor.rs#L203-L207)
* [`crates/consensus/service/src/actors/sequencer/build.rs` lines 56-63](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/sequencer/build.rs#L56-L63)
* [`crates/consensus/gossip/src/block_validity.rs` lines 218-229](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/gossip/src/block_validity.rs#L218-L229)

## Steps to replicate

All testing was performed locally only. I did not use mainnet or public testnet.

* secret gist: [PoC gist](https://gist.github.com/cgdusek/ceb8c60b5af8e2e486d4299d8a4c0b5f)
* recommended PoC name: `admin-rpc-enable-admin-bypass-network-partition`

```bash
bash run_poc.sh
```

Expected high-signal output markers:

* `POC_MARKER_FLAG_PARSED`
* `POC_MARKER_ADMIN_MERGED_WITHOUT_ENABLE_ADMIN`
* `POC_MARKER_PUBLIC_HEAD_ESTABLISHED`
* `POC_MARKER_ADMIN_POST_UNSAFE_PAYLOAD_POISONS_SUBSET`
* `POC_MARKER_POISONED_SEQUENCER_BUILDS_DESCENDANT`
* `POC_MARKER_CLEAN_VALIDATOR_REJECTS_MISSING_PARENT_DESCENDANT`
* `POC_MARKER_NETWORK_PARTITION`
* `POC_RESULT: PASS`

## Impact of finding

### Short term

An attacker who can reach the normal sequencer RPC socket can use an admin method that should have been disabled to split validator views. The combined PoC proves a poisoned subset on one unsafe branch and a clean validator on another, which matches the live High network-partition lane.

### Long term

The long-term consequence is continued integrity and availability risk while the vulnerable RPC surface remains reachable. This report does not claim permanent or finalized chain split, hard-fork-required partition, bridge loss, invalid state-root finalization on L1, or direct theft of funds.

## Comparable public incidents

These are context only, not duplicates or direct proof of identical impact.

<details>

<summary>Security Advisory [Insecurely configured geth can make funds remotely accessible], August 29, 2015</summary>

[Ethereum Foundation blog](https://blog.ethereum.org/2015/08/29/security-alert-insecurely-configured-geth-can-make-funds-remotely-accessible)

Similar because remote reachability to an operator-gated RPC surface can convert node-local authority into attacker-controlled actions. Different because the impact there was fund access, not partition of validator views.

</details>

<details>

<summary>Security Alert - cpp-ethereum keeps accounts unlocked, May 31, 2016</summary>

[Ethereum Foundation blog](https://blog.ethereum.org/2016/05/31/security-alert-cpp-ethereum-keeps-accounts-unlocked)

Similar because exploitability depended on access to an exposed JSON-RPC surface that was not meant to grant blanket remote authority. Different because it was an account-unlock bug, not an admin namespace registration bug.

</details>

## Mitigation suggestions

* Gate `AdminRpc::new(...).into_rpc()` in `RpcActor::start` on `self.config.enable_admin`.
* Add a focused RPC actor regression test proving that `admin_*` methods are absent when `enable_admin=false`, even when `network_admin` is present.
* Add a regression test covering `admin_postUnsafePayload` specifically so it cannot be reached through the normal RPC surface when admin mode is disabled.

## Optional suggested patch

No untested code patch is included in this packet. The mitigation direction above is intentionally limited to the required gate and regression coverage.

## References

### Runnable PoC package

* secret gist: [PoC gist](https://gist.github.com/cgdusek/ceb8c60b5af8e2e486d4299d8a4c0b5f)
* recommended PoC name: `admin-rpc-enable-admin-bypass-network-partition`

### Live deployment / release confirmation

* scope: [Base Azul scope page, checked 2026-04-23](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/)
* information: [Base Azul information page, checked 2026-04-23](https://immunefi.com/audit-competition/audit-comp-base-azul/information/)
* release pin: [`base/base` commit `de349fc9e8bf61531ce36ca57572345b03b2b097`](https://github.com/base/base/tree/de349fc9e8bf61531ce36ca57572345b03b2b097)

### Relevant code paths

* [`RpcArgs` admin flag and defaults](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/client/cli/src/rpc.rs#L24-L32)
* [`RpcActor::start` admin module registration](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/rpc/actor.rs#L101-L123)
* [`AdminApi` method set including `postUnsafePayload`](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/rpc/src/jsonrpsee.rs#L162-L204)
* [`admin_postUnsafePayload` implementation](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/rpc/src/admin.rs#L70-L80)
* [`NetworkActor` admin forwarding](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/network/actor.rs#L203-L207)
* [`PayloadBuilder` uses the current unsafe head](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/sequencer/build.rs#L56-L63)
* [`BlockHandler` signer validation on the normal gossip path](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/gossip/src/block_validity.rs#L218-L229)

### Scope / rules

* PoC rules: [Immunefi PoC guidelines](https://immunefisupport.zendesk.com/hc/en-us/articles/9946217628561-Proof-of-Concept-PoC-Guidelines-and-Rules)

### Public protocol documentation

* [Base Azul scope page, checked 2026-04-23](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/)
* [Base Azul upgrade guide, checked 2026-04-23](https://docs.base.org/base-chain/node-operators/base-v1-upgrade)
* [OP node run guide, checked 2026-04-23](https://docs.optimism.io/node-operators/tutorials/run-node-from-source)
* [`RpcArgs` admin flag and env var](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/client/cli/src/rpc.rs#L24-L32)
* [`RpcActor::start` admin module registration](https://github.com/base/base/blob/de349fc9e8bf61531ce36ca57572345b03b2b097/crates/consensus/service/src/actors/rpc/actor.rs#L101-L123)

### Additional context / provenance

* [Ethereum Foundation geth JSON-RPC advisory, August 29, 2015](https://blog.ethereum.org/2015/08/29/security-alert-insecurely-configured-geth-can-make-funds-remotely-accessible)
* [Ethereum Foundation cpp-ethereum JSON-RPC alert, May 31, 2016](https://blog.ethereum.org/2016/05/31/security-alert-cpp-ethereum-keeps-accounts-unlocked)

## Submission notes

* selected scope criterion: `High - Unintended chain split (network partition)`
* specific scoped assets carrying the claim: `Offchain Components`, specifically Base-native `base-consensus` code under `crates/`
* online form asset picker selection: `Offchain Components`
* exact current Immunefi scope picker URL for selected form asset: the live scope page does not expose a separate per-asset picker URL; use the [Base Azul scope page, checked 2026-04-23](https://immunefi.com/audit-competition/audit-comp-base-azul/scope/)
* severity anchor: `admin_postUnsafePayload` hidden-branch partition on a poisoned subset
* breadth-only variant: `admin_stopSequencer` remains corroborating breadth from the same disabled-admin bug family, not the primary severity anchor
* explicitly not claiming: permanent or finalized chain split, hard-fork-required partition, public endpoint exposure, Base mainnet impact, invalid state-root finalization on L1, TEE/ZK proof bypass, bridge loss, or direct theft

## Link to Proof of Concept

<https://gist.github.com/cgdusek/ceb8c60b5af8e2e486d4299d8a4c0b5f>

## Proof of Concept

### PoC specification

Recommended PoC name:

* `admin-rpc-enable-admin-bypass-network-partition`

Full PoC package:

* [PoC gist](https://gist.github.com/cgdusek/ceb8c60b5af8e2e486d4299d8a4c0b5f)

One-sentence summary:

The PoC first proves that `base-consensus` exposes the `admin_*` namespace even when `--rpc.enable-admin` is disabled, then proves that `admin_postUnsafePayload` can seed a hidden valid unsafe branch that a poisoned subset follows while a clean validator remains on the public head.

Minimal file list:

* `README.md`
* `OUTPUT.txt`
* `run_poc.sh`
* `verify_admin_rpc_gate.py`
* `admin_post_unsafe_payload_partition.rs`

Run from a fresh clone or download of the gist:

```bash
bash run_poc.sh
```

Layout note:

GitHub gist stores all files at the gist root. Run the command from that root directory.

Expected high-signal output markers:

* `POC_MARKER_FLAG_PARSED`
* `POC_MARKER_ADMIN_MERGED_WITHOUT_ENABLE_ADMIN`
* `POC_MARKER_PUBLIC_HEAD_ESTABLISHED`
* `POC_MARKER_ADMIN_POST_UNSAFE_PAYLOAD_POISONS_SUBSET`
* `POC_MARKER_POISONED_SEQUENCER_BUILDS_DESCENDANT`
* `POC_MARKER_CLEAN_VALIDATOR_REJECTS_MISSING_PARENT_DESCENDANT`
* `POC_MARKER_NETWORK_PARTITION`
* `POC_RESULT: PASS`

Notes:

* The PoC uses only local execution and pinned public dependencies. It does not connect to Base mainnet, Base Sepolia, Ethereum, or any live node RPC endpoint.
* On a stock root Ubuntu host, `run_poc.sh` bootstraps the Rust toolchain, Linux build prerequisites, and Foundry when missing. Other environments should preinstall the requirements listed in the gist README.
* The full requirements and sample output live in the gist README and `OUTPUT.txt`.
