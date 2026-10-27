# 92222 w a critical portal remove user leaves root text authority allowing a former delegate to overwrite another owner s profile

**Submitted on Sep 8th 2026 at 10:18:59 UTC by @Pig46940 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92222
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (reflected), such as:
* Reflected HTML Injection
* Loading external site data

## Description

## Brief/Intro

Portal merges a delegate's root-scoped and name-scoped resolver roles into a single row. When the name owner selects **Remove user**, Portal warns that the user will be removed “from all roles,” but it constructs only one name-scoped revocation using the union of the displayed role bits.

As a result, the name-scoped role is removed while the root-scoped `ROLE_SET_TEXT` remains active and continues to apply to the name through root inheritance.

After the owner completes the removal flow, the former delegate can independently connect their own wallet and use Portal's production record-save flow to replace the victim name's description. The modified text is stored in resolver state and is subsequently rendered by Portal to fresh visitors.

This maps to the published High impact:

> Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
>
> * HTML injection without JavaScript
> * Replacing existing text with arbitrary text
> * Arbitrary file uploads, etc.

This report claims persistent plain-text replacement, not HTML injection or XSS.

Selected asset: **Explorer app Files** (`apps/portal` in the audited repository).

## Vulnerability Details

The affected role configuration contains two permissions for the same delegate:

* root-scoped `ROLE_SET_TEXT`;
* name-scoped `ROLE_SET_ADDR` for `persistent-content-victim.eth`.

Portal groups resolver-role records by account and combines their decoded role names into one displayed row:

