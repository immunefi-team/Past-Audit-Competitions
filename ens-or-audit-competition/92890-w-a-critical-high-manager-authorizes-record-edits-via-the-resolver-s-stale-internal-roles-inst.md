# 92890 w a critical high manager authorizes record edits via the resolver s stale internal roles instead of registry ownership a name s previous owner keeps full write access incl the coins&#x20;

**Submitted on Sep 13th 2026 at 04:09:47 UTC by @rector for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92890
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

The Manager app decides who may edit a name's records by asking the name's **resolver** ("does your internal role store still authorize this wallet?") instead of asking the **registry** ("who owns this name today?"). Registration grants the registering wallet wildcard roles on a per-owner resolver; transferring the name (any marketplace sale or plain NFT transfer) moves registry ownership but revokes nothing on the resolver. The result: **the seller of a name keeps full write access to it through the Manager's own edit UI** — including `coins[60]`, the address wallets resolve when paying the name — so the past owner can silently redirect every payment sent to the buyer's name and control its profile content until a manual resolver re-point. Demonstrated end-to-end on an anvil fork of ENS Sepolia with the Manager's own calldata (19/19 assertions), with the resolver-side authority persistence confirmed against live production state (a stale grantee's write simulation succeeds today while the current owner's identical call reverts).

## Vulnerability Details

## Root cause

The Manager treats **the resolver's internal role store** as the authority for "may this wallet edit this name's records" — never the registry's ownership record. Four facts compose:

