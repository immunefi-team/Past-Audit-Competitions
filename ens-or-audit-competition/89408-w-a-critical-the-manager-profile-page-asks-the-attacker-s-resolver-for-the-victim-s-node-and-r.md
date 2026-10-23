# 89408 w a critical the manager profile page asks the attacker s resolver for the victim s node and renders the answer under the victim s real ownership and expiry

## #89408 \[W\&A-Critical] The Manager profile page asks the attacker's resolver for the victim's node, and renders the answer under the victim's real ownership and expiry

**Submitted on Aug 19th 2026 at 05:07:06 UTC by @Naresh\_Kandula for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89408
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

### Description

## The Manager profile page asks the attacker's resolver for the victim's node, and renders the answer under the victim's real ownership and expiry

**Target:** Manager app (`apps/manager`) — `https://github.com/immunefi-team/audit-comp-ens` @ `audit-comp-ready`\
(fork of `ensdomains/apps-monorepo` @ `63772fd872af472ced58b009499355f3430c2a86`)

**Severity:** Critical

**Impact in scope:** *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient — resolving or rendering an ENS name to an attacker-controlled address; showing the wrong recipient or amount on a transaction the user then approves."*

**On severity.** The clause's operative words are *"display … an incorrect address, name, or transaction detail, so that a user sends funds … to an unintended recipient"*, with the named example *"resolving or rendering an ENS name to an attacker-controlled address"*. That is this page, exactly: it renders a profile under the victim's genuine on-chain ownership, registration date and expiry, whose records were answered by a resolver the attacker selected.

Two distinctions worth naming rather than leaving for you to find. Neither moves the finding outside the clause.

* The **Main receiving address** becoming an attacker address additionally requires the attacker to point their own name at a resolver contract they control — a dedicated per-name resolver is the ENSv2 default, so this is ordinary usage rather than an exotic setup. Section *Impact* separates what holds against a stock shared resolver from what holds with a custom one.
* The page has no send or approve control on it; the transfer happens afterwards, in the user's wallet. The clause does not require the send to execute in-app — it requires the application to cause the incorrect display "so that" the user sends wrong. A profile page that attests ownership and prints a copyable receiving address is precisely the artefact that decision is made from.

### Summary

The `/$name` route takes the name straight from the URL and never canonicalises it. Downstream, that one string is subjected to **three different hashing rules**, so a single page load splits into two ENS identities:

| Step                                                    | Rule applied to the route param | For `/Vitalik.eth`                          |
| ------------------------------------------------------- | ------------------------------- | ------------------------------------------- |
| Owner / Registered / Expires                            | `normalize(name)`               | `vitalik.eth` — the **victim's** name       |
| Resolver **discovery** (which contract is asked at all) | raw, DNS-encoded                | `Vitalik.eth` — the **attacker's** resolver |
| `text(...)` node in the calldata                        | `namehash(name)` — raw          | `Vitalik.eth`                               |
| `addr(...)` node in the calldata                        | `namehash(normalize(name))`     | `vitalik.eth`                               |

The UniversalResolver picks the resolver from the **name** and then forwards the caller's calldata **verbatim**. So the app ends up asking the **attacker's resolver** for the **victim's node** — and renders whatever it answers as "Main receiving address", directly beneath the victim's genuine on-chain ownership, registration date and expiry.

This needs no homoglyph, no bidirectional control character and no invisible codepoint. One capital letter is enough.

### Vulnerability details

{% stepper %}
{% step %}

### The route param is accepted verbatim

`apps/manager/src/routes/$name/route.tsx:5-13`:

```ts
parse: ({ name }) => {
  if (name.length < 3 || !name.includes('.')) throw new Error('Invalid ENS name')
  return { name }
},
```

No `normalize()`, no `beforeLoad`, no canonical redirect. `routes/$name/index.tsx` was read end to end: its only two redirects (`:82-86`, `:111-115`) are gated on the name being absent or past grace and re-pass `params: { name }` verbatim; `routes/p/$name/index.tsx:9` forwards the share route's param unchanged. There is no global `beforeLoad` on `__root`.
{% endstep %}

{% step %}

### Records are read for the raw string

