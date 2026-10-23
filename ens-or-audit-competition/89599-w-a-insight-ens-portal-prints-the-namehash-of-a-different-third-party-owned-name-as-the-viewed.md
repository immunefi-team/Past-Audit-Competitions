# 89599 w a insight ens portal prints the namehash of a different third party owned name as the viewed name s identity

## #89599 \[W\&A-Insight] ENS Portal prints the namehash of a different, third-party-owned name as the viewed name's identity

**Submitted on Aug 20th 2026 at 05:03:28 UTC by @Naresh\_Kandula for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89599
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

## ENS Portal prints the namehash of a different, third-party-owned name as the viewed name's identity

|                   |                                                                                                                                                                                    |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Asset**         | `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal`                                                                                                |
| **File**          | `apps/portal/src/routes/$name/token.tsx:288` (with `apps/portal/src/utils/token/ascii.ts`)                                                                                         |
| **Impact**        | *Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient* |
| **PoC**           | <https://gist.github.com/NOTTIBOY137/85af56821c10a02d845e093f27aeb7f0> — 18/18 passing, plus a standalone script that needs no repo checkout                                       |
| **Prerequisites** | None. No wallet, no authentication, no privileged address, no user interaction beyond opening a URL.                                                                               |

***

### 1. Summary

Every ENS name has a **namehash** — the 32-byte `node` value that is the name's canonical machine identity and the first argument of essentially every ENS contract call (`owner(bytes32)`, `resolver(bytes32)`, `setResolver`, `setOwner`, `setSubnodeOwner`, `setRecord`).

The Portal's Token Info page (`/<name>/token`) renders a row labelled **"Namehash"**. For every name containing a non-ASCII character, **the value in that row is not that name's namehash.** It is the namehash of a *different* label — one that ENS's own normalization standard rejects outright, and which a third party can register and, on mainnet today, does own.

The page derives the value by hashing an **IDNA/punycode** transform of the name instead of the name:

```ts
// apps/portal/src/routes/$name/token.tsx:284-290
const dnsEncode  = dnsEncodeName(name)   // ← raw name          (sibling row, correct input)

const ascii      = asciiEncode(name)     // ← WHATWG URL parser ⇒ IDNA punycode

const hash       = namehash(ascii)       // ← punycode fed to namehash   ★ THE DEFECT

const normalized = isNormalized(name)    // ← renders the green ✓ "Normalized" badge
```

