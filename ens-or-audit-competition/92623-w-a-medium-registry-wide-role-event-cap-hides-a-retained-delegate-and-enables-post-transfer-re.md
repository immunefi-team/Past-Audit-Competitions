# 92623 w a medium registry wide role event cap hides a retained delegate and enables post transfer resolver and subregistry hijacking

**Submitted on Sep 11th 2026 at 02:07:57 UTC by @Ailenia for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92623
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The Portal reconstructs a name's role holders from only the newest 1,000 registry-wide `EACRolesChanged` events and filters by name afterward. A malicious seller can retain `ROLE_SET_RESOLVER` and `ROLE_SET_SUBREGISTRY` in an alternate account across a transfer, push the grant outside that window with one state-neutral self-batch on another name, and make the recipient's role page report `No role holders yet`. Because the missing row is the only edit/revoke control for that delegate, the alternate account can reattach both attacker-controlled pointers. The PoC executes this hijack to show the vulnerability.

## Vulnerability Details

The affected role query is in `../ens/repos/audit-comp-ens/apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts`.

`MAX_EVENTS` is fixed at 1,000, and the GraphQL request selects the newest 1,000 role events by `contractAddress`, which is the shared `.eth` registry rather than the requested name's resource:

```ts
const MAX_EVENTS = 1000

events(
  where: {
    type: "EACRolesChanged"
    contractAddress: $contractAddress
  }
  first: $first
  orderBy: blockNumber
  orderDirection: desc
)
```

Only after receiving this bounded registry-wide window does the client discard events for other names at lines 99-102:

```ts
for (const event of events) {
  const data = JSON.parse(event.data) as EACRolesChangedPayload
  if (data.resource.toLowerCase() !== resource.toLowerCase()) continue
  // ...
}
```

The client does not paginate or determine whether the replay is complete. An empty result is still a successful GraphQL result, so line 139 returns it and the on-chain fallback at lines 141-150 never runs:

```ts
if (indexerResult.isOk()) return ok(indexerResult.value)
```

The incomplete map reaches `../ens/repos/audit-comp-ens/apps/portal/src/features/roles/components/RolesTable.tsx:48-68`. When it is empty, the component renders `No role holders yet` and does not create the edit action that leads to role revocation.

This is cheaply attacker-controllable. One owner can call `safeBatchTransferFrom(owner, owner, ids, values, "0x")` with 500 duplicate entries for one attacker-owned name. The official registry emits two `EACRolesChanged` events per entry while the final ownership and role bitmap remain unchanged. The measured transaction consumes approximately 7.39 million gas and gives Panoptes exactly 1,000 newer events in one block and one transaction.

The transfer flow makes the omission security-relevant. `../ens/repos/audit-comp-ens/apps/portal/src/features/transfer/components/SendNameForm.tsx:36-73` enables resolver and subregistry detachment by default and describes both as leaving the recipient clean. `../ens/repos/audit-comp-ens/apps/portal/src/features/transfer/utils/buildTransferPlan.ts:22-42` detaches those pointers and transfers the token, but does not enumerate or revoke third-party grants. The official registry transfers the seller's own role bitmap to the recipient while an alternate delegate's resource-scoped grant remains active. The role panel is the Portal control that exposes that surviving grant and its revoke action; the event cap lets the attacker remove both at will after observing the transfer.

The attack is:

1. The malicious seller owns `sale.eth` and `pump.eth` and grants an alternate attacker account `ROLE_SET_RESOLVER` and `ROLE_SET_SUBREGISTRY` on `sale.eth`.
2. The seller transfers `sale.eth` to the victim through the Portal with the default detach options. The victim is the correct owner, both pointers are zero, and both third-party roles remain active.
3. After observing the transfer, the attacker sends one 500-entry self-batch transfer for `pump.eth`, producing 1,000 newer role events without changing its state.
4. The victim opens the role-management page for `sale.eth`. The production query filters the 1,000 `pump.eth` events and returns an empty map; `RolesTable` says `No role holders yet` and exposes no edit/revoke action.
5. The hidden delegate calls `setResolver` and `setSubregistry` for `sale.eth`, reattaching attacker-controlled pointers to the victim-owned name.

## Impact Details

During the transfer lifecycle, an attacker-selected alternate account retains pointer-management authority that the recipient's in-scope interface hides, then replaces both pointers after ownership moves. The attacker controls the name's resolution and subregistry until the victim independently detects the changes and recovers.

Attacker agency is not limited to accidental traffic. The attacker selects the delegate and pump name, observes the transfer, and hides the delegate with one transaction. The victim only receives the name and checks its role page. Ownership does not move to the attacker, the grants are limited to pointer management, and the owner can recover after detection; those facts bound but do not prevent the hijack.

The root cause is entirely in the in-scope Portal query and its fail-open consumption of a truncated result. The official ENS v2 registry and Panoptes behave correctly and are used only to demonstrate reachability. This is also within the stated new-finding boundary of Known Issue QA-01, which says that a demonstrated role combination granting unintended authority would be new.

The live scope names `apps/portal` as the `Explorer app Files` asset. No excluded smart-contract or indexer defect is alleged: the Portal requests and consumes a valid but insufficient prefix from a correct producer.

## Existing Safeguards and triage FAQ

