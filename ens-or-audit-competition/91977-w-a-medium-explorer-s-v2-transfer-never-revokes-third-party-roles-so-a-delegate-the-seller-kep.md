# 91977 w a medium explorer s v2 transfer never revokes third party roles so a delegate the seller kept can re point the buyer s resolver

**Submitted on Sep 5th 2026 at 22:55:40 UTC by @SaiyanVegeta for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91977
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Bug Description

**Brief.** Explorer's transfer flow moves the ERC-1155 token and, optionally, clears the name's own resolver and subregistry pointers. It never enumerates or revokes **role assignments held by other accounts** on the name's registry resource. ENS v2 moves only the *current owner's* role bitmap on transfer, so a role the seller granted to a second address they control survives the sale intact. That address can then point the buyer's name at an attacker-controlled resolver. Explorer's "Detach the resolver" checkbox — **on by default** — does not help: the retained delegate simply re-attaches one afterwards.

**Details (verified at HEAD).**

*The transfer plan has no role step:*

* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:22-44` — the plan is exactly `set-eth-addr?` → `detach-resolver?` → `detach-registry?` → `transfer-token`. Nothing else.
* `apps/portal/src/features/transfer/hooks/useTransferName.ts:134-145` maps `transfer-token` straight to `transferToken`.
* `apps/portal/src/features/transfer/helpers/transferToken.ts:67-71` emits one call: `safeTransferFrom(from, recipient, tokenId, 1n, '0x')`.

*The flow reads roles only for the sender's own capability:*

* `useCanTransferName.ts:25,37,46` (`ROLE_CAN_TRANSFER_ADMIN`) and `useTransferDetachTargets.ts:90,95` (`heldRoles.includes('ROLE_SET_RESOLVER' | 'ROLE_SET_SUBREGISTRY')`).
* `grep -rn -iE "revoke|grantRoles|roleBitmap|ROLE_" apps/portal/src/features/transfer/ | grep -v '\.test\.'` returns only those capability checks — no enumeration of other accounts, no revocation. Read the negative as bounded by that pattern.

*The recipient is given no way to see the delegate:*

* `grep -rn -iE "roleHolders|otherRoles|useNameRoleAccounts|roleAccounts|delegate" apps/portal/src/features/transfer/` returns **no output**. Nothing in the transfer screens surfaces who else holds roles.
* The roles UI that exists elsewhere is not a reliable fallback: `apps/portal/src/features/roles/hooks/useNameRoleAccounts.ts:50,72-89` sets `MAX_EVENTS = 1000` and queries `first: MAX_EVENTS`, replaying one unpaginated page of the **newest 1000** `EACRolesChanged` events and filtering by resource only afterwards — so a grant made before that window is silently absent.

*The form presents detachment as the protection:*

* `apps/portal/src/features/transfer/components/SendNameForm.tsx:69-73` — `{ setEthAddress: true, detachResolver: true, detachRegistry: true }`. Both detach options default to on; neither changes any role assignment.

*Both halves of the attack are ordinary Explorer actions:*

* `ROLE_SET_RESOLVER` is a first-class role in the roles UI — *"Set Resolver — Can change the resolver addresses"* (`apps/portal/src/lib/roles/permissions.ts:30-34`), granted through `RegistryAddUserSheet.tsx:17-18,70-71` → `useGrantRegistryRolesMutation`, and it is **not** in `nonSettableManagerRoles` (`permissions.ts:42`).

*Why it bites — measured, not read.* The ENS v2 contracts are out of scope and are not in this repository, so I make no claim about their source. What the PoC **measures** on the real deployed registry is the behaviour that matters: the delegate's role bitmap on the name's resource is byte-identical before and after `safeTransferFrom` (`16777216` → `16777216`), and a `setResolver` from the delegate then succeeds against a token the buyer owns. Registry role bitmaps are token-versioned, but a transfer does not bump that version — observed directly, since the same `getTokenId` value serves before and after the transfer while the bitmap is unchanged.

***

## Impact

A buyer completes a handover inside the official Explorer, with the flow's own default protections enabled, and receives a name a third party still controls the resolver of.

1. **Authenticated action on the buyer's name, by someone else, with no buyer interaction.** The delegate calls `setResolver` on a token the buyer owns and it succeeds (PoC, `tx status: success`).
2. **Funds misdirection.** With an attacker-controlled resolver attached, Explorer and Manager both resolve the buyer's name to an address the attacker chose, and anyone paying that name pays the attacker.
3. **No signal to the victim.** The transfer screens never show the delegate, and the roles view elsewhere can miss it entirely (truncated 1000-event window).

No wallet compromise, no phishing site, no cross-origin content: the buyer is on the genuine Explorer throughout, and the seller's grant was made with Explorer's own "Add user" sheet.

***

## Risk Breakdown

**Not QA-01 / WEB-1257 — stated first, because that is the likeliest objection.** QA-01 (= WEB-1257) says role combinations on transfer *"cause various **errors** when later updating profile records or setting a primary name"*, and its primary-name half harms the **previous** owner. That is the erroring symptom; this is retained third-party **write authority** over the **recipient's** name. The section containing QA-01 carves this out by name: *"a demonstrated security consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding."* This report is that example, demonstrated on-chain.

* **The matched category fits, and a second one backs it.** The Critical entry reads *"during a Manager flow (registration, v1 to v2 migration, or **transfer**)"*. The Manager app implements no transfer — `grep -rln -iE "safeTransferFrom|transferToken|buildTransferPlan" apps/manager/src` returns one hit, a Markdown case study, against five source files in `apps/portal/src` — so read as "the Manager app only", the word *transfer* would be vacuous. If the project scopes it narrowly anyway, the report stands on *"resolving or rendering an ENS name to an attacker-controlled address"*: once the resolver is re-pointed, both apps resolve the buyer's name to an attacker-chosen address.
* **Not a contracts report — the one material uncertainty.** Roles persisting on a resource is ENS v2 design and the contracts are out of scope. Nothing here asks for a contract change; every remediation is app-side. The defect is that the transfer flow ships a control ("Detach the resolver", default on) implying the recipient is protected, does not deliver it, and gives the recipient no way to see the delegate. Better raised here than found in triage.
* **Attacker-controlled trigger; exclusions checked.** The seller grants the role through Explorer's own roles UI and hands the name over through its transfer flow — both normal in-app workflows, so the "un-prompted actions outside normal app workflows" exclusion does not apply. No privileged/governance address, leaked credential, physical or local-network access, third-party system or social engineering. Nothing in R2-01…R2-06, R3-01…R3-08 or the SEC-/EXP- entries touches role persistence across a transfer.
* **Severity Critical (the program's tier).** Honest bounds: a handover/sale scenario, not a drive-by, and the gain is realised only when someone later pays the hijacked name. I do not claim the "state-modifying actions on behalf of other users" entry — the delegate acts under its own retained authority rather than impersonating the buyer.

***

## Recommendation

1. Before the transfer, enumerate every account holding roles on the name's resource **from chain** — not from the truncated indexer window — and show them to the sender and to the recipient.
2. Add a revocation step to `buildTransferPlan` for every non-owner role holder, or block the transfer until they are cleared.
3. Verify the cleared state from authoritative contract state before submitting `safeTransferFrom`, the way the flow already pre-checks transfer and detach roles.
4. Until that ships, stop presenting "Detach the resolver" as the handover protection: it does not survive a retained `ROLE_SET_RESOLVER`.

***

## References

* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:22-44` (the plan; no role step)
* `apps/portal/src/features/transfer/hooks/useTransferName.ts:112-115,134-145` (`getEnsTokenId`, step dispatch)
* `apps/portal/src/features/transfer/helpers/transferToken.ts:67-71` (`safeTransferFrom` only)
* `apps/portal/src/features/transfer/components/SendNameForm.tsx:69-73` (both detach options default on)
* `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:25,37,46` and `useTransferDetachTargets.ts:90,95` (sender-only role reads)
* `apps/portal/src/lib/roles/permissions.ts:30-34,42` (`ROLE_SET_RESOLVER` is user-grantable in the UI)
* `apps/portal/src/features/registry/components/v2/RegistryAddUserSheet.tsx:17-18,70-71` (the grant path)
* `@ensdomains/ensjs` `dist/utils/v2/roles/registryRoles.js:20` (`ROLE_SET_RESOLVER = 1n << 24n` = 16777216)

