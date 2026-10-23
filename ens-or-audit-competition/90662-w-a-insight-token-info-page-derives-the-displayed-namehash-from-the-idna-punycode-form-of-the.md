# 90662 w a insight token info page derives the displayed namehash from the idna punycode form of the name showing the node of a different and separately registrable ens name

**Submitted on Aug 26th 2026 at 08:33:42 UTC by @Another for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90662
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

`/<name>/token` renders a "Normalization" card whose purpose is to hand developers the canonical identifiers for an ENS name. Its Namehash row computes `namehash()` over the output of `asciiEncode()`, which is IDNA/Punycode ToASCII — not the ENS name.

EIP-137 namehash is defined over the UTF-8 bytes of the ENSIP-15-normalised label. Punycode is a different encoding, and `xn--…​.eth` is a different, independently registrable `.eth` name. For every name containing a non-ASCII character, this row therefore displays the node of some other name, and that other name is one an attacker can register.

The bug is invisible for ASCII names, which is consistent with it surviving prior audit rounds: the unit test for `asciiEncode` (`utils/token/ascii.test.ts`) exercises the helper in isolation and never composes it with `namehash`.

## Vulnerability Details

`apps/portal/src/utils/token/ascii.ts` — the helper performs IDNA ToASCII via the WHATWG URL parser:

```ts
export const asciiEncode = (name: string) => {
  try {
    return new URL(`https://${name}`).hostname   // → Punycode, e.g. "xn--ls8h.eth"
  } catch {
    return name
  }
}
```

`apps/portal/src/routes/$name/token.tsx:284-288`:

```ts
const dnsEncode = dnsEncodeName(name)   // ✅ raw name
const ascii = asciiEncode(name)         //    Punycode form
const hash = namehash(ascii)            // ❌ namehash of the Punycode name
```

Rendered at `routes/$name/token.tsx:392-394`, labelled simply "Namehash", inside an `EntityBadge` with an explicit copy affordance:

```tsx
<InfoRow label="Namehash">
  <div>
    <EntityBadge type="content" variant="default" copyValue={hash}>
