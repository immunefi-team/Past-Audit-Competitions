# 91635 w a critical incorrect resolution on the manager profile page displays an attacker controlled receiving address under a legitimate ens name s owner identity

**Submitted on Sep 3rd 2026 at 08:10:15 UTC by @SaiyanVegeta for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91635
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Bug Description

**Brief.** The Manager profile page at `/<name>` builds one identity from two lookups on the *same* URL string, but one normalizes the name and the other does not. The **owner, primary name, expiry and registration date** are looked up on the **normalized** name — so for `/Vitalik.eth` (or an invisible-character variant of `vitalik.eth`) they show the real `vitalik.eth` and its real owner. The **records — including the receiving ETH address shown with a copy button — are looked up on the RAW URL string**, which is a *different* ENS name the attacker registered and whose resolver they control. The page therefore presents the real owner's identity next to an address the attacker chose. This is not display confusion (EXP-INPUT-005); it is an incorrect resolution the app presents as authoritative, proven end-to-end on real contracts in PoC 3.

**Details — the raw-vs-normalized node divergence (verified at HEAD, and live in PoC 3).**

* `apps/manager/src/routes/$name/route.tsx` — the route param is accepted with only a length/dot check; there is **no normalization and no canonical redirect**.
* `apps/manager/src/routes/$name/index.tsx` (\~lines 52-55) — the loader runs `profileRecordsQuery(name)` and `profileOwnerQuery(name)` in parallel on the **same raw param**; the loader's only gate is that an owner was found, which the normalized lookup satisfies.
* `apps/manager/src/features/profile/service/profileOwner.ts:24` — `normalizeEthName(params.name)` before `getOwner`; likewise `profileExpiry.ts` and `profileRegistration.ts` normalize first. Owner, primary name, expiry and registration date therefore come from the **normalized** node (the real `vitalik.eth`).
* `apps/manager/src/features/profile/service/profileRecords.ts:95` (indexer id = raw) and `~:123` — `getRecords(client, { name })` is called with the **raw** URL name. Records — including `addr(60)` — are therefore resolved against the **raw name's ENS node and its resolver**, which the attacker owns. Proven live (PoC 3): for the raw twin, `getRecords` returns `resolverAddress = the attacker's resolver` and `addr(60) = the attacker's address`, while `getOwner(normalize(name))` returns the victim. The attacker simply sets that address on the twin they own — an ordinary name-owner action.
* `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx` (\~:76-132, `MainAddressCard`) renders that resolved ETH address inside a `CopyableButton` labelled "Main receiving address"; `ProfileDetails` renders the real owner and the real primary name next to it.

**Attack, end to end (every step reproduced in PoC 3).**

1. The attacker registers a raw twin of a target name — e.g. `Vitalik.eth`, or an invisible-character variant of `vitalik.eth` that renders identically. The v2 registrar accepts it (no ENSIP-15 enforcement on-chain).
2. On the twin's own resolver, the attacker sets `addr(60)` to their own address, and copies the target's avatar and description.
3. The attacker sends the victim a link to the twin on the official Manager domain, e.g. `https://app.ens.domains/Vitalik.eth`.
4. The page shows the real `vitalik.eth` owner, primary name and expiry (normalized lookups) next to the attacker's address as the "Main receiving address", with a copy button (raw lookup).
5. The victim copies the address and sends funds to the attacker.

**Fix locus differs from EXP-INPUT-005.** The one-line root-cause fix is in `routes/$name` (normalize the param, redirect to canonical, and pass the normalized name to `profileRecordsQuery`). Hardening the register/search *validators* — the EXP-INPUT-005 remediation — does not touch `routes/$name/route.tsx` or `profileRecords.ts`, and the bug persists.

## Impact

A user who opens a crafted link to the official Manager domain sees the real owner, real primary name and real expiry of a valuable name next to a receiving address that the attacker controls, and copies it to send funds. This is the enumerated impact "resolving or rendering an ENS name to an attacker-controlled address … so that a user sends funds to an unintended recipient." The attacker also controls the avatar and description shown (keyed on the raw node), so the page is visually indistinguishable from the legitimate profile.

## Risk Breakdown