```ts
// apps/portal/src/utils/token/ascii.ts  (complete file)
export const asciiEncode = (name: string) => {
  try {
    return new URL(`https://${name}`).hostname
  } catch {
    return name
  }
}
```

`new URL().hostname` applies the WHATWG URL standard's **UTS-46/IDNA** transform, which emits punycode (`café.eth` → `xn--caf-dma.eth`). ENS nodes are `namehash()` over the **ENSIP-15** form (UTS-46 + NFC, *never* punycode). Two different strings produce two different nodes.

`asciiEncode` is not a badly-conceived helper — it is correct, and it is legitimately rendered one row above as the **"ASCII"** row (`token.tsx:372-377`), where punycode is exactly the right thing to display. The defect is that line 288 reuses that *display* variable as the *hashing* input.

**No remedy exists for anyone.** The name's owner holds no key, role, or token that changes what this page prints, and there is nothing on-chain to revoke or re-issue. This is not a pre-existing or revocable relationship — there is nothing for any party to undo.

***

### 2. Root cause

Two normalization standards are in play and the page uses the wrong one for hashing:

|               | ENSIP-15 (correct for ENS)                       | UTS-46 / IDNA (what the URL parser applies)       |
| ------------- | ------------------------------------------------ | ------------------------------------------------- |
| purpose       | canonicalising an **ENS name** before hashing    | canonicalising a **DNS hostname** for the network |
| `café.eth` →  | `café.eth` (already canonical)                   | `xn--caf-dma.eth`                                 |
| `🚀.eth` →    | `🚀.eth` (emoji are valid ENS labels)            | `xn--158h.eth`                                    |
| `xn--caf-dma` | **rejected** — `invalid label extension: "xn--"` | the canonical ASCII form                          |

Punycode is not an alternate spelling of an ENS name. ENSIP-15 refuses the `xn--` label extension, so the punycode string is a **separate, separately-registrable label** with its own independent node and its own independent owner.

Consequently `namehash(asciiEncode(name))` does not return "the same node, written differently". It returns the node of an unrelated name.

***

### 3. Evidence 1 — the measured divergence

Running the production expression `namehash(asciiEncode(n))` against the true node `namehash(ens_normalize(n))`, using the repository's own pinned `viem@2.52.2` and `@adraffy/ens-normalize`:

| name viewed                     | `asciiEncode` output | **Namehash the page prints**                                         | the name's true node                                                 | equal     |
| ------------------------------- | -------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------- | --------- |
| `café.eth`                      | `xn--caf-dma.eth`    | `0xdd27588bc3afa51ad47246943c881d01914fbfd190c3393ba61c3cc1aa7a1c3f` | `0xa7369e1df22e06ec6d91162508e400d7af475860638f927e6d1085bb0134a74a` | **false** |
| `🚀.eth`                        | `xn--158h.eth`       | `0x6a0029f351e009b8336f30a46e6688681d33c64c57bb83e1e9507db8cdea6dca` | `0xfb4826c8682290c58ce9375c4cac0a4904d6b3b237684b60e0a6eaba308eac62` | **false** |
| `日本.eth`                        | `xn--wgv71a.eth`     | `0xdfc13be7c6f0bc5d7cc010ee079d0eb22b9721458473134477df487dfcd868e0` | `0xe0de267e9e2295c296d2afbc29b2a3a39227a8111e1b4dab77803488ec096613` | **false** |
| `vitalik.eth` *(ASCII control)* | `vitalik.eth`        | `0xee6c4522aab0003e8d14cd40a6af439055fd2577951148c14b6cea9a53475835` | *(identical)*                                                        | true      |
| `nick.eth` *(ASCII control)*    | `nick.eth`           | `0x05a67c0ee82964c4f7394cdd47fee7f4d9503a23c09c38341779ea012afe6e00` | *(identical)*                                                        | true      |

Both ASCII controls reproduce the **publicly known** `vitalik.eth` and `nick.eth` nodes. That anchors the right-hand column to ground truth and shows the harness is measuring correctly rather than reporting everything as broken.

The label the page hashed is rejected by the project's own normalizer:

```
ens_normalize('xn--caf-dma') -> ERROR: invalid label extension: "xn--"
ens_normalize('xn--158h')    -> ERROR: invalid label extension: "xn--"
ens_normalize('xn--wgv71a')  -> ERROR: invalid label extension: "xn--"
```

***

### 4. Evidence 2 — the node the page prints belongs to someone else

Read-only `owner(bytes32)` calls against the mainnet ENS registry `0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e`:

| node                                      | owner                                                                       |
| ----------------------------------------- | --------------------------------------------------------------------------- |
| `vitalik.eth` — **control**               | `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` ✅ matches ground truth         |
| **the node the page prints for `🚀.eth`** | **`0x671028A812aB5F2feEb8982ac23B20708517C1fF`** — an unrelated third party |
| `🚀.eth`'s true node                      | `0x0000…0000` — unowned                                                     |
| **`café.eth`'s true node**                | **`0xF138984Bd659edd8DE38fb2FebEdae8bED5E0255`** — a real registered owner  |
| the node the page prints for `café.eth`   | `0x0000…0000` — unowned                                                     |

The failure runs in both directions:

* For **`🚀.eth`**, the Portal presents as that name's identity a node that is **owned by a third party**.
* For **`café.eth`** — a registered name with a real owner — the Portal presents a node owned by **nobody**.

The printed value is therefore not an inert wrong string. It is a live node that resolves to a *different party*.

***

### 5. Evidence 3 — the page contradicts itself, three ways, in one render

This is what forecloses a "working as intended" reading: the application already disagrees with itself on screen.

#### 5a. It certifies the name as canonical while hashing a non-canonical string

The same card renders a green ✓ **"Normalized"** badge, because `isNormalized('café.eth')` returns `true` (measured) — and immediately prints, as that normalized name's namehash, the node of a label `ens_normalize` refuses to accept.

```ts
// apps/portal/src/utils/token/isNormalized.ts
const isNormalizedLabel = (label: string) => {
  try { return ens_normalize(label) === label } catch { return false }
}
export const isNormalized = (name: string) =>
  name.split('.').every((label) => isEncodedLabelhash(label) || isNormalizedLabel(label))
