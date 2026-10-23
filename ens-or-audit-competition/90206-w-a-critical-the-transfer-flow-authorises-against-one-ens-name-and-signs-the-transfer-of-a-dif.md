# 90206 w a critical the transfer flow authorises against one ens name and signs the transfer of a different one

## #90206 \[W\&A-Critical] The transfer flow authorises against one ENS name and signs the transfer of a different one

**Submitted on Aug 23rd 2026 at 17:59:29 UTC by @Naresh\_Kandula for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90206
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs
  * Direct theft of user funds

### Description

## The transfer flow authorises against one ENS name and signs the transfer of a different one

**Severity:** High **Target asset:** `apps/portal` — Explorer app **Revision tested:** `63772fd872af472ced58b009499355f3430c2a86`

### Impact-clause mapping

> *"Direct theft of user NFTs / user funds."*

and

> *"Causing the application to display or pre-fill an incorrect address, name, or transaction detail so a user sends funds / submits a tx to an unintended recipient."*

Both are quoted in full. The victim signs `safeTransferFrom` of an ERC-1155 ENS name they did not intend to part with, having been shown a different name on every screen and in every transaction description.

A third clause is a near-fit and I am deliberately **not** leaning on it: *"Hijacking or bricking an ENS name during a **Manager** flow (registration, v1→v2 migration, transfer) … leaving a name permanently inaccessible."* The behaviour matches its wording closely — ownership, registry and resolver all end up assigned away — but that clause names the **Manager**, and this finding is in the Explorer. I flag the mismatch rather than argue past it.

***

### 1. Summary

`/$name/index` is gated. `routes/$name/index.tsx:442,458` calls `isValidEnsName(name)` and renders `<InvalidNameMessage title="Invalid name" />` for anything that is not ENSIP-15-normalised — the comment there states the intent: *"Validate name format - must be a valid ENS name (normalized + ends with .eth)"*.

**That gate exists on no sibling route under `/$name/**`.** On the ungated transfer route the two halves of the operation address two different on-chain names:

* **AUTHORISE — raw.** `routes/$name/ownership/transfer.tsx:27` resolves the owner of the **raw path parameter**, ending in `findOwner(toHex(packetToBytes(name)))` — raw UTF-8 label bytes, no normalisation.
* **SIGN — normalised.** `features/transfer/hooks/useTransferName.ts:112-115,132` derives the label with `getLabel(name)` = `ens_normalize` + `ens_split`, and that label produces the token id transferred and the resource for both registry writes.

So the app proves the connected wallet owns the name spelled in the URL, then emits `setResolver(0x0)`, `setSubregistry(0x0)` and `safeTransferFrom` against a **different** name.

### 2. Root cause

`apps/portal/src/routes/$name/ownership/transfer.tsx:16-19` — the entire route definition:

```ts
export const Route = createFileRoute('/$name/ownership/transfer')({
  component: RouteComponent,
  notFoundComponent: () => <NotFoundMessage />,
})
```

No `beforeLoad`, no `params.parse`, no validation of any kind.

The shared layout does not compensate. `apps/portal/src/routes/$name.tsx:25-45`:

```ts
beforeLoad: async ({ params }) => {
  const { name } = params
  // Redirect valid TLDs to the dedicated /tld/$tld route
  if (isValidEnsName(name) && isTLD(name)) { ... }
  const ownerData = await queryClient.fetchQuery(getEnsOwnerQueryOptions({ name }))
  ...
}
```

Its only `isValidEnsName` call is `&&`-guarded by `isTLD(name)`, so for a 2LD the condition short-circuits and the check **can never reject**. The layout then fetches the owner of the raw name unconditionally.

**This is the shipped pattern being omitted, not a pattern the team never wrote.** Three sibling route families do validate their parameter in `beforeLoad` and redirect on failure — `routes/addr/$addr.tsx:13-17`, `routes/resolver/$address.tsx:13`, `routes/registry/$address.tsx:13`.

### 3. The two derivations, traced

**Authorise (raw).** `transfer.tsx:22` `Route.useParams()` → `:27` `useQuery(getEnsOwnerQueryOptions({ name }))` → `features/profile/hooks/useEnsOwner.ts` `getEnsOwner` → `utils/ens/resolveEnsOwner.ts:48-50` `getOwnerV2(client, { name })` and `getNameRegistries(client, { name })` → ensjs `dist/actions/public/v2/getOwner.js:24`:

```js
functionName: 'findOwner',
args: [toHex(packetToBytes(name))],
```