```

Two corroborating signals that this is a defect rather than a deliberate "show me the IDNA form's hash" feature:

1. Internal inconsistency within the same card. `dnsEncodeName(name)` (line 284) and `escapeUnicode(name)` (lines 363/366) both use the raw `name`. Only `namehash` (line 288) is fed the transformed string.
2. Internal inconsistency within the same page. `TokenV1Name` (`token.tsx:211`) computes the wrapped-name token id correctly:

   ```ts
   const hex = isWrapped ? namehash(name) : labelhash(name.split('.')[0])
   ```

   So for a wrapped V1 unicode name the page simultaneously renders two different 32-byte values, both presented as that name's hash — the "Token ID HEX" row (correct) and the "Namehash" row (incorrect). They disagree.
3. Isolated outlier. Grepping every `namehash(` / `labelhash(` / `labelToCanonicalId(` call site in `apps/portal/src` shows all of them take the raw name or a `getLabel`-derived label. `token.tsx:288` is the only place in the codebase where an ENS identifier is derived from a transformed string.

The row is also silently correct-by-accident for ASCII names (`asciiEncode('vitalik.eth') === 'vitalik.eth'`), which is why it does not surface in ordinary use.

## Impact Details

The namehash (node) is the first argument of essentially every ENS write: `registry.setResolver(node, …)`, `registry.setOwner(node, …)`, `resolver.setAddr(node, …)`, `resolver.setText(node, …)`, and the ERC-1155 token id under `NameWrapper`. This page exists precisely so that developers can copy those identifiers out and use them.

A user viewing `/💩.eth/token` is shown, and offered a one-click copy of, `0x372991e2…802557a5`. That is not their name's node. It is the node of `xn--ls8h.eth`.

Why this is worse than a cosmetic display error: the wrong value is not arbitrary — it is deterministically the node of a real, unregistered, registrable `.eth` name. An attacker can pre-register the Punycode twin of any popular emoji or IDN name and thereby control the exact node that the ENS Explorer tells every visitor is that name's namehash. Consequences for a victim who copies the value into Etherscan's write tab, a deploy script, a Safe transaction builder, or a support ticket:

* Record writes (`setText`, `setAddr`) intended for their own name land on state the attacker owns, and their real name silently keeps its old records — the user believes a record has been updated when it has not.
* Any per-node authorisation or operator approval scoped to the copied node is granted against the attacker's name.
* Off-chain integrations (indexers, allow-lists, gating rules) keyed on the copied node key off the wrong name entirely.

## References

* `apps/portal/src/routes/$name/token.tsx:284-288` (derivation), `:392-394` (render + copy affordance), `:211` (contradicting correct value)
* `apps/portal/src/utils/token/ascii.ts:1-7`
* `apps/portal/src/utils/token/ascii.test.ts` (helper tested in isolation; never composed with `namehash`)
* EIP-137, Ethereum Domain Name Service — Specification (namehash definition)
* ENSIP-15, ENS Name Normalization Standard
* RFC 3492 (Punycode) / UTS #46 (IDNA Compatibility Processing)

## Proof of Concept

Manual reproduction (no build required):

{% stepper %}
{% step %}

## Install and start the Explorer

```sh
pnpm install --frozen-lockfile && pnpm --filter portal dev
```

The Explorer runs on `http://localhost:3001`.
{% endstep %}

{% step %}

## Navigate to the token page

Navigate to `http://localhost:3001/💩.eth/token`.
{% endstep %}

{% step %}

## Compare the displayed values

Read the **Namehash** row of the Normalization card and the **Token ID HEX** row of the token card above it.
{% endstep %}

{% step %}

## Observe the divergence

The two values differ. The Namehash row shows the node of `xn--ls8h.eth`.
{% endstep %}
{% endstepper %}

Automated PoC: — reproduces the divergence directly from the shipped helper:

```js
// poc.mjs   —   npm i viem@2 && node poc.mjs
import { namehash, labelhash } from 'viem/ens'

// verbatim copy of apps/portal/src/utils/token/ascii.ts
const asciiEncode = (name) => {
  try { return new URL(`https://${name}`).hostname } catch { return name }
}

for (const n of ['💩.eth', 'münchen.eth', 'strauß.eth', 'вася.eth', 'vitalik.eth']) {
  const a = asciiEncode(n)
  console.log(`\n  name                        : ${n}`)
  console.log(`  asciiEncode(name)           : ${a}`)
  console.log(`  namehash(name)      CORRECT : ${namehash(n)}`)
  console.log(`  namehash(ascii)   DISPLAYED : ${namehash(a)}`)
  console.log(`  identical?                  : ${namehash(n) === namehash(a)}`)
}

const n = '💩.eth'
console.log('\n--- Same page, two contradictory values for one wrapped V1 name ---')
console.log('Token ID HEX row (token.tsx:211, namehash(name)) :', namehash(n))
console.log('Namehash row     (token.tsx:288, namehash(ascii)):', namehash(asciiEncode(n)))

console.log('\n--- The displayed hash is the node of a real, registrable .eth name ---')
console.log('namehash("xn--ls8h.eth") =', namehash('xn--ls8h.eth'))
console.log('equals displayed value?  =', namehash('xn--ls8h.eth') === namehash(asciiEncode(n)))
console.log('labelhash("xn--ls8h")    =', labelhash('xn--ls8h'), '<- attacker registers this label')
```

Every namehash below is independent of viem: each was recomputed byte-for-byte with a standalone EIP-137 keccak-256 implementation (validated against the canonical vectors `keccak256("") = 0xc5d2…a470` and `namehash("eth") = 0x93cd…c4ae`) and matched exactly — so the logs are reproducible even without installing viem.

Logs (Node v22.22.2, viem 2.x):

```
  name                        : 💩.eth
  asciiEncode(name)           : xn--ls8h.eth
  namehash(name)      CORRECT : 0x3aef7cc933c5fb65036d4ddd389fdf5c65b1fc79e9c1f34655c86e373d974d76
  namehash(ascii)   DISPLAYED : 0x372991e2b0c78e4769330c627c8c1c6657478cd936fb569d81822028802557a5
  identical?                  : false

  name                        : münchen.eth
  asciiEncode(name)           : xn--mnchen-3ya.eth
  namehash(name)      CORRECT : 0xbbb992919e46599352f5c040315d40d6943e02f87e4b18bfb4fe31371c7682b4
  namehash(ascii)   DISPLAYED : 0x6525ac2394c8338e4f37c29f6b194c905276b79cf971c09ca9987f7972ff7da1
  identical?                  : false

  name                        : strauß.eth
  asciiEncode(name)           : xn--strau-pqa.eth
  namehash(name)      CORRECT : 0xe98c2cecd9f124870d0b2eac6725edabe1e8d7665415a75d519aed86b54a41c0
  namehash(ascii)   DISPLAYED : 0xa4629afb7fb0dc4dcbfe2dc08d0e6d9e49a5ef5f4f10d46a7d91364f098610ca
  identical?                  : false

  name                        : вася.eth
  asciiEncode(name)           : xn--80ad0c0c.eth
  namehash(name)      CORRECT : 0x88dc99b760ed222af61c5b1c0713d42eb8142ffec52f580050a02d408222e1a1
  namehash(ascii)   DISPLAYED : 0xa6e61d16886da4eecb104492f3dd7b273e5ef82c5a984535cf1fb255c5c0366c
  identical?                  : false

  name                        : vitalik.eth                      <-- ASCII: bug invisible
  asciiEncode(name)           : vitalik.eth
  namehash(name)      CORRECT : 0xee6c4522aab0003e8d14cd40a6af439055fd2577951148c14b6cea9a53475835
  namehash(ascii)   DISPLAYED : 0xee6c4522aab0003e8d14cd40a6af439055fd2577951148c14b6cea9a53475835
  identical?                  : true

--- Same page, two contradictory values for one wrapped V1 name ---
Token ID HEX row (token.tsx:211, namehash(name)) : 0x3aef7cc933c5fb65036d4ddd389fdf5c65b1fc79e9c1f34655c86e373d974d76
Namehash row     (token.tsx:288, namehash(ascii)): 0x372991e2b0c78e4769330c627c8c1c6657478cd936fb569d81822028802557a5

--- The displayed hash is the node of a real, registrable .eth name ---
namehash("xn--ls8h.eth") = 0x372991e2b0c78e4769330c627c8c1c6657478cd936fb569d81822028802557a5
equals displayed value?  = true
labelhash("xn--ls8h")    = 0x9a8fc2f23e8e206c0c83170a829d7f4bfcee3e44cd2d01518f771018afeb9856 <- attacker registers this label
```