***

## Proof of Concept

## Proof of Concept

All testing is local (competition rule): a local Anvil fork of the **real deployed Sepolia v2 contracts**. Nothing was sent to Sepolia; no production system, third-party account or wallet was touched.

### PoC 1 — the delegate keeps writing after the sale (executed on a Sepolia fork)

**What it does.** It replays the app's own calls, not hand-rolled equivalents: `getTokenId` (as `useTransferName` → `getEnsTokenId`), `grantRoles` (as `grantRegistryRoles.ts` → ensjs `grantRolesWriteParameters`), `safeTransferFrom` (as `transferToken.ts`), `setResolver` (as `changeResolver.ts`). A seller registers a name, grants `ROLE_SET_RESOLVER` to a delegate, transfers the token to a buyer, and the delegate then re-points the resolver.

**What to read in the result.** Three lines decide it: the owner becomes the buyer (the transfer really happened), the delegate's bitmap is byte-identical before and after (the authority really survived), and the delegate's `setResolver` returns `success` against a name it does not own.

```
=== PoC 1 — transfer retains a third party ROLE_SET_RESOLVER ===
seller   0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
buyer    0x70997970C51812dc3A010C7d01b50e0d17dc79C8
delegate 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
register receipt status: success
seller balanceOf(tokenId): 1n

registered poc1647279.eth  tokenId=17288351613010310653342597517535826388930153317777572181333923563012769185792
seller resolver: 0x350c3a1cD28803C74BB5D996846CBe4999684F83
registry resource: 17288351613010310653342597517535826388930153317777572181333923563012769185792n
  live tokenId (v)     : 0x00000001

--- BEFORE TRANSFER ---
  ownerOf(token)       : 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (seller)
  delegate role bitmap : 16777216n = ROLE_SET_RESOLVER (1<<24)

transfer tx status: success

--- AFTER TRANSFER ---
  ownerOf(token)       : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 (BUYER - transfer succeeded)
  delegate role bitmap : 16777216n *** UNCHANGED - authority survived the transfer ***

attacker resolver: 0x408f6e02F0b6Ea309FfEFD2DD68B2c61146f1622

--- DELEGATE CALLS setResolver ON THE BUYER NAME ---
  resolver before: 0x350c3a1cD28803C74BB5D996846CBe4999684F83
  tx status      : success
  resolver after : 0x408f6e02F0b6Ea309FfEFD2DD68B2c61146f1622

RESULT: CONFIRMED - buyer owns the token, the seller delegate still controls its resolver.
```

`16777216` is `1 << 24` = `ROLE_SET_RESOLVER` (`@ensdomains/ensjs` `dist/utils/v2/roles/registryRoles.js:20`).

**One implementation note, so the run reproduces.** The registry's read functions canonicalise the id (`setResolver(uint256 anyId, …)`), but ERC-1155 `safeTransferFrom` requires the exact **token-versioned** id, and a `grantRoles` in between bumps that version. The script therefore re-reads `getTokenId` immediately before the transfer — which is what `useTransferName` does at prepare time. A PoC that caches the id fails with `ERC1155InsufficientBalance` and looks like a false alarm.

The script (`poc1-transfer-retains-role.mjs`, self-contained Node, run against `anvil --fork-url <any Sepolia RPC> --chain-id 11155111 --port 8545`) is available on request; the submission form does not accept `.mjs` attachments.