* **Not EXP-INPUT-005.** That known issue scopes itself, verbatim, to inputs that cause *"display confusion in the UI rather than incorrect resolution."* This report demonstrates the opposite: an actual incorrect resolution. No name *validator* is on this path; the divergence is between two sibling resolution services (`profileOwner` normalizes, `profileRecords` does not) that both run after any validator. The EXP-INPUT-005 fix does not modify the affected files.
* **Not WEB-1301.** WEB-1301 is the *address* view showing owner/registered/expires from the wallet's primary name. This is the *name* view, and the incorrect field is the receiving address, not the metadata.
* **Not the "records are correct for the attacker's own name" reframe.** The app itself has decided the page *is* `vitalik.eth`: owner, primary name, expiry and registration are all read from `namehash('vitalik.eth')`, and the inner call is literally `addr(namehash('vitalik.eth'))`. The app asks the chain for vitalik.eth's ETH address and renders an attacker contract's answer. Under ENSIP-15 there is no second name — the app either treats the input as vitalik.eth or rejects it, and it does the former for every identity field.
* **Threat model:** no MITM, no RPC compromise, no connected wallet. The attacker uses only on-chain state they permissionlessly own, delivered by a link to the official domain.
* **Precondition and full mechanism — reproduced end-to-end on real contracts (no caveat left).** I ran the whole attack on a local Sepolia fork of the **real deployed contracts** (registry, Universal Resolver, PermissionedResolver, VerifiableFactory, v2 ETHRegistrar), driven by the **exact ensjs functions the app calls**. Registering the non-normalized twin succeeds (the v2 registrar does no ENSIP-15 normalization), and then `getRecords(rawTwin)` returns the attacker's address while `getOwner(normalize(rawTwin))` returns the victim (see PoC 3). This is not modelled — the real Universal Resolver routes the raw name to the attacker's resolver (`getRecords(T).resolverAddress == attacker resolver`).
* **Severity Medium (honest):** no already-connected-wallet interaction (not Critical); no persistent injection and no other user's data changed (not High); the victim must open a crafted link on the official domain and copy-paste the address into another app (no on-page send). Do not claim High.

## Recommendation

Normalize the name once at the `routes/$name` boundary; if it is already canonical, proceed, otherwise redirect to the canonical spelling (or 404 on a non-normalizable input). Pass the **normalized** name to `profileRecordsQuery` so owner, records and the receiving address are all resolved against a single node.

## References

* `apps/manager/src/routes/$name/route.tsx`, `apps/manager/src/routes/$name/index.tsx` (\~52-55)
* `apps/manager/src/features/profile/service/profileOwner.ts:24`, `profileExpiry.ts`, `profileRegistration.ts`, `profileRecords.ts:95,~123`
* `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx`
* ensjs resolution internals: `@ensdomains/ensjs` `resolveNameData` DNS-encodes the raw name, so records for the raw name resolve against the raw name's node and resolver (behaviour proven empirically in PoC 3).

## Proof of Concept

## Proof of Concept

All testing is local (competition rule). Three PoCs build up the claim: PoC 1 shows the two lookups hit two different nodes; PoC 2 shows the app admits the crafted name in the first place; PoC 3 reproduces the whole attack on the real deployed contracts. Each is a runnable file under the package's `src/audit/`; no source file was modified.

### PoC 1 — the two lookups resolve two different nodes (vitest, 7/7 pass)

**What it does.** It calls the app's own two services — `getProfileRecords` and `getOwner` — for the same URL string, and records, at the wire level, which ENS name each one actually sends to the chain. The Universal Resolver is modelled on the real walk semantics so the test needs no network.

**What to read in the result.** For every non-normalized spelling of `vitalik.eth` the *records* call goes out for the raw string while the *owner* call goes out for the normalized string, and the address the profile component ends up rendering is the attacker's. For the plain `vitalik.eth` (already canonical) the two agree — the control.

```
name "Vitalik.eth"   -> records lookup uses "Vitalik.eth" (raw) | owner lookup uses "vitalik.eth" (normalized) | rendered addr = ATTACKER
name "ｖitalik.eth"   -> records lookup uses "ｖitalik.eth" (raw) | owner lookup uses "vitalik.eth" (normalized) | rendered addr = ATTACKER
name "vitalik.eth"   -> records lookup == owner lookup == "vitalik.eth"                                        | rendered addr = owner's  (control)
```