The authorisation decision is `transfer.tsx:104` `isAddressEqual(address, data.owner)`, which renders `<SendNameForm name={name} registryAddress={registryAddress} owner={owner} />` at `:196`.

**Sign (normalised).** `utils/token/getLabel.ts:10-19`:

```ts
export function getLabel(name: string): string {
  const normalized = ens_normalize(name)
  const labels = ens_split(normalized)
  ...
}
```

`features/transfer/hooks/useTransferName.ts:112-115` `getEnsTokenId({ label: getLabel(name), registryAddress })` → ensjs `getTokenId.js:21` `BigInt(labelhash(label))`; `:132` `const label = getLabel(name)`; `:137` detach resolver, `:140` detach registry, `:143` `transferToken({ registryAddress, tokenId, recipient, id })`.

The transfer-permission check agrees with the **sign** side, not the authorise side — `features/transfer/hooks/useCanTransferName.ts:59` also uses `getLabel(name)` — so nothing anywhere in the flow ever compares the two.

**A fourth derivation, in the opposite direction.** `features/transfer/helpers/setEthAddress.ts:37-43` uses the **raw** `name` for `getResolver` and `createSetForwardResolutionRequest`. So the "point the ETH address at the recipient" step writes on the junk name while the other three steps write on the real one — the one option presented as protecting the user is applied to the wrong name.

### 4. Measured, not asserted

Measured with the repository's pinned `viem` and `@adraffy/ens-normalize`:

| URL parameter         | authorise — `packetToBytes(raw)` | sign — `getLabel` | same name? |
| --------------------- | -------------------------------- | ----------------- | ---------- |
| `cool.eth`            | `0x04636f6f6c0365746800`         | `cool`            | **yes**    |
| `COOL.eth`            | `0x04434f4f4c0365746800`         | `cool`            | **no**     |
| `cool<U+200B>.eth`    | `0x07636f6f6ce2808b0365746800`   | `cool`            | **no**     |
| `caf<U+0301>e…` (NFD) | `0x0663616665cc810365746800`     | `café` (composed) | **no**     |

**Plain uppercase is sufficient** — no invisible or confusable character is required. `isValidEnsName('COOL.eth')` is `false`, which is exactly why the gated sibling route rejects it and why this route should have too.

### 5. Attacker and reach

**Attacker: any unauthenticated user.** No app account, no privileged address, no RPC control.

**Setup is a single transaction.** The registrar's `register` takes the beneficiary as an argument — `@ensdomains/ensjs-abi/src/v2/ethRegistrar.ts:244-258`:

```
register(label string, owner address, secret bytes32, subregistry address,
         resolver address, duration uint64, paymentToken address, referrer bytes32)
```

so the attacker registers the confusable label with `owner` = **the victim**, and with non-zero `subregistry` and `resolver`, in one call. No unsolicited ERC-1155 transfer is involved, and setting those two fields forces both destructive detach toggles to be *offered* — their visibility is computed from the **raw** name (`features/transfer/hooks/useTransferDetachTargets.ts:76-95`) and they default to on (`features/transfer/components/SendNameForm.tsx:69-73`).

**The victim's actions are entirely ordinary**: open a link (or click the row for the new name in their own names table), and transfer a name they own. Nothing here is an "un-prompted action outside normal app workflows".

**The route is reachable even when the name is displayed as invalid.** `components/ProfileSidebar.tsx:79-81` renders the "Ownership" link with the raw name and lives in the layout (`routes/$name.tsx:65`), so it renders alongside the `$name/index.tsx:458` "Invalid name" message; from there `routes/$name/ownership/index.tsx:114-119` offers Transfer with no validity gate.

**Everything the victim sees is the raw name** — the Transfer button, `transferToken.ts:88` `` `Transfer ${name}` ``, `detachNameResolver.ts:91`, `detachNameRegistry.ts:92`, and `useTransferName.ts:177`. The wallet prompt shows only an opaque `uint256` token id.

### 6. Impact

The victim loses the ERC-1155 token for their real name, and immediately before that its resolver pointer and subregistry pointer are set to zero — so the real name stops resolving and its subnames are orphaned even if the token itself is later returned.

The cleanest theft shape: the attacker places the confusable name in the victim's wallet, offers to buy "that name" from them, and receives the **real** name when the victim transfers. The same defect also produces pure loss with no counterparty — a victim moving an unfamiliar airdropped name to their own second wallet moves their real name instead.

### Proof of Concept

### 7. Steps to reproduce

1. `pnpm install`
2. Create `apps/portal/src/features/transfer/transferNameSplit.poc.test.ts` with the source in §8.
3. `cd apps/portal && TZ=UTC pnpm vitest run src/features/transfer/transferNameSplit.poc.test.ts`