`apps/manager/src/features/profile/service/profileRecords.ts:82` takes `name` unchanged and at `:122-131` passes it to ensjs `getRecords`. `data/records/utils.ts:42` sets `alwaysProbeAddressRecords = ['60']`, unioned into `coins` at `:116-120`, so `addr(60)` is requested **even when the indexer has no entry for the raw label** (`Domain.graphql` declares `query Domain($id: String!)`, so a miss simply returns `domain: null` and execution continues).
{% endstep %}

{% step %}

### Ownership, registration and expiry are read for the normalised string

`profileOwner.ts:24-25,56` (`normalizeEthName` → `profileName.ts:18` → `viem/ens` `normalize`), `profileExpiry.ts:77`, `profileRegistration.ts:17`.
{% endstep %}

{% step %}

### ensjs applies three different rules, which splits the identity

In the installed `@ensdomains/ensjs` @ `e96662c`:

* `src/actions/public/resolveNameData.ts:70,93` sends `toHex(packetToBytes(getNameWithSizedLabels(name)))` to the UniversalResolver. `src/utils/name/getNameWithSizedLabels.ts` only hashes labels longer than 255 bytes — **it does not normalise**. Resolver discovery therefore happens on `Vitalik.eth`.
* `src/utils/coders/getText.ts:49` → `args: [namehash(name), key]` — **raw** node.
* `src/utils/coders/getAddress.ts:121-137` → `namehash(normalize(name))` — **normalised** node.
  {% endstep %}

{% step %}

### The UniversalResolver does not reconcile them

In `ensdomains/ens-contracts`, `contracts/universalResolver/AbstractUniversalResolver.sol`, `resolveWithGateways(name, data, gateways)` derives `ResolverInfo` from the **name** (`requireResolver(name)`) and then passes the caller-supplied `data` straight into `_callResolver`. The only validation on that path is of the *response* (`UnsupportedResolverProfile(callSelector)`); the node embedded in `data` is never compared against `info.node`. The attacker's resolver receives `addr(namehash("vitalik.eth"), 60)` and may return anything.
{% endstep %}

{% step %}

### Both halves are rendered together

`ProfileView.tsx:193-210` passes the raw `name` and the raw-derived `records` alongside the normalised-derived `owner` / `registrationDate` / `displayExpiryDate`.

`ProfileDetails.tsx:113-141` prints Owner, Registered and Expires; `ProfileAddressesSection.tsx:159,171-175` prints `getMainReceivingAddress(records)` under the heading **"Main receiving address"** inside a `CopyableButton`.
{% endstep %}

{% step %}

### There is no warning

`apps/manager` contains no `isNormalized` check and no "not normalized" string anywhere in the profile feature; `ProfileStatusBanners.tsx` carries only grace-period and migration banners. The Explorer *does* have such a badge (`apps/portal/src/routes/$name/token.tsx:290,338-352`), so the asymmetry is not a stylistic choice.
{% endstep %}
{% endstepper %}

**Premise — non-normalised labels are registrable.** Asserted by the codebase itself: `apps/portal/src/utils/token/isNameOrAddress.ts:11-13` — *"the ENSv2 contracts impose no character constraints on labels (only a 1–255 byte length)"* — and `apps/manager/src/features/dashboard/components/NamesTable.tsx:84-86`.

The v2 registrar ABI declares `NotValid(string label)` (`@ensdomains/ensjs-abi/src/v2/ethRegistrar.ts:97-98`) and the repo's own comments describe the check as `StringUtils.strlen` code-point length (`apps/portal/src/features/register/utils/premium.ts:25`, `useBaseRate.ts:44`, `apps/manager/src/routes/$name/index.tsx:37-45`) — i.e. length-only.

### Impact

A user opening a link to `/Vitalik.eth` sees a profile whose **Owner, Registered and Expires are genuinely those of `vitalik.eth`** — the exact fields a careful user checks to confirm authenticity — above a **receiving address that the attacker's resolver returned**, with a one-click copy button. The correct ownership data makes the page *more* convincing than an ordinary lookalike, not less.

Even against a stock shared resolver, all **text records** (`avatar`, `description`, `url`, `com.twitter`, `header`, `theme`) and `contenthash` are read on the raw node and are attacker-controlled with no custom contract at all, so the entire visual identity of the page is the attacker's while the ownership proof is the victim's.

Secondary: for the genuine owner of `vitalik.eth`, `ProfileView.tsx:113` `useIsOwner({ owner })` is computed from the normalised read and therefore returns true on `/Vitalik.eth`, so the app offers Edit-profile / resolver actions bound to the raw name while attesting the name is theirs.