```

Measured: `isNormalized('café.eth') === true`, `isNormalized('🚀.eth') === true`, `isNormalized('日本.eth') === true` — all render the green badge.

#### 5b. Two mutually exclusive namehashes for one name, in one file

```ts
// token.tsx:211  — the Token ID row
const hex = isWrapped ? namehash(name) : labelhash(name.split('.')[0])   // raw name — correct
// token.tsx:288  — the Namehash row
const hash = namehash(ascii)                                            // punycode — wrong
```

`RouteComponent` renders the protocol-version card and then the Normalization card **unconditionally**:

```tsx
// token.tsx:304-312
{data?.protocolVersion === 'ENSv1' ? (
  <TokenV1Name name={name} />
) : data?.protocolVersion === 'ENSv2' ? (
  <TokenV2Name name={name} registryAddress={data.registryAddress} />
) : null}

<InfoCard title="Normalization" …>
```

so both values appear on the same screen. For a **wrapped ERC-1155 name the token ID&#x20;*****is*****&#x20;the namehash** — definitionally one and the same quantity — yet the page prints two different values for it.

#### 5c. The Owner shown and the Namehash shown refer to different nodes

The owner comes from `resolveEnsOwner` → ensjs `getOwner({ name })` on the **raw** name, i.e. the true node. The Namehash row is a different node. The page therefore reports *ownership of node A* directly above *identity = node B*. Both cannot be true.

#### Sibling asymmetry

`dnsEncodeName(name)` two lines above uses the raw name, and **all five other `namehash()` call sites in `apps/portal/src`** use the raw or normalized name:

```
features/ownership/components/V1NameManagerRecord.tsx:56   namehash(name)
features/history/hooks/useNameHistoryTimeline.ts:234       namehash(normalizedName)
routes/$name/resolver.tsx:170                              namehash(name)
routes/$name/token.tsx:211                                 namehash(name)
lib/roles/resolverRoles.ts:140                             namehash(name)
routes/$name/token.tsx:288                                 namehash(ascii)   ← the only punycode one
```

***

### 6. Reachability

* The route `/$name/token` is registered in `routeTree.gen.ts` and linked from `ProfileSidebar.tsx:124` (`to: '/$name/token'`) — one click from any name page.
* The parent route `apps/portal/src/routes/$name.tsx` `beforeLoad` redirects **TLDs only**. It applies no `isValidEnsName` gate — unlike the sibling route `routes/$name/index.tsx:442`, which does call it.
* The Normalization card renders **outside** the `data?.protocolVersion` conditional, so it renders for registered and unregistered names alike.
* The owner lookup is an on-chain read, so a name with no owner yields *no owner* rather than an error.
* Anonymous: no wallet, no authentication, no privileged address.

The affected population is every non-ASCII ENS name — all emoji names and every name in a non-Latin script.

***

### Link to Proof of Concept

<https://gist.github.com/NOTTIBOY137/85af56821c10a02d845e093f27aeb7f0>

### Proof of Concept

### Proof of Concept

### 7. Steps to reproduce

#### A. Standalone — no repo checkout, \~60 seconds

```bash
mkdir ens-namehash-poc && cd ens-namehash-poc
npm init -y >/dev/null
npm i viem@2.52.2 @adraffy/ens-normalize
# download repro-standalone.mjs from the PoC gist, then:
node repro-standalone.mjs
```

Or inline, the minimal core:

```js
import { namehash } from 'viem'
import { normalize } from 'viem/ens'