No network, no chain, no deployed infrastructure. The chain mocks record **which name they were asked about**, so the test cannot manufacture the split — whichever derivation the application uses is the one that shows up.

### 8. Proof of concept

`apps/portal/src/features/transfer/transferNameSplit.poc.test.ts`

```ts
/**
 * The transfer flow authorises against one ENS name and signs against another.
 *
 * `/$name/index` is gated — `routes/$name/index.tsx:442,458` runs
 * `isValidEnsName(name)` and renders `<InvalidNameMessage/>` for anything that
 * is not ENSIP-15-normalised. **That gate exists on no sibling route.**
 * `routes/$name/ownership/transfer.tsx:16-19` declares only `component` and
 * `notFoundComponent`, and the shared layout `routes/$name.tsx:25-45` only uses
 * `isValidEnsName` inside `isValidEnsName(name) && isTLD(name)`, a TLD-redirect
 * condition that can never reject a 2LD.
 *
 * On that ungated route the two halves of the operation disagree:
 *
 *   - AUTHORISE — `transfer.tsx:27` `getEnsOwnerQueryOptions({ name })` with the
 *     RAW path param -> `resolveEnsOwner.ts:48-50` `getOwnerV2(client,{name})`
 *     -> ensjs `getOwner.js:24` `findOwner(toHex(packetToBytes(name)))`.
 *     Raw UTF-8 label bytes, zero normalisation.
 *
 *   - SIGN — `useTransferName.ts:112-115` `getEnsTokenId({label:getLabel(name)})`
 *     and `:132-143` `const label = getLabel(name)` feeding detach-resolver,
 *     detach-registry and transfer-token. `getLabel` (`utils/token/getLabel.ts:10-19`)
 *     is `ens_normalize` + `ens_split`.
 *
 * So the app proves the connected wallet owns the name spelled in the URL, then
 * emits `safeTransferFrom` / `setResolver(0)` / `setSubregistry(0)` against the
 * NORMALISED name — a different ERC-1155 token.
 *
 * Positive 1: authorise-name and sign-name are different on-chain names.
 * Positive 2: the token id actually handed to the transfer step is the
 *             normalised name's, not the authorised name's.
 * Oracle 1  : an already-normalised spelling produces NO split.
 * Oracle 2  : the sibling `/$name/` route's gate really would have rejected the
 *             spelling — i.e. the protection exists and is simply absent here.
 */

import { ens_normalize } from '@adraffy/ens-normalize'
import { labelhash, toHex } from 'viem'
import { packetToBytes } from 'viem/ens'
import { beforeEach, describe, expect, it, vi } from 'vitest'

const getOwnerV2Mock = vi.fn()
const getNameRegistriesMock = vi.fn()
const getTokenIdMock = vi.fn()

vi.mock('@ensdomains/ensjs/public/v2', async (importOriginal) => {
  const actual = await importOriginal<Record<string, unknown>>()
  return {
    ...actual,
    getOwner: (...a: unknown[]) => getOwnerV2Mock(...a),
    getNameRegistries: (...a: unknown[]) => getNameRegistriesMock(...a),
    getTokenId: (...a: unknown[]) => getTokenIdMock(...a),
  }
})
vi.mock('@/lib/wagmi/helpers', async () => {
  const { ok } = await import('neverthrow')
  const { sepoliaWithEns } = await import('@ens-apps/indexer/chain')
  // A client with the real ENS-extended chain, but no transport: every chain
  // answer in this test comes from the ensjs mocks above.
  return { safeGetClient: () => ok({ chain: sepoliaWithEns }) }
})

const OWNER = '0x1111111111111111111111111111111111111111' as const
const REGISTRY = '0x2222222222222222222222222222222222222222' as const

/** Plain uppercase. No invisible character is needed on this route. */
const RAW_UPPERCASE = 'COOL.eth'
/** U+200B, for completeness. */
const RAW_INVISIBLE = 'cool​.eth'

const labelOf = (name: string) => name.slice(0, -'.eth'.length)

/** The exact bytes the authorisation read sends to UniversalResolver.findOwner. */
const authorisedAs = async (routeParam: string) => {
  // `transfer.tsx:27` uses getEnsOwnerQueryOptions, whose queryFn is exactly
  // `getEnsOwner(params)` — call the production function directly.
  const { getEnsOwner } = await import('@/features/profile/hooks/useEnsOwner')
  await getEnsOwner({ name: routeParam })
  const asked = getOwnerV2Mock.mock.calls.at(-1)?.[1] as { name: string }
  return toHex(packetToBytes(asked.name))
}

/** The label the signing path derives for the token id and both detach writes. */
const signedAs = async (routeParam: string) => {
  const { getLabel } = await import('@/utils/token/getLabel')
  const { getEnsTokenId } = await import('@/features/profile/hooks/useTokenId')
  await getEnsTokenId({ label: getLabel(routeParam), registryAddress: REGISTRY })
  return (getTokenIdMock.mock.calls.at(-1)?.[1] as { label: string }).label
}

beforeEach(() => {
  vi.clearAllMocks()
  getOwnerV2Mock.mockResolvedValue(OWNER)
  getNameRegistriesMock.mockResolvedValue([REGISTRY, REGISTRY])
  getTokenIdMock.mockResolvedValue(1n)
})

describe('transfer: the authorised name and the signed name are not the same name', () => {
  it.each([
    ['plain uppercase', RAW_UPPERCASE],
    ['U+200B ZERO WIDTH SPACE', RAW_INVISIBLE],
  ])(
    'POSITIVE (%s) - authorise-name and sign-name are different on-chain names',
    async (_tag, raw) => {
      const authBytes = await authorisedAs(raw)
      const signedLabel = await signedAs(raw)

      // What the authorisation read actually asked about.
      expect(authBytes).toBe(toHex(packetToBytes(raw)))

      // What the signing path will write against — a different name.
      const signedBytes = toHex(packetToBytes(`${signedLabel}.eth`))
      expect(signedBytes).not.toBe(authBytes)
      expect(labelhash(signedLabel)).not.toBe(labelhash(labelOf(raw)))
    },
  )

  it.each([
    ['plain uppercase', RAW_UPPERCASE],
    ['U+200B ZERO WIDTH SPACE', RAW_INVISIBLE],
  ])(
    'POSITIVE (%s) - the token id handed to the transfer step is NOT the authorised name',
    async (_tag, raw) => {
      const authBytes = await authorisedAs(raw)
      const signedLabel = await signedAs(raw)

      // `useTransferName.ts:112-115` passes exactly this label to getEnsTokenId,
      // and `:143` transfers the resulting token id. Compare it against the name
      // the ownership check actually asked about — not against the URL string —
      // so this assertion tracks the SPLIT and not merely the fact that
      // getLabel normalises.
      expect(signedLabel).toBe(ens_normalize(raw).slice(0, -'.eth'.length))
      expect(toHex(packetToBytes(`${signedLabel}.eth`))).not.toBe(authBytes)
    },
  )

  it('ORACLE - an already-normalised spelling produces NO split', async () => {
    const raw = 'cool.eth'
    const authBytes = await authorisedAs(raw)
    const signedLabel = await signedAs(raw)

    expect(toHex(packetToBytes(`${signedLabel}.eth`))).toBe(authBytes)
    expect(labelhash(signedLabel)).toBe(labelhash(labelOf(raw)))
  })

  it('ORACLE - the sibling route’s shipped gate WOULD have rejected these', async () => {
    const { isValidEnsName } = await import('@/utils/token/isNormalized')

    // `routes/$name/index.tsx:442,458` runs exactly this check and renders
    // <InvalidNameMessage/>. The protection exists; it is absent on the
    // transfer route.
    expect(isValidEnsName(RAW_UPPERCASE)).toBe(false)
    expect(isValidEnsName(RAW_INVISIBLE)).toBe(false)
    expect(isValidEnsName('cool.eth')).toBe(true)
  })
})
```

