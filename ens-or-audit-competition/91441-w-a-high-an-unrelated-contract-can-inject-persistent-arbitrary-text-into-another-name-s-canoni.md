# 91441 w a high an unrelated contract can inject persistent arbitrary text into another name s canonical portal history

**Submitted on Sep 1st 2026 at 14:20:44 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91441
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
    * HTML injection without JavaScript
    * Replacing existing text with arbitrary text
    * Arbitrary file uploads, etc.

## Description

## Summary

The Portal's V1 history fetcher asks the configured ENS data service for every `Resolver` row associated with the requested namehash:

```graphql
resolvers(where: { domain: $id }, first: 100) {
  events(...) {
    resolverId
    ... on TextChanged { key value }
  }
}
```

The fetcher's own comment describes these rows as resolvers that the name "has ever pointed at." That assumption is false for the data source it consumes. The corresponding published ENSNode `0.36.0` release deliberately listens for resolver event signatures without binding them to one contract address. Its handler documents that a `Resolver` may be **any** contract emitting a relevant signature, including a contract not intended to act as an ENS resolver. For every matching event, it derives the row from:

```
event emitter address + attacker-supplied node
```

and upserts that row without checking whether the ENS Registry ever assigned the emitter to the node. This is the documented ENSNode/subgraph-compatible matching-log behavior.

The in-scope Portal fails to authenticate the resulting row before presenting it as canonical name history. It:

1. accepts every row linked by `domain = victimNamehash`;
2. extracts the apparent resolver address directly from `resolverId`;
3. places the attacker's `key` and `value` into a canonical `TextChanged` timeline event;
4. makes the unrelated emitter the event's `contractAddress`;
5. summarizes the event as `Set text record key → value`; and
6. labels the emitter `resolver` in the expanded detail row.

No later stage checks Registry `NewResolver` events, current resolver state, historical resolver assignment intervals, ownership, manager authority, or the emitter's relation to the name.

I reproduced the full E2E path with the frozen Portal source, a disposable local chain, and the published ENSNode `0.36.0` image pinned by digest. The run uses an EOA-owned forked V1 name with an ordinary legitimate resolver, proves the independent attacker is neither Registry owner nor approved operator, proves that the attacker cannot call `setText` on the legitimate resolver, deploys an unrelated emitter, emits exactly one harmless proof log, waits for real indexing, and opens the normal Portal history route in Chromium. Fresh-browser and reload checks demonstrate that the injected content is persistent service-backed history rather than reflected input or browser state. Direct on-chain reads prove the Registry resolver and the legitimate resolver's `text(node,"com.twitter")` value are unchanged. A complete node-filtered `NewResolver(node,*)` scan of the current Registry contract, from its deployment block through deterministic chain head `11565324`, proves that the emitter was never assigned by that Registry.

## Finding Description

### Intended behavior

The public history page for a name should distinguish canonical ENS history from arbitrary logs that merely reuse an ENS event signature.

For resolver history, a log is canonical only if its actual emitting contract was the Registry-assigned resolver for the exact node at the log's block and log position. Historical resolvers must remain visible during intervals when they were assigned, but events emitted:

* before assignment;
* after replacement or clearing;
* by an address that was never assigned; or
* by an unrelated contract that copied the event signature

must not be summarized as actions on the name.

At minimum, an unauthenticated matching log must be separated and clearly marked as unverified. It must not appear as `Set text record` in the canonical timeline, and its emitter must not receive a `resolver` label.

### Actual behavior

#### 1. The Portal asks for every resolver row associated with the node

[`apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L13-L19`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L13-L19) allows up to 100 resolver rows for one name:

```ts
const RESOLVERS_PER_NAME = 100
```

The same file describes the selected rows as every resolver the name has ever pointed at:

```ts
Resolver events are read from every resolver the name has ever pointed at
(`resolvers(where: { domain })`), not just `domain.resolver`...
```

[`fetchV1NameHistory.ts#L101-L119`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L101-L119) requests all domain-associated rows and their typed event fields:

```graphql
resolvers(where: { domain: $id }, first: $resolvers) {
  events(first: $first, orderBy: blockNumber, orderDirection: $orderDirection) {
    id
    blockNumber
    transactionID
    type: __typename
    resolverId
    ... on TextChanged { key value }
  }
}
```