// verbatim apps/portal/src/utils/token/ascii.ts
const asciiEncode = (n) => { try { return new URL(`https://${n}`).hostname } catch { return n } }

const name = 'café.eth'
console.log('page prints:', namehash(asciiEncode(name)))  // token.tsx:288
console.log('true node  :', namehash(normalize(name)))    // what it should be
// -> 0xdd27588bc3afa51ad47246943c881d01914fbfd190c3393ba61c3cc1aa7a1c3f
// -> 0xa7369e1df22e06ec6d91162508e400d7af475860638f927e6d1085bb0134a74a
```

#### B. In the repository — the full test, 18/18

```bash
cd apps/portal
pnpm exec vitest run 'src/routes/$name/poc-punycode-namehash.test.ts'
```

The test imports the **real** `asciiEncode` and `isNormalized` from the application — nothing is reimplemented.

#### C. Confirm the printed node has a real, unrelated owner (read-only)

```bash
REG=0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e
RPC=https://eth.drpc.org

# the node the page prints for 🚀.eth  -> an unrelated third party
cast call $REG "owner(bytes32)(address)" \
  0x6a0029f351e009b8336f30a46e6688681d33c64c57bb83e1e9507db8cdea6dca --rpc-url $RPC
# 0x671028A812aB5F2feEb8982ac23B20708517C1fF

# the TRUE node for 🚀.eth -> unowned
cast call $REG "owner(bytes32)(address)" \
  0xfb4826c8682290c58ce9375c4cac0a4904d6b3b237684b60e0a6eaba308eac62 --rpc-url $RPC
# 0x0000000000000000000000000000000000000000

# control: vitalik.eth
cast call $REG "owner(bytes32)(address)" \
  0xee6c4522aab0003e8d14cd40a6af439055fd2577951148c14b6cea9a53475835 --rpc-url $RPC
# 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
```

#### D. In the running application

1. Open `/café.eth/token` (or any emoji / non-Latin name).
2. Read the **"Namehash"** row.
3. Compare it against the green ✓ **"Normalized"** badge on the same card, and — for a wrapped name — against the **Token ID (hex)** value, which is definitionally the same quantity.

***

### 8. PoC output

```
✓ TIER 1 — asciiEncode is an IDNA/punycode transform (2)
✓ TIER 2 — the rendered Namehash is not the name (6)
✓ TIER 3 — the punycode label is a distinct name, not an alias (4)
✓ TIER 4 — the card certifies the name it fails to hash (3)
✓ TIER 5 — token.tsx disagrees with itself (3)

Test Files  1 passed (1)
     Tests  18 passed (18)