#### Output — unmodified production code

```
 Test Files  1 passed (1)
      Tests  6 passed (6)
```

#### What it asserts

* **POSITIVE ×2** (uppercase, `U+200B`) — the bytes the authorisation read sends to `findOwner` and the name the signing path derives are two different on-chain names, with different labelhashes.
* **POSITIVE ×2** (uppercase, `U+200B`) — the label handed to `getEnsTokenId` (and therefore the token id `transferToken` moves) is **not** the name the ownership check authorised. Asserted against the name the authorisation actually asked about, not against the URL string, so it tracks the split rather than merely restating that `getLabel` normalises.
* **ORACLE (non-vacuity)** — an already-normalised spelling produces no split: authorise-bytes and sign-bytes are equal and the labelhashes match.
* **ORACLE (the protection exists)** — `isValidEnsName` returns `false` for both attack spellings and `true` for the canonical one. This is the same function the sibling route calls at `routes/$name/index.tsx:442`, so it demonstrates the shipped gate *would* have rejected these inputs and is simply not invoked here.

#### Negative control — executed in-tree, then reverted

Authorise against the same canonical name the writes derive from, in `features/profile/hooks/useEnsOwner.ts`:

```ts
  const canonicalName = ens_normalize(name)
  const result = yield* fromPromise(
    resolveEnsOwner(client, canonicalName),
```