The query does not request an assignment block, registry proof, assignment interval, or any evidence that the emitter was ever a resolver for the node.

[`fetchV1NameHistory.ts#L125-L133`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L125-L133) then flattens every returned resolver event into the same list as Registry and registrar history:

```ts
return [
  ...(domain?.events ?? []),
  ...(domain?.registration?.events ?? []),
  ...(resolvers ?? []).flatMap(({ events }) => events),
]
```

#### 2. The configured backend deliberately indexes matching events from arbitrary contracts

The frozen ENSjs configuration points the Portal to `https://v1-graphql.ens.dev/subgraph`. On 2026-08-27, I made read-only metadata requests to that configured service. The response was HTTP 200, identified `x-ensnode-version: 0.36.0`, reported `_meta.deployment: 0.36.0-b7e5262258`, and identified the `subgraph` plugin, Sepolia chain ID `11155111`, and schema `schema_sepolia_v6`. I sent no exploit log or automated attack traffic to that service or public Sepolia.

The local PoC uses the published ENSNode `v0.36.0` source at `528d8902960130aa980b0795ab36b691b72c1cf1` and pins the release image by index digest `sha256:1f10c952a13956c8f6e80502683234cc5876047a811a90ef32875aadaceb4001`. The service's runtime build identifier and the OCI digest are distinct identifiers; the locally pinned release supplies the executable dependency used for the end-to-end proof.

The matching-log behavior is expected data-source behavior rather than an indexer compromise.