```

The suite carries four independent oracles, so a passing run cannot be vacuous:

1. `vitalik.eth` and `nick.eth` reproduce their **published** nodes — the true column is ground-truthed.
2. `asciiEncode('vitalik.eth') === 'vitalik.eth'` — the transform is not mangling everything.
3. `VITALIK.eth` yields the **same** value in both columns — case folding is handled correctly here, so this is not an un-canonicalised-parameter defect (see §9).
4. Each punycode label hashes to exactly the node the page printed — the printed value is a real, ownable node, not garbage.

***

### 9. Why this is not a duplicate

#### Not EXP-INPUT-005

That disclosed finding is that name **validators** accept homoglyph / bidi / control characters, producing **display confusion**, and it expressly reserves that *"incorrect **resolution** would be new."* This is that reserved case:

* No validator is involved, and no look-alike string exists. `café.eth` and `🚀.eth` are plain, fully ENSIP-15-canonical names — the page's own `isNormalized()` correctly returns `true` for both.
* The defect is the **hashing standard** used to derive the node, i.e. name → node resolution.

#### Not report #89408

\#89408 is the Manager's `routes/$name/route.tsx`: a URL parameter that is never canonicalised and is then hashed three different ways by three different readers. It reproduces on plain ASCII `/Vitalik.eth`.

**This page is immune to that exact vector.** Measured: `VITALIK.eth` yields identical values in both columns here, because the URL parser lowercases. Whereas `café.eth` — already canonical, with nothing left to canonicalise — still diverges. Different application, different file, a single hashing site using the *wrong standard* rather than several sites disagreeing with one another. A fix for #89408 does not fix this.

#### Not covered elsewhere

No record in the published known-issues set (25 security findings, 10 QA issues, 25 tracker entries) concerns namehash derivation, punycode, IDNA or UTS-46. There is no `by design` / accepted-risk annotation in `ascii.ts`, `ascii.test.ts` or `token.tsx`. `ascii.test.ts` asserts `asciiEncode`'s **own** contract (that it emits punycode) — which is correct for the ASCII row — and says nothing about using it as a hashing input.

***

### 10. On the scope of the impact

The impact clause reads *"…display or pre-fill an incorrect address, name, or transaction detail, **so that a user sends funds or submits a transaction to an unintended recipient**."* This page displays rather than auto-submits, so it is worth meeting that second half directly instead of leaving it implicit.

**A namehash is a transaction detail — specifically, it is the one every ENS write begins with.** `node` is the first argument of `setResolver(bytes32,address)`, `setOwner(bytes32,address)`, `setSubnodeOwner`, `setRecord`, `resolver(bytes32)` and `owner(bytes32)`. Telling a user which node belongs to a name is the entire product function of the page they are on, and the Portal is the reference explorer for that question.

**The page supplies the mechanism for carrying it into a write.** The row is rendered as `<EntityBadge … copyValue={hash}>` — a one-click copy control whose only purpose is to move that value somewhere it will be used.

**The unintended recipient is concrete, not hypothetical.** Anyone who takes the node this page gives for `🚀.eth` and submits it as the `node` argument of an ENS write is operating on the node owned by `0x671028A812aB5F2feEb8982ac23B20708517C1fF` — a party unrelated to the name they looked up. That is the "unintended recipient" the clause describes, expressed in the identifier this page exists to provide.

The residual limitation, stated once and priced: a human or an integrator has to act on the value for funds to move. What removes the usual mitigations is that nothing warns them — the same card certifies the name as ✓ Normalized, and no party, including the name's owner, holds anything that can correct or revoke what is shown.

***

### 11. Recommended fix

```diff
  const ascii = asciiEncode(name)
- const hash  = namehash(ascii)
+ const hash  = namehash(name)
```

viem's `namehash` already applies ENSIP-15 normalization. `asciiEncode` should be retained for the **ASCII** row (`token.tsx:372-377`), where punycode is the correct value to display.

A regression test worth adding alongside it: assert that for at least one emoji name and one non-Latin name the rendered Namehash equals `namehash(name)`, and that it equals the Token ID hex for a wrapped name.

***

### Appendix — full PoC source

Both files are in the PoC gist: <https://gist.github.com/NOTTIBOY137/85af56821c10a02d845e093f27aeb7f0>

* `poc-punycode-namehash.test.ts` — the 18/18 vitest suite; drop it at `apps/portal/src/routes/$name/poc-punycode-namehash.test.ts` and run `pnpm exec vitest run 'src/routes/$name/poc-punycode-namehash.test.ts'`.
* `repro-standalone.mjs` — a standalone script requiring only `viem` and `@adraffy/ens-normalize`; prints the divergence, the controls, the rejected labels, the self-contradiction, and the on-chain commands.