With that change to production code:

```
      Tests  4 failed | 2 passed (6)

AssertionError: expected '0x04636f6f6c0365746800' to be '0x04434f4f4c0365746800'
AssertionError: expected '0x04636f6f6c0365746800' to be '0x07636f6f6ce2808b0365746800'
AssertionError: expected '0x04636f6f6c0365746800' not to be '0x04636f6f6c0365746800'
```

**All four defect assertions fail** — the authorisation now addresses `cool.eth` (`0x0463…`) instead of the raw `COOL.eth` (`0x0443…`), so authorise and sign finally agree — **and both non-vacuity oracles still pass**, which is what separates a fix from a test that merely broke. Reverted afterwards; production `git diff` is empty and the positive run returns to 6/6.

⚠ One note on building this control: an earlier version of the second assertion compared the signed label against the raw URL string, which passes with **and** without the fix — it restated that `getLabel` normalises rather than testing the split. It is written against the authorised name for that reason.

#### Regression

Full Explorer suite, `TZ=UTC`: **151 files / 1471 tests, all passing.** Production `git diff` is empty; the only change to the tree is the single added `*.poc.test.ts` file above.

### 9. Novelty

**Not the `/$name` profile split.** I have previously reported a name-canonicalisation split on the *profile* page (owner read normalised, records read raw). That is a different route, a different pair of derivations, and a read-only display defect. This is `/$name/ownership/transfer`, `packetToBytes(raw)` vs `labelhash(ens_normalize(name))`, and three **write** sinks. One fix does not resolve both.

**Not EXP-INPUT-005** (*"name validators accept homoglyph/bidi/control chars"*, impact framed as **display confusion only**, with the stated carve-back that *incorrect resolution would exceed it*). Here the validator is **correct** — `isValidEnsName('COOL.eth')` is `false`, asserted by the project's own test at `utils/token/isNormalized.test.ts` — it is simply never invoked on this route. The impact is a state-changing on-chain write against a different name, not display.

**The program's own carve-back applies verbatim:** *"a bypass of a shipped fix is a new finding."* The fix is shipped at `routes/$name/index.tsx:442,458`; this report is that it is absent on every sibling.

**Not WEB-1249** (subname creation orphaning nested subnames) — different flow, different mechanism.

### 10. Recommended fix

Reject a non-normalised `$name` once, in the shared layout, so every route under `/$name/**` inherits it — matching what `routes/addr/$addr.tsx:13-17`, `routes/resolver/$address.tsx:13` and `routes/registry/$address.tsx:13` already do for their parameters:

```ts
// routes/$name.tsx, in beforeLoad, before any owner fetch
if (!isValidEnsName(name)) throw notFound()
```

Independently, make the authorisation read address the same name the writes do — that is the change used as the negative control in §8. Either one closes this report; doing both is defence in depth, and the second also protects any future consumer of `getEnsOwner`.

### 11. Honest limitations

* **One link is contract-side and unexecuted**, because the ENS contracts are out of scope and absent from this repository: that the deployed ENSv2 registrar accepts a label which is not ENSIP-15-normalised. If it rejects them, the attacker cannot create the confusable name and the chain collapses. The target's own in-scope code asserts the opposite in five independent places — `apps/portal/src/utils/token/isNameOrAddress.ts:9-13` (*"the ENSv2 contracts impose no character constraints on labels (only a 1–255 byte length), so the actual client-side rule is ENSIP-15 normalization"*), `features/registry/hooks/useRegistryLabels.ts:20` (*"null when unnormalized"*), the live "Normalized / Not Normalized" badge at `routes/$name/token.tsx:290,352`, `features/reverse-resolution/hooks/useSetL2ReverseName.ts:87` (*"accepts any UTF-8 bytes verbatim, so unnormalized…"*), and `apps/manager/src/features/dashboard/components/NamesTable.tsx:84-85` (*"the display name may be unnormalized (e.g. mixed case) and would hash to the wrong label"*) — but a comment is not bytecode, and I did not touch a chain.
* The PoC exercises the real authorisation function and the real label/token-id derivation. It does **not** drive a wallet through to a mined transaction; the write sinks are established by reading the production call chain in §3.
* No live network, RPC, DNS or third-party service was contacted at any point.