At ENSNode `v0.36.0`, [`packages/datasources/src/sepolia.ts#L64-L68`](https://github.com/namehash/ensnode/blob/528d8902960130aa980b0795ab36b691b72c1cf1/packages/datasources/src/sepolia.ts#L64-L68) configures the ENS root `Resolver` datasource without an address:

```ts
Resolver: {
  abi: ResolverABI,
  filter: ResolverFilter,
  startBlock: 3702721,
},
```

[`packages/datasources/src/lib/resolver.ts#L18-L31`](https://github.com/namehash/ensnode/blob/528d8902960130aa980b0795ab36b691b72c1cf1/packages/datasources/src/lib/resolver.ts#L18-L31) explicitly includes both legacy and four-argument `TextChanged` signatures in that wildcard filter:

```ts
{
  event: 'TextChanged(bytes32 indexed node, string indexed indexedKey, string key, string value)',
  args: {},
}
```

The exact release's shared handler explains the semantics explicitly at [`Resolver.ts#L13-L20`](https://github.com/namehash/ensnode/blob/528d8902960130aa980b0795ab36b691b72c1cf1/apps/ensindexer/src/plugins/subgraph/shared-handlers/Resolver.ts#L13-L20):

```ts
NOTE: The indexing logic in this file must use upserts because a 'Resolver'
can be any contract that emits events with the relevant signatures. The
contract may not necessarily be intended for use with ENS as a Resolver.
```

The four-argument handler at [`Resolver.ts#L164-L209`](https://github.com/namehash/ensnode/blob/528d8902960130aa980b0795ab36b691b72c1cf1/apps/ensindexer/src/plugins/subgraph/shared-handlers/Resolver.ts#L164-L209) takes `node`, `key`, and `value` from the log and constructs the resolver identity from the actual emitter:

```ts
const { node, key, value } = event.args
const id = makeResolverId(context.chain.id, event.log.address, node)
const resolver = await upsertResolver(context, {
  id,
  domainId: node,
  address: event.log.address,
})
...
await context.db.insert(schema.subgraph_textChanged).values({
  ...sharedEventValues(context.chain.id, event),
  resolverId: id,
  key: sanitizedKey,
  value: sanitizedValue,
})
```

There is no Registry query or canonical resolver check because the data model intentionally preserves subgraph-compatible matching-log behavior.

Therefore an attacker-controlled, unrelated contract can create the following row with one log:

```
Resolver.domain   = victim node
Resolver.address  = attacker emitter
TextChanged.key   = attacker text
TextChanged.value = attacker text
```

The attacker need not own, manage, wrap, transfer, or resolve the target name.

#### 3. The first-party adapter derives apparent authority from the unverified row ID

[`apps/portal/src/features/history/v1/adaptV1Events.ts#L93-L101`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/adaptV1Events.ts#L93-L101) extracts any address-shaped segment from `resolverId`:

```ts
const resolverAddress = (id?: string): string | undefined =>
  id?.split('-').find((part) => isAddress(part, { strict: false }))
```

This proves only which contract emitted the matching log. It does not prove that the contract was assigned to the node.

#### 4. Attacker text is promoted to a canonical typed event

[`adaptV1Events.ts#L200-L207`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/adaptV1Events.ts#L200-L207) copies the row directly into the canonical `TextChanged` shape:

```ts
.with('TextChanged', () => {
  const key = str(event.key)
  const value = str(event.value)
  return {
    key,
    value,
    asTextChanged: { key, value, resolver, namehash: node },
  }
})
```

[`adaptV1Events.ts#L237-L277`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/adaptV1Events.ts#L237-L277) sets:

```ts
contractAddress: resolver
```

for a resolver-signature event, and spreads the attacker's payload into the same event object used for legitimate V1 history.

The adapter's comment says that resolver events "name their own resolver." In this data contract, they name only their emitter. The missing check is the security boundary.

#### 5. The merged history path performs no provenance check

[`apps/portal/src/features/history/hooks/useNameHistoryTimeline.ts#L236-L300`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/hooks/useNameHistoryTimeline.ts#L236-L300) fetches V1 and V2 events, enriches timestamps, adapts the full V1 vector, merges it, sorts it, and truncates it.

It never compares the resolver emitter with:

* the current Registry resolver;
* a Registry `NewResolver` event;
* a historical assignment interval;
* the name owner or manager;
* an approved resolver list; or
* an on-chain result.

Grouping, sorting, and truncation alter presentation only. They do not authenticate the event.

#### 6. The collapsed timeline renders the arbitrary key and value

[`apps/portal/src/features/history/summarize/descriptors.ts#L83-L93`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/summarize/descriptors.ts#L83-L93) turns a single `TextChanged` event into:

```ts
const key = primary.asTextChanged?.key ?? primary.key ?? '—'
const value = primary.asTextChanged?.value ?? primary.value ?? ''
const slots = [{ kind: 'text', value: key }]
if (value) {
  slots.push({ kind: 'glyph', value: '→' }, { kind: 'text', value })
}
return { label: 'Set text record', slots }
```

[`apps/portal/src/features/history/components/ActionSummaryRow.tsx#L41-L50`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/components/ActionSummaryRow.tsx#L41-L50) mounts the label and slots in the collapsed public timeline. The victim does not need to expand a detail panel before the attacker-selected content appears.

React escapes the strings, so the rendered content remains persistent static plain text without JavaScript execution.

#### 7. The expanded row labels the unrelated emitter `resolver`

[`apps/portal/src/features/history/components/EventRow.tsx#L29-L40`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/components/EventRow.tsx#L29-L40) classifies `TextChanged` as a resolver event.

[`EventRow.tsx#L128-L150`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/components/EventRow.tsx#L128-L150) supplies the unverified emitter to `ContractBadge` with this fallback:

```tsx
<ContractBadge
  address={contractAddress}
  label={RESOLVER_EVENT_TYPES.has(event.type) ? 'resolver' : undefined}
/>
```

Thus the detail view reinforces rather than discloses the provenance error.

#### 8. Persistence does not depend on browser state

The log is mined on chain, indexed into a resolver/event row, and returned on each new GraphQL request. The Portal query has no URL parameter carrying `key` or `value`, no local-storage dependency, and no user-specific state.

The E2E proof verifies all of the following after the attack:

* the forged row is returned by a fresh GraphQL request;
* a hard reload still displays the same key/value;
* a fresh Chromium context still displays the same key/value;
* current Registry resolver state is unchanged;
* the attacker emitter is absent from the complete node-filtered `NewResolver` history of the current Registry contract.

The content remains until the immutable historical event is removed by a chain reorganization or displaced beyond the application's history window by later events. It is stored/persistent, not reflected.

## Root Cause

The in-scope Portal confuses two different predicates:

```
observed:   contract E emitted a resolver-shaped event containing node N
required:   Registry assigned E as resolver for N when that event was emitted
```

It treats the first as proof of the second.

The vulnerable first-party composition is:

```
expected wildcard matching-log dataset
  -> resolvers(where: { domain: victimNode })
  -> resolverAddress(resolverId)
  -> canonical TimelineIndexerEvent
  -> Set text record key → value
  -> emitter badge: resolver
```

No canonical resolver-assignment evidence is requested, represented in the adapted event, or validated before rendering.

## Impact Explanation

The exact High row covers persistent static-content modification without JavaScript, including replacing existing text with arbitrary text.

This finding meets every element:

* **unauthorized:** the attacker is not the Registry owner, is not an approved Registry operator for that owner, is not the current resolver, and cannot call `setText` on the legitimate resolver;
* **arbitrary:** the attacker selects both the visible key and visible value;
* **target application:** the frozen first-party Portal itself creates and renders the canonical history action;
* **persistent:** the content is backed by an indexed immutable log and survives reload plus a fresh browser context;
* **without JavaScript:** the payload is harmless plain text and React escaping remains active; and
* **another user's object:** the attacker supplies the victim namehash even though the current Registry's complete node-filtered history never assigns the emitter.

The history is presented as actions taken on that ENS name, not as an untrusted global log search. The summary says `Set text record`, and the detail calls the attacker emitter a `resolver`. This is a material application-content integrity failure.

## Likelihood Explanation

The exploit requires only ordinary untrusted capabilities:

1. know or calculate the public namehash of a target ENS name;
2. deploy or call an attacker-owned contract with the standard event declaration; and
3. emit one short event.

The attacker does not need the victim to connect a wallet, click a link, approve a transaction, choose an unsafe resolver, alter configuration, or cooperate. The attack does not depend on admin, governance, oracle, Registry, indexer-operator, browser-extension, or third-party credential compromise.

Wildcard event matching, upsertion, and Portal's canonical-history conversion are deterministic throughout the reproduced stack.

The only ordinary delay is indexer finality/ingestion. Once indexed, every visitor to the normal history route receives the same stored row.

## Attack Path / Reproduction Path

{% stepper %}
{% step %}

### Identify the victim name and legitimate resolver

Victim `V` controls an ordinary V1 ENS name `N`.

Registry state identifies legitimate resolver `R`; the complete node-filtered `NewResolver(N, *)` history of the current Registry contract contains no assignment of attacker emitter `E`.
{% endstep %}

{% step %}

### Establish the attacker's lack of authority

Independent attacker `A` is neither Registry owner nor approved Registry operator for `N`.
{% endstep %}

{% step %}

### Deploy an unrelated emitter

`A` deploys unrelated contract `E`, which merely declares the standard four-argument `TextChanged` event.
{% endstep %}

{% step %}

### Emit the forged resolver-shaped event

`A` calls `E.emitText(N, indexedKey, key, value)` once with short harmless proof strings.

The log's actual emitter is `E`, while its indexed `node` argument is the victim's public namehash `N`.
{% endstep %}

{% step %}

### ENSNode indexes the matching event

ENSNode's wildcard resolver datasource receives the matching signature.

ENSNode upserts resolver row `(chainId, E, N)` and records the attacker-selected key/value, as its documented subgraph-compatible behavior requires.
{% endstep %}

{% step %}

### Portal retrieves the unverified resolver row

A fresh Portal request for `/$name/history` invokes `fetchV1NameHistory`.

Portal queries every resolver row linked to `N`, including `(E, N)`.
{% endstep %}

{% step %}

### Portal adapts the event as canonical history

Portal extracts `E` from `resolverId`, adapts the event as canonical V1 history, and assigns `E` as `contractAddress`.
{% endstep %}

{% step %}

### The Portal renders attacker-controlled text

The collapsed timeline persistently renders `Set text record <key> → <value>`.

Expanding the action shows `E` with the label `resolver`.
{% endstep %}

{% step %}

### Verify canonical state remains unchanged

Direct Registry controls still report `R`, the legitimate resolver returns exactly the same `text(N, key)` value before and after the attack, and the complete current-Registry assignment scan proves `E` was never assigned by that Registry.
{% endstep %}

{% step %}

### Verify persistence

Reloading and opening a fresh browser context reproduce the injected history text.
{% endstep %}
{% endstepper %}

## External Preconditions

* The target name has a V1 domain row reachable by the Portal history route.
* The backend used for V1 history preserves the documented ENSNode/subgraph-compatible wildcard resolver-event semantics.
* The attacker can deploy or reuse an emitter and submit one ordinary event-emitting transaction to the relevant chain. The canonical PoC deploys and emits in two transactions.
* The event remains inside Portal's returned history window. One recent event is sufficient for an ordinary name with fewer than 100 resolver rows/events.

## Internal Preconditions

* `fetchV1NameHistory` continues to query `resolvers(where: { domain: $id })` without canonical assignment evidence.
* `adaptV1Events` continues to derive resolver identity from `resolverId` and promote the payload into `asTextChanged`.
* no downstream component verifies the emitter against Registry assignment intervals; and
* the summary and detail components continue to render the event as canonical history and label the emitter `resolver`.

## Affected Code

All first-party links below reference frozen competition commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

* [`apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L13-L19`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L13-L19) — defines the unqualified 100-row resolver-history set.
* [`apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L40-L45`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L40-L45) — documents the incorrect assumption that every selected row is a resolver the name once pointed at.
* [`apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L65-L123`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L65-L123) — requests every domain-associated resolver row and attacker-controlled `TextChanged` fields without assignment provenance.
* [`apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L125-L133`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L125-L133) — merges unverified resolver events with canonical Registry and registrar history.
* [`apps/portal/src/features/history/v1/adaptV1Events.ts#L93-L101`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/adaptV1Events.ts#L93-L101) — treats an address parsed from the row ID as resolver identity.
* [`apps/portal/src/features/history/v1/adaptV1Events.ts#L121-L136`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/adaptV1Events.ts#L121-L136) — copies unverified fields into decoded event data.
* [`apps/portal/src/features/history/v1/adaptV1Events.ts#L183-L207`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/adaptV1Events.ts#L183-L207) — maps attacker-selected text and the unverified emitter into the canonical typed payload.
* [`apps/portal/src/features/history/v1/adaptV1Events.ts#L237-L277`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/v1/adaptV1Events.ts#L237-L277) — marks the emitter as `contractAddress` and produces a normal V1 timeline event.
* [`apps/portal/src/features/history/hooks/useNameHistoryTimeline.ts#L236-L300`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/hooks/useNameHistoryTimeline.ts#L236-L300) — merges and returns the event without a Registry provenance check.
* [`apps/portal/src/features/history/summarize/descriptors.ts#L83-L93`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/summarize/descriptors.ts#L83-L93) — renders arbitrary key/value in the collapsed `Set text record` action.
* [`apps/portal/src/features/history/components/ActionSummaryRow.tsx#L41-L50`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/components/ActionSummaryRow.tsx#L41-L50) — displays the attacker-selected slots without requiring disclosure expansion.
* [`apps/portal/src/features/history/components/EventRow.tsx#L29-L40`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/components/EventRow.tsx#L29-L40) and [`#L128-L150`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/history/components/EventRow.tsx#L128-L150) — classify the forged event as a resolver event and label the unrelated emitter `resolver`.
* [`apps/portal/src/routes/$name/history.tsx#L7-L19`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/routes/%24name/history.tsx#L7-L19) — exposes the vulnerable public history route.

## Recommended Mitigation

Do not treat a resolver-shaped log as canonical name history merely because its payload contains the requested node. Before adapting or rendering a resolver event, bind the log's actual emitter to the Registry's resolver assignment for that node at the event's exact block and log position. Preserve historical resolver events only inside a proven assignment interval, and reject events from emitters that were never assigned or that emitted outside their interval.

If the data service cannot provide assignment provenance in the same response, keep matching-signature rows out of the canonical timeline or place them in a separate, explicitly unverified view. Do not label an emitter `resolver` until that provenance check succeeds. Add regression coverage for events emitted before assignment, after replacement or clearing, and by a never-assigned contract using the victim's node.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/0a8d400311e19f229d12200a533508a5/8bb892b568b2e5a9d2abb935a9b3e2b73679c0fc>

## Proof of Concept

### Description

This full E2E PoC runs the frozen Portal, a fixed Sepolia fork, the published digest-pinned ENSNode `0.36.0` indexer, PostgreSQL, a byte-preserving local TLS bridge, and real Chromium.

During setup, the harness verifies that `purple.tomato.eth` is owned by an EOA and points to legitimate resolver `0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5`. Solely to materialize that already-forked canonical baseline inside the short local indexing window, it impersonates the existing victim EOA on the disposable fork and submits idempotent `setOwner(node, sameOwner)` and `setResolver(node, sameResolver)` transactions. Before-and-after reads prove that neither value nor any authority changes. These two local setup transactions are not attacker actions or exploit preconditions. A separate attacker account cannot call `setText` on the legitimate resolver, is not a Registry operator, and deploys an unrelated contract that implements no resolver functions.

The exploit is one real transaction from that attacker contract emitting `TextChanged(namehash("purple.tomato.eth"), "com.twitter", "FORGED_WITHOUT_RESOLVER_AUTHORITY")`. Real ENSNode indexes the unrelated emitter as a resolver row. The unmodified Portal consumes its normal V1 history query and persistently renders the attacker's key and value as `Set text record`, while the expanded row labels the unrelated emitter `resolver`.

The controls scan all `NewResolver(node,*)` events from Registry deployment through deterministic head `11565324` and prove that the emitter was never assigned. They also prove that the Registry resolver and legitimate resolver text remain unchanged, the forged content survives reload and a fresh browser context, and sibling `pink.tomato.eth` does not receive it. No target source, GraphQL response, indexer handler, Portal component, receipt, or browser DOM is mocked or replaced. The runner rechecks target hashes after browser execution and its exit trap removes only its own processes, containers, network, and volume.

### Where to place files and their names

Place the fifteen-file Gist package under the frozen repository root as follows:

```
poc/portal-unassigned-resolver-history-injection/
├── README.md
├── SOURCE_SHA256SUMS
├── TARGET_SHA256SUMS
├── ENSNODE_SOURCE_SHA256SUMS
├── browser.mjs
├── consolidate.mjs
├── docker-compose.poc.yml
├── finalize.mjs
├── foundry.toml
├── https-proxy.mjs
├── image-provenance.mjs
├── query-indexer.mts
├── run.sh
├── setup.mts
└── src/
    └── ArbitraryTextEmitter.sol
```

The clean successful run used macOS 26.5.2 arm64, Node.js 22.23.2, pnpm 10.27.0, Foundry 1.7.1, Playwright 1.58.2, Chrome for Testing 145.0.7632.6, Docker with Compose, OpenSSL, `curl`, Git, and `shasum`. `SOURCE_SHA256SUMS` itself has SHA-256 `586b19afa0a09a06b8d203ccc1d2373b998d657a500f540691c1dbc3cad68f20`.

### Command to run

Run these commands from a clean directory. The Gist stores Solidity at its root, so the commands reconstruct the single required `src/` directory before verification and execution:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/0a8d400311e19f229d12200a533508a5.git \
  poc/portal-unassigned-resolver-history-injection
git -C poc/portal-unassigned-resolver-history-injection checkout --detach \
  8bb892b568b2e5a9d2abb935a9b3e2b73679c0fc
mkdir -p poc/portal-unassigned-resolver-history-injection/src
mv poc/portal-unassigned-resolver-history-injection/ArbitraryTextEmitter.sol \
  poc/portal-unassigned-resolver-history-injection/src/ArbitraryTextEmitter.sol

node --version
corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
pnpm --dir e2e exec playwright install chromium

POC_LOG_DIR="$(mktemp -d)" \
bash poc/portal-unassigned-resolver-history-injection/run.sh
```

`node --version` must report Node.js 22. The runner verifies the target commit, all PoC and affected-source hashes, the fork block hash, ENSNode source hashes, and the three Docker image digests before executing the attack.

### Logs proving the bug

The fresh clean-clone run produced:

```
ENS28_INDEXER_PROVENANCE_PASS version=0.36.0 revision=528d8902960130aa980b0795ab36b691b72c1cf1
ENS28_FORK_ANCHOR_PASS block=11565219 hash=0xda564596b0ab93b11a215fe3113ec9c925b3b5a49a63be4349f2b26cdae4ee61
ENS28_EOA_OWNER_PASS name=purple.tomato.eth owner=0xc1e1d3Bc99fb4B6529Ba7BE62E628FCD74cD8d8f
ENS28_SENDER_SEPARATION_PASS attacker=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 victim=0xc1e1d3Bc99fb4B6529Ba7BE62E628FCD74cD8d8f
ENS28_REGISTRY_ASSIGNMENT_PASS current=0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5 unassignedEmitter=0x1bf32a79CbA9198eFE2bcaDA987E80c940Cb3FeF
ENS28_COMPLETE_ASSIGNMENT_HISTORY_PASS count=2 canonical=1 local=1
ENS28_ATTACK_LOG_DECODE_PASS emitter=0x1bf32a79CbA9198eFE2bcaDA987E80c940Cb3FeF node=0x4e5a8813e37794fd69f5095b86ea3583e68aee30c56979b7e40f3d7e65a611cb key=com.twitter logIndex=0
ENS28_ATTACKER_AUTHORITY_NEGATIVE_PASS registryOperator=false resolverWrite=false
ENS28_SETUP_PASS emitter=0x1bf32a79CbA9198eFE2bcaDA987E80c940Cb3FeF attackTx=0xe2ebfa4c22aefa4cd2b16dbe2e653053625c94fd2e5becffc49faea92587abd1
ENS28_INDEXER_QUERY_PASS resolverId=0x1bf32a79cba9198efe2bcada987e80c940cb3fef-0x4e5a8813e37794fd69f5095b86ea3583e68aee30c56979b7e40f3d7e65a611cb event=11565223-0
ENS28_INDEXER_UNASSIGNED_EMITTER_PASS registry=0xE99638b40E4Fff0129D56f03b55b6bbC4BBE49b5 emitter=0x1bf32a79CbA9198eFE2bcaDA987E80c940Cb3FeF
ENS28_SIBLING_INDEXER_NEGATIVE_CONTROL_PASS
ENS28_COLLAPSED_UI_PASS
ENS28_EXPANDED_UI_PASS
ENS28_RELOAD_PERSISTENCE_PASS
ENS28_FRESH_CONTEXT_PERSISTENCE_PASS
ENS28_SIBLING_NEGATIVE_CONTROL_PASS
ENS28_V2_UNRELATED_READ_ONLY_PASS responses=4
ENS28_BROWSER_PASS v1Continuations=4
ENS28_CROSS_LAYER_EVIDENCE_PASS receiptSha256=ee2bdbd1ed9370e0c8b34e999371d347b02848f8adf656521760b0cb67736ab7
ENS28_POST_TARGET_INTEGRITY_PASS
ENS28_FINAL_RECEIPT_WRITTEN sha256=e5621653852b3869a8d5bad46ced7c8fc37accba8fd9795a8f6a3d2071a2c55b
ENS28_FINAL_RECEIPT_SHA256 e5621653852b3869a8d5bad46ced7c8fc37accba8fd9795a8f6a3d2071a2c55b
ENS28_EVIDENCE_MANIFEST_SHA256 1d22f22e32c685d973624c30883a9257b562ebfd38cfd76b3b78215b4cce541a
ENS28_FINAL_PASS
ENS28_CLEANUP_PASS containers=removed volumes=removed processes=stopped
```

The generated `final-receipt.json` binds attack transaction `0xe2ebfa4c22aefa4cd2b16dbe2e653053625c94fd2e5becffc49faea92587abd1`, emitter `0x1bf32a79CbA9198eFE2bcaDA987E80c940Cb3FeF`, event `11565223-0`, unchanged legitimate resolver/text state, and all browser assertions. The evidence directory also contains `state.json`, `indexer-query.json`, `proxy-trace.jsonl`, `browser-receipt.json`, the reload/fresh-context/sibling screenshots, `cross-layer-receipt.json`, `post-target-integrity.log`, and `EVIDENCE_SHA256SUMS`.

**Full source:** [**immutable fifteen-file PoC Gist**](https://gist.github.com/aydarkhusaenov/0a8d400311e19f229d12200a533508a5/8bb892b568b2e5a9d2abb935a9b3e2b73679c0fc)