### Why this is not `EXP-INPUT-005`

This is the objection to pre-empt, so it is addressed on root cause rather than on exploitation path.

`EXP-INPUT-005`'s root cause is **lax input validation**: validators accept homoglyphs, bidi and control characters — strings that merely *look like* the victim's name. `Vitalik.eth` is not a lookalike. Under ENSIP-15 it **is** the victim's name: `normalize("Vitalik.eth") === "vitalik.eth"`. There is nothing for a user to spot and no validator that should have rejected it. The defect is that one canonical name is resolved to **two different nodes inside a single page render**, and the published known-issues text carves this out verbatim: *"(Incorrect RESOLUTION would be new.)"*

A second objection worth answering in advance: *"the asymmetry is inside ensjs — an unmodified third-party dependency, which is out of scope."* ensjs behaves as documented; like viem, it expects **the caller** to normalise. The defect is the app's: `routes/$name/route.tsx` does not canonicalise the param, and `profileRecords.ts` passes a different string to ensjs than `profileOwner.ts` / `profileExpiry.ts` / `profileRegistration.ts` do. The fix is entirely in `apps/manager`.

### Why this is not otherwise a known issue

`EXP-INPUT-005`'s root cause is a permissive **validator** accepting homoglyphs / bidi / control characters, and the published list scopes it to *"homograph-style display confusion in our UI rather than incorrect resolution"*, adding explicitly *"(Incorrect RESOLUTION would be new.)"*

This root cause is different: **one page resolves one string under three different rules**, so resolver discovery and the queried node come apart. It reproduces with plain ASCII (`/Vitalik.eth`) — no homoglyph, no bidi, no control character — and the consequence is that the app queries an attacker-chosen resolver for the victim's node while attesting the victim's ownership. Tightening the validators would not fix it; only canonicalising the route param, or making every reader hash the same string, would. No entry in Part 1 or Part 2 mentions the profile route, dual name resolution, or a records-vs-owner mismatch.

On the 19 bare `WEB-*` identifiers, which publish no behaviour: the six that *are* described are product defects (a commit loop, missing gas warnings, a stale wrapper expiry, a detached child registry) — the character of a UX tracker. This report does not claim a malfunction. It claims that the app **queries a resolver the attacker chose for a node the attacker does not own, and prints the answer under the victim's genuine ownership attestation**. If ENS believes a tracker entry covers it, that entry will name `routes/$name/route.tsx` or the `profileRecords` / `profileOwner` split; nothing in the published set does.

### Every component outside `apps/manager` behaves correctly

Stated early, because "this is really an ensjs / UniversalResolver issue, and those are out of scope" is the objection this finding most invites.

* **ensjs is not defective.** Like viem, it documents that the *caller* normalises. Its coders are internally consistent with their own contracts: `getText.ts:49` hashes what it was given, `getAddress.ts:121-137` hashes `normalize(name)` because a coin-60 address record is defined on the canonical name.
* **The UniversalResolver is not defective.** `AbstractUniversalResolver.sol` derives the resolver from the name it is given and forwards the caller's `data` verbatim. That is its documented contract; it is a router, not a validator of caller-supplied nodes.
* **The registrar is not defective.** ENSv2 imposes no character constraints on labels beyond a 1–255 byte length, which the repository states itself (`apps/portal/src/utils/token/isNameOrAddress.ts:11-13`).

The defect is entirely `apps/manager`'s: `routes/$name/route.tsx:5-13` does not canonicalise the param, and `profileRecords.ts` then passes a **different string** to ensjs than `profileOwner.ts`, `profileExpiry.ts` and `profileRegistration.ts` do. The fix is one redirect at the route boundary, in an in-scope file. No dependency and no contract needs to change.

### Link to Proof of Concept

<https://gist.github.com/NOTTIBOY137/9841ee68ac2309770ce4371132b93f66>

### Proof of Concept

**PoC:** <https://gist.github.com/NOTTIBOY137/9841ee68ac2309770ce4371132b93f66> (secret gist — README, PoC file, run transcript)

**Self-contained. No network, no wallet.** Added to the repo at:

```
apps/manager/src/features/profile/service/poc-dual-resolution.test.ts
```

{% tabs %}
{% tab title="pnpm" %}

```bash
pnpm install
pnpm --filter manager exec vitest run \
  src/features/profile/service/poc-dual-resolution.test.ts --reporter=verbose
```