The companion `tech-trust-boundaries-ensjs-mixed-normalization.test.ts` intercepts the single Universal-Resolver `eth_call` and confirms it carries `dnsEncode(raw)` — never `dnsEncode(normalized)` — which is *why* the resolver picked is the raw name's (attacker's) resolver.

```
cd apps/manager && NODE_OPTIONS="--max-old-space-size=8192" npx vitest run src/audit/tech-displayed-vs-signed-profile-resolution.test.tsx
```

### PoC 2 — the app accepts the crafted name (vitest, 7/7 pass)

**Why this PoC exists.** A reviewer could object "the route would reject that weird name before any of this runs." This proves it does not, and that the crafted name is exactly the kind that splits the two lookups: it must (a) be accepted by the route parser, (b) normalize to the victim's canonical name (so the owner lookup lands on the victim), yet (c) be a different string from the canonical name (so the records lookup goes elsewhere).

**What to read in the result.** For six spellings of `vitalik.eth` — including invisible ones (soft hyphen, word joiner, zero-width space) that render identically to the real name — all three conditions hold:

```
spelling                | route accepts it | normalizes to | differs from canonical?
mixed case  "Vitalik.eth"        | yes | vitalik.eth | yes
soft hyphen "vit­alik.eth"       | yes | vitalik.eth | yes   (invisible in the URL)
word joiner "vita⁠lik.eth"       | yes | vitalik.eth | yes   (invisible)
zero-width  "vitalik​.eth"       | yes | vitalik.eth | yes   (invisible)
fullwidth   "ｖitalik.eth"        | yes | vitalik.eth | yes
```

```
cd apps/manager && NODE_OPTIONS="--max-old-space-size=8192" npx vitest run src/audit/tech-a1-loader-gate.test.ts
```

### PoC 3 — the whole attack on the real contracts (Sepolia fork, no fakes)

**The setup, in plain terms.** On a local Anvil fork of the **real deployed Sepolia contracts** (registry, Universal Resolver, PermissionedResolver, VerifiableFactory, v2 ETHRegistrar):

1. register a canonical name `C` (`zqa….eth`) to a **victim** with a normal resolver;
2. register its raw twin `T` (`ZQA….eth` — same look-alike relationship as `Vitalik`/`vitalik`, a *different* on-chain node) to an **attacker**;
3. as the attacker, set the ETH address record on the twin `T` to the attacker's own address — a completely ordinary thing any name owner can do;
4. then call the **exact functions the Manager profile page calls**: `getOwner` (from `@ensdomains/ensjs/public/v2`) and `getRecords` (from `@ensdomains/ensjs/public`), on an `extendChainWithEns` client — the same client type the app uses.

**What to read in the result.** The owner lookup (which the app normalizes) returns the victim; the records lookup for the raw twin (which the app does not normalize) returns the attacker's address; the canonical name resolves cleanly (control). The `resolverAddress` line shows the real Universal Resolver sent the raw name to the attacker's resolver — no test scaffolding, the live contracts did the routing.

```
canonical C = zqa420539.eth  (victim)      raw twin T = ZQA420539.eth  (attacker)

owner of normalize(T)        = VICTIM    0x7099…79C8   <- what the profile shows as the OWNER   (getOwner, normalized)
receiving address for T      = ATTACKER  0x3C44…93BC   <- what the profile shows as the ADDRESS (getRecords, raw name)
receiving address for C      = VICTIM    0x7099…79C8   <- control: the canonical name is clean
getRecords(T).resolverAddress = the attacker's resolver <- the REAL Universal Resolver routed the raw name here

ASSERTIONS
  owner(normalize(T)) == VICTIM    : true
  getRecords(T).eth   == ATTACKER  : true
  getRecords(C).eth   != ATTACKER  : true
>>> BUG REPRODUCED END-TO-END ON REAL CONTRACTS <<<
```

**What it proves.** On the in-scope deployment's real contracts, opening the profile for the twin shows the victim's owner, primary name and expiry next to the attacker's receiving address — the exact funds-misdirection impact, with nothing modelled. (A companion `onchain__fork-label-validity.mjs` separately confirms the v2 registrar accepts uppercase, fullwidth, soft-hyphen, zero-width-space and FE0F labels, so the twin from step 2 is mintable.)