1. **Registration grants the wallet wildcard authority on a per-owner resolver.** The default registration flow ends with `authorizeNameRoles(0x00, ROLES_ALL, wallet, true)` ([`packages/smart-account/src/providers/rhinestone/registration-calls.ts#L321-L329`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/smart-account/src/providers/rhinestone/registration-calls.ts#L321-L329)); the deployed `PermissionedResolver`'s wildcard semantics make a root-resource grant authorize **every node the resolver serves** (`_effectiveRoles = _roles[ROOT] | _roles[resource]`, verified against the Sourcify exact-match source of the live implementation `0x9EAe5C27…`). The same per-owner resolver is salt-reused across all the owner's names ([`ensureOwnedPermRes.ts#L50-L95`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/migration/service/ensureOwnedPermRes.ts#L50-L95)).
2. **Ownership transfer does not revoke that authority.** Out-of-app transfers (marketplace sales, wallet NFT transfers, raw registry ops — all reduce to registry `setOwner`) leave both the resolver pointer and the role store untouched. The Portal's transfer flow detaches the resolver by default, but the detach is user-toggleable ([`SendNameForm.tsx#L68-L73`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/components/SendNameForm.tsx#L68-L73)) and no other transfer path performs it. The Manager itself has no transfer flow at all.
3. **The Manager's write-permission probe consults only that stale store.** `resolverWriteAccess` dry-runs the real record write from `ownerAddress` against the resolver and reads revert/no-revert ([`resolverWriteAccess.ts#L35-L54`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/resolverWriteAccess.ts#L35-L54)). **No leg of the probe → machine → save path reads registry ownership** (the `profileOwner.getOwner` result is display-only). For a transferred name the probe therefore returns **true for the past owner and false for the actual current owner** — both directions demonstrated with live `eth_call`s below.
4. **The edit UI gates only on the connected wallet echoing the stored owner address** ([`useEditProfileDialogSave.ts#L218-L229`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/components/dialogs/edit-profile/useEditProfileDialogSave.ts#L218-L229); [`EditProfileDialog.machine.ts#L127-L195`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialog.machine.ts#L127-L195); the primary-name path is unconditional: [`ChoosePrimaryNameDialog.tsx#L311-L324`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx#L311-L324)) and sends the writes straight to the resolver ([`profileRecordTransactions.ts#L441-L455`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/profileRecordTransactions.ts#L441-L455)) — which, per (2)+(3), still obeys the past owner.

This is squarely the shape Part 2 of the known-issues disclosure carves out as **new**: QA-01 covers transfers where role combinations *break* follow-up actions (errors); this report demonstrates a role combination that *grants authority it should not* — persistent write execution by the wrong party.

## Vulnerable code (branch permalinks)

**1. The authority probe — resolver-internal roles only, no registry leg** — [`apps/manager/src/features/profile/service/resolverWriteAccess.ts#L35-L63`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/resolverWriteAccess.ts#L35-L63):

```ts
// dry-run the actual setRecords call from the prospective editor
const result = await publicClient.call({
  account: ownerAddress,          // <-- whoever the app believes edits the name
  to: resolverAddress,
  data: setRecordsCalldata,       // multicall([setAddr(node, 60, owner)])
})
// revert=false  =>  "resolverWriteAccess = true"
// No registry owner lookup exists anywhere in this file, the machine,
// or the save path that consumes the probe result.
```

**2. The write path — direct to the resolver, gated only client-side** — [`apps/manager/src/features/profile/service/profileRecordTransactions.ts#L441-L455`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/service/profileRecordTransactions.ts#L441-L455) + [`useEditProfileDialogSave.ts#L218-L229`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/profile/components/dialogs/edit-profile/useEditProfileDialogSave.ts#L218-L229):

```ts
// the sole guard before constructing the resolver write:
if (connectedWallet.address !== account.ownerAddress) return  // trivially true for the past owner
// then: { to: resolverAddress, data: setRecordsWriteParameters(...) }  -- resolver-side role gate decides
```

**3. The precondition is created by the app's own default** — [`packages/smart-account/src/providers/rhinestone/registration-calls.ts#L321-L329`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/packages/smart-account/src/providers/rhinestone/registration-calls.ts#L321-L329):

```ts
// end of the reveal batch: grant the registering wallet EVERY role at the root resource
authorizeNameRoles(hex'00', ROLES.ALL, wallet, /* setRoles */ true)
// root-resource roles are wildcard: they authorize every node the resolver serves
```

**4. Nothing on the transfer side revokes it** — the portal's detach is opt-out ([`SendNameForm.tsx#L68-L73`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/components/SendNameForm.tsx#L68-L73), [`useTransferDetachTargets.ts#L83-L96`](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts#L83-L96)); marketplace/NFT/raw-registry transfers bypass the apps entirely.

## Attack path (step-by-step)

1. **Attacker registers `victimname.eth` via the Manager** (the default flow). The app deploys/reuses their per-owner `PermissionedResolver` proxy and grants the attacker's wallet wildcard `ALL_ROLES` at the root resource — the resolver now serves every record for this name under the attacker's authority.
2. **Attacker sells the name** on any marketplace, or transfers it with a plain wallet NFT transfer. Registry ownership moves to the buyer; the resolver pointer and the attacker's role entries are untouched (this is the property every out-of-app transfer shares — they all bottom out in registry `setOwner`).
3. **Attacker opens the Manager and connects the same wallet.** On the sold name's profile, `resolverWriteAccess` dry-runs a record write from their address against the resolver — the resolver's internal store still lists them → probe returns **true**. (The actual new owner's identical probe **reverts** — live demonstration below.)
4. **Attacker opens Edit Profile** (or the primary-name dialog) and saves: `coins[60]` (the ETH payment address) → attacker address, plus arbitrary text/avatar records. The only guard — connected wallet equals the stored owner address — is satisfied because the Manager considers whoever the *resolver* authorizes as the editor; the write goes straight to the resolver and executes.
5. **Every resolver consumer now pays the attacker.** Wallets and payment integrations that resolve `victimname.eth` read the attacker-controlled `coins[60]`. Funds sent to the name inside the window are gone; the victim's profile text/avatar display attacker content. The state persists until the victim notices and executes a registry-owner write to re-point the resolver (`setResolver(node, 0/new)`) — a write that requires holding `ROLE_SET_RESOLVER` and knowing to do it.

## Impact Details

## Severity

**High** — with the demonstrated consequence mapping directly onto the program's own Critical impact row example:

> "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: **resolving or rendering an ENS name to an attacker-controlled address** …"

and the transfer-flow row ("Hijacking or bricking an ENS name during a Manager flow … assigning the name, its ownership, registry, or **resolver** to an attacker") applying to the write path's authority choice.

* *Likelihood:* the attacker **controls the arming event** — they register a name through the Manager's default flow (which grants their wallet wildcard roles on the per-owner resolver), then sell or transfer the name on **any** marketplace or via a plain wallet NFT transfer. Every such transfer path bottoms out in a registry `setOwner` that does not touch the resolver pointer or its role store. The attacker then simply uses the Manager's own edit UI. No victim error, no phishing, no unusual state — the victim's only "action" is buying a name.
* *Impact:* the past owner rewrites **every record class** on the transferred name — critically `coins[60]`, the address wallets and payment processors read when resolving the name for transfers. Funds sent to the name inside the window land at the attacker's address and are unrecoverable. This is resolution hijack at the protocol level, not a display artifact.
* Claimed High rather than Critical only because the arming precondition (an out-of-app transfer that keeps the resolver pointer) is buyer-dependent and today's deployment is the testnet instance; the impact row cited above is Critical-tier and demonstrated end-to-end in the PoC.

## Impact

* **Payment-resolution hijack on a transferred name.** Anyone who sends funds to the victim's name inside the window pays the attacker — not because a screen lied, but because the name's canonical record now points at the attacker. That is the program's own Critical-row example ("resolving an ENS name to an attacker-controlled address"), executed through the Manager's UI by the one party that should have lost all authority: the seller.
* **Persistent cross-user content control.** All record classes (text, avatar, coin addresses) on the victim's name are attacker-writable until a manual resolver re-point; the victim's public profile serves attacker-chosen content.
* **Blast radius = the secondary market for Manager-registered names.** Every sale/transfer that bypasses the Portal's (toggleable) detach leaves the seller armed. The resolver-side authority persistence is measurable in production state today (stale grantees holding `ALL_ROLES` on live proxies; stale-grantee write simulation succeeding). The attacker sets the price of the arming event: they can sell their own name cheap and immediately strike the buyer.
* **Both-sided failure of the authority model.** The same probe that wrongly admits the past owner also wrongly **rejects the real owner** (live revert from the current owner) — the Manager's write gate is decoupled from ownership in both directions, which is the definition of the wrong authority source, independent of any single transfer scenario.

## References

All code references are permalinks to `immunefi-team/audit-comp-ens` @ `audit-comp-ready` (`1c9b47f`): apps/manager (write path, probe, edit UI), apps/portal (toggleable transfer detach), packages/smart-account (registration grant). Resolver-contract line references point at the Sourcify exact-match source of the deployed PermissionedResolver implementation.

## Proof of Concept

## Proof of concept

**Two legs: a local fork run of the full attack (19/19 assertions), and read-only live probes proving the authority persistence against production state.**

**(a) Fork PoC — full attack, byte-exact Manager calldata.** A local anvil fork of ENS Sepolia (block 11,691,720). The resolver stack is **real**: the Manager-pinned `VerifiableFactory`, a real factory-deployed per-owner proxy, and the live `PermissionedResolver` implementation (fork bytecode keccak asserted equal to the deployed `0x9EAe5C27…`). The write and probe calldata are generated by the **real `@ensdomains/ensjs`** `setRecordsWriteParameters` — the same two calls the Manager makes (`resolverWriteAccess.ts:35-54`, `profileRecordTransactions.ts:441-452`).

Sequence (tx hashes in run log): deploy `R_A` with `initialize(A, ALL_ROLES, [])` → create `esc1poc.eth` owned by A → A `setResolver(node, R_A)` → baseline write ok → **registry `setOwner(node, B)`** (the primitive every transfer reduces to) → assert `owner=B`, `resolver=R_A` unchanged, `hasRootRoles(ALL_ROLES, A)` unchanged → **Manager probe simulation: `eth_call` from A succeeds (= probe true), from B reverts** → A's post-transfer writes via the Manager's own calldata **mine successfully** (status 0x1): `multicall([setAddr(node,60,A)])` and `multicall([setAddr, setText(description, "attacker-controlled profile text")])` → readback: `addr(esc1poc.eth) = A`, `text(description) = attacker string` → control: identical write from B reverts (`eth_call` revert + mined status 0x0, `EACUnauthorizedAccountRoles` fully decoded).

Key assertions:

| #     | Assertion (from the 19)                              | Proves                                                                               |
| ----- | ---------------------------------------------------- | ------------------------------------------------------------------------------------ |
| 1     | fork impl bytecode == live bytecode                  | executes the exact deployed resolver code                                            |
| 8     | resolver pointer STILL `R_A` after transfer          | the enabling precondition                                                            |
| 9     | `hasRootRoles(ALL_ROLES, A)` unchanged post-transfer | transfer revokes nothing                                                             |
| 10–11 | probe `eth_call` from A → success; from B → revert   | the Manager's own probe says **yes to the past owner, no to the real owner**         |
| 12–15 | A's writes mine; readback `addr=A` + attacker text   | wrong-authority **execution** + protocol-level resolution hijack                     |
| 16–17 | B's identical write reverts both ways                | the gate is resolver-internal roles, decoupled from ownership in **both** directions |

**(b) Live read-only probes (production state, today).** All 42 factory-deployed `PermissionedResolver` proxies on ENS Sepolia enumerated (blocks 11,383,823 → 11,691,538); root-grant events cross-referenced with served nodes and current registry owners:

* live resolver `0x6abf…3c76`: `roles(0, 0x03ba…17ef) = ALL_ROLES` — a grantee that no longer owns the enumerated names still holds full authority **today**;
* `eth_call setAddr(brave-osprey.eth, 60, self)` **from that stale grantee against live state → success** (`0x`);
* the identical call **from the current registry owner `0xF83f…4750` → revert `EACUnauthorizedAccountRoles`**.

I.e. the resolver-side half of the attack is armed in production state right now; the only variable per name is whether the current owner re-pointed the resolver — and nothing in the Manager warns either party. (No full transferred+pointer-intact pair existed within the probe budget at audit time; the enumerated population shows owners who happened to re-point or detach — the Portal's default mitigation doing its job for the one flow that has it.)

**Stub disclosure (the only one):** the demo's name substrate is a locally deployed minimal v1 `ENSRegistry` with the public MIT contract's verbatim semantics — Sepolia's V2 rotation removed every registrar path that mints fresh EOA-owned names (v1 controller de-authorized; V2 registrar mints ERC-1155 to contract owners only), so no fresh production-registrable name was available to transfer. The factory, proxy, resolver implementation, role store, calldata, and the transfer primitive (`registry.setOwner` — what every transfer bottoms out in) are all real; no stubbed surface touches the mechanism under test. Zero public-testnet state was modified (fork + read-only calls only).

Reproduce: anvil fork at the pinned block + the run script and calldata generators (PoC package attached to this report; run log with all 19 assertions and both probe transcripts included).

## Regression test

The fork PoC's assertions 8–17 are the contract, drop-in: after the fix, (i) the probe/preflight must return **false** for a wallet that is not the current registry owner of the node (assertion 10 inverts), (ii) the Manager-constructed write from the past owner must not be offered/execute (12–15 fail closed), and (iii) the legitimate current owner's write must keep working (16–17 invert to success). The live probe pair (stale grantee succeeds / current owner reverts) is the production-state canary for the same property.