{% endtab %}
{% endtabs %}

#### Actual output

```
 ✓ TIER 1 … > sends the NORMALIZED name to the ownership read and the RAW param to the records read 200ms
 ✓ TIER 1 … > always requests addr(60) for the raw param, even with no indexer entry 1ms
 ✓ TIER 2 … > addresses the UniversalResolver and DNS-encodes the RAW label for resolver discovery 21ms
 ✓ TIER 2 … > carries the VICTIM node for addr() and the ATTACKER node for text() in the same calldata 7ms
 ✓ TIER 2 … > ORACLE: the already-canonical name produces exactly ONE node 2ms
 ✓ TIER 2 … > needs nothing but a capital letter 0ms

 Test Files  1 passed (1)
      Tests  6 passed (6)
```

#### Tier 1 — the app splits the param between two readers

Drives the **real** service functions; only the network edge is mocked. For the single input `"Vitalik.eth"`: `getOwner` issues its read for `"vitalik.eth"` while `getProfileRecords` issues its read for `"Vitalik.eth"`, and `coins` always contains `60` so `addr(60)` is requested even with no indexer entry.

#### Tier 2 — the literal RPC payload, built by the real unmocked ensjs

This tier recomputes **no hashes of its own**. It runs the real `@ensdomains/ensjs` `getRecords` against a client that intercepts the read (`resolveNameData.ts:71` resolves its reader via `getAction(client, readContract, 'readContract')`, so a client carrying a `readContract` property captures the call before any I/O) and then asserts on the bytes ensjs actually produced:

* the call goes to the UniversalResolver with `functionName: 'resolve'`;
* **`args[0]`** — the name used to choose *which resolver contract is called* — is byte-for-byte `packetToBytes("Vitalik.eth")`. It contains `Vitalik` and does **not** contain `vitalik`. Resolver discovery lands on the attacker's name;
* **`args[1]`** — the multicall blob of inner resolver calls — contains **both** `namehash("Vitalik.eth")` (the `text()` node) **and** `namehash("vitalik.eth")` (the `addr()` node).

That last assertion is the finding in one line: **a single `eth_call`, dispatched to a resolver chosen by the attacker's name, asking that resolver for the victim's node.**

#### The non-vacuity control

The `ORACLE` test runs the same capture with the already-canonical `"vitalik.eth"`. The two nodes collapse into one, the attacker's node disappears from the payload entirely, and the DNS-encoded discovery name contains `vitalik` and not `Vitalik`. The divergence is caused by the un-canonicalised param and by nothing else.

### Live reproduction (Sepolia)

{% stepper %}
{% step %}

### Register the attacker name

As attacker, register the label `Vitalik` (capital V) against the ENSv2 registrar **directly** — not through the Manager UI, which lowercases in `features/register-v2/utils/name-parser.ts:46`.
{% endstep %}

{% step %}

### Configure the resolver

Set that name's resolver to a contract you control whose `addr(bytes32,uint256)` returns your address for any node. A dedicated per-name resolver is the ENSv2 default, so this is ordinary usage. Optionally copy `avatar` / `description` / socials from the real `vitalik.eth`.
{% endstep %}

{% step %}

### Open the profile

Open `https://<manager-host>/Vitalik.eth`.
{% endstep %}

{% step %}

### Observe the mismatched data

Observe: **Owner**, **Registered** and **Expires** are the real `vitalik.eth` values; **Main receiving address** is the value your resolver returned for `namehash("vitalik.eth")`.
{% endstep %}
{% endstepper %}

*Degraded variant, stated so it is not discovered as a gap:* if the attacker leaves their name on a stock shared resolver, the address field falls back to whatever that resolver holds for the victim's node, and the finding degrades to full control of the profile's text records, avatar, socials and contenthash under the victim's genuine ownership attestation — still an identity forgery, one band lower.

### Recommendation

Canonicalise once, at the route boundary: in `routes/$name/route.tsx`, normalise the param and `redirect` to the canonical path when it differs, so every downstream reader receives the same string. Until then, make `profileRecordsQuery` take the normalised name exactly as `profileOwnerQuery` / `profileExpiryQuery` / `profileRegistrationQuery` already do, and render an unmissable warning whenever the requested string is not the canonical form — the Explorer's "Not Normalized" badge is the existing precedent.
