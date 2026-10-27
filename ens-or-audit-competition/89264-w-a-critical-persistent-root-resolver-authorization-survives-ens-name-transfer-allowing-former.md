# 89264 w a critical persistent root resolver authorization survives ens name transfer allowing former owners to rewrite sold name records and unrelated resolver resources

**Submitted on Aug 18th 2026 at 10:41:39 UTC by @sl4x for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89264
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Registration and migration grant the name owner **permanent ROOT roles on the per-HCA PermissionedResolver** (`authorizeNameRoles(0x00, ROLES_ALL, wallet, true)` in `packages/smart-account/src/providers/rhinestone/registration-calls.ts` step 6 and `apps/manager/src/features/migration/service/buildAtomicMigrationBatches.ts` `buildWalletCoAdminCall`). The portal transfer flow moves a name with a plain ERC-1155 `safeTransferFrom` and **never revokes the seller's grant nor provisions the buyer**. A former owner can therefore, at any time after selling, silently re-point the sold name's `addr` and text records at an attacker address and rewrite the records the Portal renders (avatar/description OG cards, name pages). The new owner receives no record-write authority on the retained resolver and has no resolver-level mechanism to revoke the seller's persistent ROOT grant.

## Vulnerability Details / Root Cause

1. **Wildcard root grant.** Registration reveal step 6 always appends `authorizeNameRoles(hex"00", ROLES_ALL, wallet, true)`. An empty `toName` (`0x00`) is `NameCoder.namehash("")` = the ROOT node = `ROOT_RESOURCE`. Migration repeats the identical grant.
2. **Root roles are universal.** `EnhancedAccessControl._effectiveRoles(resource, account) = _getRoles(ROOT_RESOURCE, account) | _getRoles(resource, account)` — root roles satisfy **every** resource-scoped check in the resolver (setters, `onlyPartRoles`, and root-only `setAlias` / `_authorizeUpgrade`). The wallet therefore holds every role on every node of its resolver: any name, subname, or unrelated node on the same per-HCA resolver, plus `ROLE_SET_ALIAS` (1<<28) and `ROLE_UPGRADE` (1<<124).
3. **Transfer never touches the role store.** `buildTransferPlan` emits only `set-eth-addr`/`detach-resolver`/`detach-registry`/`transfer-token`; `transferToken.ts` encodes a bare `erc1155Abi.safeTransferFrom`. No revoke of the seller's resolver roles, no grant to the buyer. `SendNameForm` defaults `detachResolver` ON but it is toggleable (keeping records is an advertised option) and hidden for owners lacking registry-side `ROLE_SET_RESOLVER` (migrated/locked names are forced to keep the resolver). Subname creation defaults the subname's resolver to the parent's per-HCA resolver, so subnames handed to third parties resolve through — and remain writable by — the parent's resolver.

## Impact

Matches the program's listed Critical impacts: "resolving or rendering an ENS name to an attacker-controlled address" and "Changing NFT metadata" (in ENS v2 the resolver text/addr records ARE the name's metadata).

* **Kept-resolver sale:** the seller later re-points the sold name's `addr` to an attacker address; anyone paying the name pays the attacker; the buyer cannot write records at all.
* **Subname hijack:** a subname granted/sold to a third party is silently re-pointed by the parent; the recipient's name resolves to the attacker.
* **Portal rendering / identity attachment:** the OG worker and name pages render attacker-supplied avatar/description.
* **Role-store despotism:** the seller can grant/revoke any account's roles at will (including roles the app may later grant the buyer); the buyer cannot revoke the seller, self-grant, re-initialize, or upgrade.

## Severity

**Critical** per the program's impact list — the PoC directly demonstrates "resolving or rendering an ENS name to an attacker-controlled address" after a real transfer (the seller's `setAddr` makes the sold name resolve to the attacker address; the portal OG card / name page render the seller-rewritten avatar/description). Permanent (never revoked, survives all future transfers), wildcard (any node on the resolver, proven on third-party subname and unrelated nodes), and no buyer-side remedy at the resolver level.