* **"The delegate already had the roles."** That pre-transfer authorization is not authorization from the recipient. The application's security function is to enumerate surviving authority so the new owner can revoke it. The differential control proves causation: when the row is present, production revocation removes both roles and both writes fail; only the bounded replay removes the control and lets the writes succeed.
* **"Third-party roles intentionally survive transfer."** The report does not claim the registry transfer is defective. Intended retention makes the Portal's post-transfer authority inventory load-bearing. The transfer form cleans the direct pointers, while the separate role page is the product control for auditing and revoking surviving delegates.
* **"The owner can recover."** On-chain admin authority remains a recovery safeguard, but it requires independently discovering an account the Portal says does not exist. Recovery is reactive and does not prevent the completed resolver change. The report therefore does not claim ownership theft or permanent bricking.
* **"The indexer fallback protects the page."** The fallback runs only when GraphQL throws. The attack returns a syntactically successful empty result, so it never executes. The paginated history view can still find the grant, but that view is not used to populate the role-management rows or their revoke actions.
* **"This is a traffic-volume attack."** The carrier is one valid on-chain transaction on an attacker-owned name, not automated traffic against the Portal or indexer. No denial-of-service request flood is required.

## Duplicate Check

Reports 89614 and 89253 are about account-level aggregation of multiple name resources into one row, followed by an incomplete one-name revoke. This report uses one name and one resource and excludes the delegate before any revoke request can be constructed. Grouping by `(account, resource)` fixes `#89253` but leaves this 1,000-event prefix unchanged; resource-scoped pagination or completeness fallback fixes this report but does not fix cross-resource grouping. I couldn't find a published Known Issue describing the registry-wide event cap, and QA-01 preserves demonstrated unintended authority as new.

## Remediation

Do not reconstruct per-name authority from a fixed registry-wide prefix. Prefer a server-side query scoped by both registry and resource. If that is unavailable, paginate until the requested resource's current account states are provably reconstructed, using a stable cursor that includes block number and log identity so more than 1,000 events in one block cannot be skipped. If completeness cannot be established, fail closed and use the existing on-chain `getNameRoleAccounts` fallback instead of returning an empty success.

As defense in depth, the transfer flow should enumerate outstanding third-party roles and require their revocation or present an explicit warning before claiming that the recipient starts clean. Add a regression test where the target grant is older than 1,000 unrelated registry events, including 1,000 events emitted in one transaction.

## References

* `../ens/repos/audit-comp-ens/apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts:50-150`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/roles/components/RolesTable.tsx:48-68`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/roles/components/NameRolesOverviewTable.tsx:30-59`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/transfer/components/SendNameForm.tsx:36-73`
* `../ens/repos/audit-comp-ens/apps/portal/src/features/transfer/utils/buildTransferPlan.ts:22-42`

## Link to Proof of Concept

<https://gist.github.com/niafreu/648d2c65c5b5e33e2c2ec9bc101f9804>

## Proof of Concept

## Proof of Concept

The three-file PoC is available at `https://gist.github.com/niafreu/648d2c65c5b5e33e2c2ec9bc101f9804` and runs entirely against loopback. It starts a fresh Anvil chain, deploys the official ENS v2 `LabelStore` and `PermissionedRegistry` from `ensdomains/contracts-v2` commit `48b3e2d39513b9dd32ef1850877a29009bc807b9`, starts Panoptes pinned by image digest, and imports the production Portal code from the audited commit.

### Prerequisites

* Linux, Git, Docker, Node.js, Corepack/pnpm, and Foundry (`forge` and `anvil`).
* Save the three attached files in a directory named `poc`.

### Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
git -C audit-comp-ens checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
(
  cd audit-comp-ens
  corepack pnpm install --frozen-lockfile
)

git clone --recurse-submodules https://github.com/ensdomains/contracts-v2.git
git -C contracts-v2 checkout 48b3e2d39513b9dd32ef1850877a29009bc807b9
git -C contracts-v2 submodule update --init --recursive
```

### Run

From the directory containing `audit-comp-ens`, `contracts-v2`, and `poc`:

```bash
AUDIT_REPO_DIR="$PWD/audit-comp-ens" \
CONTRACTS_V2_DIR="$PWD/contracts-v2" \
audit-comp-ens/apps/portal/node_modules/.bin/vitest run \
  --disableConsoleIntercept --reporter=verbose --config poc/vitest.config.ts
```

Expected result: `2 passed (2)`.

The first test renders the production transfer form and verifies its default clean-transfer options. The second performs the full control and exploit against the real local producer and official contracts. The decisive output is:

```
portalMessage: "No role holders yet"
roleStillActiveOnChain: true
resolverAfterExploit: "0x000000000000000000000000000000000000bEEF"
subregistryAfterExploit: "0xdc64a140aa3e981100a9beca4e685f962f0cf6c9"
ownerAfterExploit: "0x70997970C51812dc3A010C7d01b50e0d17dc79C8"
```

The negative control uses the production Portal revocation helper while the delegate is visible and verifies that both pointer writes revert. The exploit branch uses the production transfer helpers, actual Panoptes ingestion, production role query, real `RolesTable`, and successful delegate transactions. No indexer row or contract state is staged, and no ENS deployment or public RPC is contacted during execution.

### What is directly observed versus staged

| Element                                                                                    | Treatment                                                                                                                                         |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| ENS v2 role retention, transfer, revocation, event emission, and both pointer replacements | Directly executed against official contracts on fresh Anvil                                                                                       |
| Grant, detach-resolver, detach-registry, transfer, and control revocation                  | Directly submitted through production Portal helpers and transaction manager                                                                      |
| Empty authority result and UI consequence                                                  | Directly produced by the production hook and rendered by the real `RolesTable`                                                                    |
| Users and names                                                                            | Local test identities and registrations, because programme rules require local/testnet proof                                                      |
| Wallet transport for the role query                                                        | `safeGetClient` is bound to the loopback viem client; no query, filtering, fallback, or UI logic is replaced                                      |
| Transfer-form surroundings                                                                 | Async address and detach-target hooks are deterministic test doubles; the real component, option defaults, and security descriptions are rendered |