* [ResolverRolesTable.tsx lines 90-104](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx#L90-L104)
* [resolverRoles.ts lines 168-220](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts#L168-L220)

The sidebar then chooses one non-root name for the grouped row and combines it with the union of all decoded roles:

* [ResolverRolesSidebar.tsx lines 165-173](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L165-L173)
* [ResolverRolesSidebar.tsx lines 307-322](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L307-L322)

When the owner chooses **Remove user**, Portal displays:

> Are you sure you want to remove this user from all roles?

See [ResolverRolesSidebar.tsx lines 441-463](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L441-L463).

However, the pending action contains the selected non-root name together with the unioned role bits. `revokeResolverRoles` interprets a non-empty name as a name-scoped revocation and submits only one request:

* [ResolverRolesSidebar.tsx lines 467-493](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx#L467-L493)
* [revokeResolverRoles.ts lines 29-74](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L29-L74)
* [revokeResolverRoles.ts lines 84-117](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts#L84-L117)

For the demonstrated mixed-scope row, Portal submits:

```
authorizeNameRoles(
  dns("persistent-content-victim.eth"),
  ROLE_SET_ADDR | ROLE_SET_TEXT,
  Former Delegate,
  false
)
```

This clears the name-scoped roles but does not revoke the root-scoped `ROLE_SET_TEXT`.

After the transaction:

```
name-scoped roles = 0
root-scoped roles = ROLE_SET_TEXT (16)
```

The owner has completed Portal's explicit “remove ... from all roles” flow, but the former delegate still retains effective text-record authority for the name.

### Exploit sequence

1. The name owner uses Portal to grant the delegate root-scoped `ROLE_SET_TEXT`.
2. The owner separately grants the same delegate name-scoped `ROLE_SET_ADDR` for `persistent-content-victim.eth`.
3. Portal groups both permissions into one delegate row.
4. The owner chooses **Edit user roles** → **Remove user**.
5. Portal warns that the delegate will be removed “from all roles.”
6. The owner confirms the removal and submits the Portal-generated transaction.
7. The name-scoped role is removed, but root `ROLE_SET_TEXT` remains active.
8. The owner takes no further action.
9. The former delegate connects their own wallet and uses Portal's production record-save flow to replace the name's description.
10. Fresh Portal visitors load the profile and see the attacker-controlled text.

Portal's production record-save flow constructs and submits the mutation here:

* [saveRecords.ts lines 97-149](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/saveRecords.ts#L97-L149)
* [saveRecords.ts lines 152-171](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/saveRecords.ts#L152-L171)

Fresh visitors retrieve and render the modified description through Portal:

* [NameProfileCard.tsx lines 37-73](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/components/NameProfileCard.tsx#L37-L73)
* [NameProfileCard.tsx lines 115-120](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/components/NameProfileCard.tsx#L115-L120)

No race, stale visitor cache, crafted URL, or additional owner approval is required.

### PoC results

The PoC uses separate owner and former-delegate accounts.

The owner creates both roles through mounted production Portal UI and then performs the production **Remove user** flow. The exact Portal-generated removal transaction is sent and mined unchanged.

The resulting role state is:

```
root role after removal: 16
name role after removal: 0
```

The former delegate then connects independently with their own wallet and uses Portal's production `saveRecords` path to set:

```
Former delegate controls this profile
```

Two fresh empty QueryClient instances independently load the profile through Portal's normal ENSjs / Universal Resolver query path and render the modified description.

A control then correctly revokes the remaining root role. After that revocation, another write by the former delegate reverts, confirming that the retained root role was what enabled the successful modification.

The attacker does not need to construct a direct resolver transaction. The successful write is created by Portal's production record-save flow, sent through the audited transaction manager, persisted in resolver state, and rendered by Portal.

## Impact Details

The owner explicitly confirms Portal's **Remove user** flow after being told that the delegate will be removed “from all roles.” Portal nevertheless leaves root `ROLE_SET_TEXT` active.

The former delegate can then independently use Portal to replace the owner's profile description with attacker-selected text. That text persists in resolver state and is displayed to fresh Portal visitors.

This matches the published High impact for persistent modification of static application content, including:

> Replacing existing text with arbitrary text

The attacker was previously authorized, but that authorization was supposed to end when the owner completed Portal's explicit all-role removal flow.

The owner can later repair the profile and revoke the remaining root role, but that does not undo the already completed persistent modification.

A previous review of this transaction-building issue (#91318) assumed that concrete harm required the former delegate to call excluded ENS contracts directly. This PoC shows that no direct attacker-constructed resolver call is required: the mutation is performed through Portal's production record-save path and is rendered by Portal itself.

## Suggested Fix

1. Preserve the resource/scope association of each role when resolver roles are grouped for display.
2. When **Remove user** covers roles from multiple resources, construct a separate revocation for each represented resource.
3. Revoke root-scoped permissions using the root scope rather than applying their role bits to a name-scoped request.
4. Show the exact resources and roles that will be revoked before submission.
5. After completion, reread the effective role state and do not report successful all-role removal while any displayed permission remains.
6. Add a regression test where one account holds both a root-scoped and a name-scoped role.

## References

* [Resolver role grouping](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/lib/roles/resolverRoles.ts)
* [Remove-user UI and transaction construction](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx)
* [Role revocation helper](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/resolver/helpers/revokeResolverRoles.ts)
* [Portal record-save orchestration](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/records/helpers/saveRecords.ts)
* [Portal profile renderer](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/components/NameProfileCard.tsx)

## Link to Proof of Concept

<https://gist.github.com/HiroyukiNaito/515a1b4a2de6d5c70f3e66ade6de0e8f>

## Proof of Concept

## Proof of Concept

The PoC is provided through the following unlisted Secret Gist:

<https://gist.github.com/HiroyukiNaito/515a1b4a2de6d5c70f3e66ade6de0e8f>

Pinned Gist revision:

```
12d02408e0d37e4db56d26bff9dfca194fe689a9
```

It contains two executed files:

* `portal-remove-all-persistent-content.poc.test.tsx` mounts the production Portal Add User flow, resolver-role table/sidebar, remove confirmation, transaction modal, production record-save helper, transaction manager, and fresh `NameProfileCard` consumers.
* `vitest.config.mjs` resolves application imports to the fixed audited checkout.

The PoC deploys deterministic execution infrastructure from the fixed `contracts-v2` revision to a fresh loopback Anvil. The contracts provide the authoritative permission and persistence boundary; they do not replace Portal’s grouping, confirmation, transaction construction, record-save, or rendering code under test.

### Prerequisites

* Linux or WSL
* Node.js `22.16.0`
* Corepack/pnpm `10.27.0`
* Foundry (`forge` and `anvil`)
* Git

### Reproduction procedure

Open a terminal in an empty working directory. Run every command below in order in the same shell session. The PoC and its two source repositories will be downloaded into the current directory.

Set paths relative to the current directory:

```bash
export ROOT="$(pwd -P)"
export POC="$ROOT/portal-poc"
export TARGET="$ROOT/audit-comp-ens"
export CONTRACTS="$ROOT/contracts-v2"
```

Download the Secret Gist and pin its executed revision:

```bash
git clone \
  https://gist.github.com/HiroyukiNaito/515a1b4a2de6d5c70f3e66ade6de0e8f.git \
  "$POC"

git -C "$POC" checkout \
  12d02408e0d37e4db56d26bff9dfca194fe689a9
```

Download the audited application and pin the competition revision:

```bash
git clone \
  https://github.com/immunefi-team/audit-comp-ens.git \
  "$TARGET"

git -C "$TARGET" checkout \
  1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

Download `contracts-v2`, pin the tested revision, and initialize its submodules:

```bash
git clone \
  https://github.com/ensdomains/contracts-v2.git \
  "$CONTRACTS"

git -C "$CONTRACTS" checkout \
  8d1c89350729f87b3968cf41ef3d683951444a44

git -C "$CONTRACTS" submodule update --init --recursive
```

Install the audited repository’s locked dependencies:

```bash
cd "$TARGET"
corepack pnpm install --frozen-lockfile
cd "$ROOT"
```

Build the fixed contract artifacts:

```bash
cd "$CONTRACTS/contracts"

forge build \
  --use 0.8.27 \
  --optimize true \
  --optimizer-runs 1000 \
  --evm-version prague \
  --use-literal-content

cd "$ROOT"
```

Run the PoC:

```bash
ENS_PORTAL_POC_TARGET="$TARGET" \
ENS_PORTAL_POC_CONTRACTS_OUT="$CONTRACTS/contracts/out" \
ENS_PORTAL_POC_ANVIL="$(command -v anvil)" \
"$TARGET/apps/portal/node_modules/.bin/vitest" run \
  --config "$POC/vitest.config.mjs" \
  "$POC/portal-remove-all-persistent-content.poc.test.tsx"
```

### Expected result

```
Test Files  1 passed (1)
Tests       1 passed (1)
```

The evidence output should include:

```yaml
classification: CONFIRMED FULL PORTAL E2E PERSISTENT STATIC-CONTENT END STATE
owner: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
formerDelegate: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
name: persistent-content-victim.eth
removeAll:
  rootRoleAfter: "16"
  nameRoleAfter: "0"
portalRecordSave:
  description: Former delegate controls this profile
freshVisitorRenders: 2
correctRevokeBlocksFurtherWrites: true
externalAttempts: []
```

All state-changing transactions occur exclusively inside the local loopback Anvil instance.