Honest caveat (documented, not a downgrade): for a plain kept-resolver sale the buyer CAN swap the resolver pointer via the registry `ROLE_SET_RESOLVER` that moves with the token — that is the only mitigation, requires the buyer to notice and act, and is unavailable to remedy-free populations: subnames whose granted registry role set lacks `ROLE_SET_RESOLVER`, and migrated/locked names where the app hides the detach option entirely (`useTransferDetachTargets`). The seller's authority persists regardless, for the lifetime of the resolver.

## Remediation

* On transfer, revoke the seller's ROOT grant (`authorizeNameRoles(0x00, ROLES_ALL, seller, false)`) and grant the buyer name-scoped roles in the same batch that moves the token. Granting the buyer alone is insufficient — the seller can revoke it.
* Never grant the wildcard `0x00`/`ROLES_ALL` combination outside the resolver-owner bootstrap; scope grants to the name's own node.
* Require detach-or-migrate-roles before a kept-resolver transfer.

## Not a duplicate (exact Known-Issues text)

The competition's published Known Issues document (2026-08-14), Part 2 preamble, states verbatim:

> "The items in this part are functional or UX defects, not security findings… However, **a demonstrated security consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding.**"

QA-01 itself is listed as: "Name transfers with certain role combinations break follow-up actions… Expect errors in these flows after transfers." — i.e. errors the *seller* hits. This finding is the opposite direction and the exact carve-out: a demonstrated role combination ("wallet holds ROOT ROLES\_ALL on the resolver") that grants the **former** owner authority it should not after transfer. WEB-1257 ("the previous owner can no longer set a subname… as their primary name" — inability) and R2-03 (session-key scope) are different mechanisms. No entry in the doc describes resolver-role retention after transfer.

## Link to Proof of Concept

<https://gist.github.com/sl4x0/f8d38bf7e754b5b08eafe10ceb2e5b6d>

## Proof of Concept

## Steps to Reproduce / PoC

> Full reproducible Forge tests (base + escalation + detach-comparison, with outputs): <https://gist.github.com/sl4x0/f8d38bf7e754b5b08eafe10ceb2e5b6d> (secret gist — shared only with triagers via this report, not public).

{% stepper %}
{% step %}

## Deploy the resolver

Resolver deployed exactly as the app does:

`deployProxy(PERMITTED_RESOLVER_IMPL, salt, initialize(hca, ROLES_ALL, []))`.
{% endstep %}

{% step %}

## Grant the seller ROOT roles

Prank HCA:

`authorizeNameRoles(hex"00", ROLES_ALL, seller, true)` -> SUCCESS;

`hasRootRoles(ROLES_ALL, seller)` -> true.
{% endstep %}

{% step %}

## Transfer the name

Transfer mirrors `transferToken.ts`: ERC-1155 `safeTransferFrom` only; resolver role store untouched.
{% endstep %}

{% step %}

## Attempt a buyer record update

Buyer `setAddr(node,60,buyer)` -> REVERTS (no roles).
{% endstep %}

{% step %}

## Rewrite records as the former owner

Seller `setAddr(node,60,attacker)` -> SUCCESS; `addr(node,60)==attacker`; `setText(node,"url","https://phish.example")` SUCCESS; seller `hasRootRoles(1<<124)` (ROLE\_UPGRADE) -> true.
{% endstep %}
{% endstepper %}

Escalation tests:

* Seller writes third-party subname `bob.alice.eth` AND unrelated node `other.eth` (wildcard).
* Seller grant/revoke despotism over the buyer; buyer cannot revoke seller or self-grant.
* Buyer cannot re-initialize or upgrade; seller ROLE\_UPGRADE intact.
* Seller rewrites exactly the records the portal worker renders (avatar/description) + resolution -> attacker.

Observed (base test console): `authorizeNameRoles ok: true`, `seller hasRootRoles(ROLES_ALL) = true`, `buyer setAddr -> REVERTED`, `sold name addr now = <attacker>`, `seller holds ROLE_UPGRADE`.

## Why detachResolver=false is not an accepted trust model (intent validation, from the competition implementation)

The one argument that could kill this finding is "the user chose to keep the resolver, so keeping the seller's authority is intended". The competition code itself disproves that:

{% stepper %}
{% step %}

## The protocol's invariant is "authority follows the token"

`PermissionedRegistry._update` (contracts-v2 PR #388 namespace, the deployed competition registry `0xBDC85dD5...`) runs on every ERC-1155 transfer:

```solidity
if (!hasRoles(tokenId, RegistryRolesLib.ROLE_CAN_TRANSFER_ADMIN, from)) revert TransferDisallowed(tokenId, from);
else if (amounts[i] > 0) {
    _transferRoles(getResource(tokenId), from, to, false);   // revokes ALL of the seller's registry roles, grants them to the buyer
}
```

The override's own comment states the intent: *"Override `ERC1155Singleton._update()` to transfer the roles to the new owner if the token is transferred."* So when a name changes hands, ENS v2 **moves the name's authority with the token** — at the registry level. The resolver role store is a second, parallel authority for the same name that the transfer lifecycle never touches: no revoke of the seller's ROOT grant, no grant to the buyer, no hook anywhere in the competition source (`rg "authorizeNameRoles|revokeRootRoles|transferRoles"` across apps/packages finds only the grant sites, the manual Roles UI, and this transfer flow that never calls them).
{% endstep %}

{% step %}

## The app itself documents the defect and expects the opposite

`apps/portal/src/features/records/hooks/useCanEditRecords.ts`:

> "Permissioned resolvers: authorized by resolver `ROLE_SET_*` (root or name-scoped). **After a transfer that keeps the resolver, the previous owner typically retains these roles while the new token owner does not.**"

The app gates the records editor on resolver roles (`canEdit = hasResolverRole`), so after a kept-resolver transfer the **buyer is locked out of the profile editor for their own name** — a user-visible break of the ownership model, not a role-store quirk. The app ships manual resolver-role management (`grantResolverRoles`/`revokeResolverRoles`, Resolver Roles sidebar) but never invokes it in the transfer flow.
{% endstep %}

{% step %}

## The UI promises the buyer control

`SendNameForm` detach-resolver description: "The recipient starts clean and sets up their own"; its warning: "Until the recipient updates them…" — the transfer UX explicitly assumes the recipient can update records. The buyer cannot (no roles), the seller can (ROOT retained).
{% endstep %}

{% step %}

## Detach only unplugs the pointer

`detachNameResolver.ts`: "It never touches the resolver contract itself… it only clears the registry's resolver pointer." The seller's ROOT roles persist on the resolver in every case — detach=true just makes them dormant until the name (or any node) resolves through that resolver again; detach=false leaves them live.
{% endstep %}

{% step %}

## Intent-validation fork PoC

`test/DetachComparison.t.sol`:

* `detach=false`: full app flow (commit → reveal → register `pocalice7q9x` via the competition ETHRegistrar with the per-HCA resolver → `safeTransferFrom` to buyer) → **buyer owns token, seller `hasRootRoles(ROLES_ALL)` = true, buyer revoke-seller attempt fails, buyer `setAddr` reverts, seller rewrites the sold name's `addr` to an attacker address (resolution hijack)**; buyer's only remedy = the registry role that moved with the token (`ROLE_SET_RESOLVER` = true — can swap the resolver pointer, losing the profile).
* `detach=true`: same flow with `registry.setResolver(tokenId, 0)` before transfer → name stops resolving through the old resolver, **seller still holds ROOT roles on it and can still write records** (dormant until any re-attach).

All writes on an ephemeral fork; nothing real modified.
{% endstep %}
{% endstepper %}
