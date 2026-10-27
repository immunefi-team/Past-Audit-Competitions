# 92912 w a medium any address can replace the rendered history of any ens name

**Submitted on Sep 13th 2026 at 07:27:43 UTC by @nomorecaffeine for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92912
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

### Bug Description

The Explorer builds a name's History tab by asking the v1 subgraph for that name's resolver events:

[fetchV1NameHistory.ts#L101-L103](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L101-L103)

```graphql
        resolvers(where: { domain: $id }, first: $resolvers) {
          events(first: $first, orderBy: blockNumber, orderDirection: $orderDirection) {
```

The file states the premise this rests on at `:40-45`: *"Resolver events are read from every resolver the name has ever pointed at"*.

That premise is false. A `Resolver` row is keyed on the **emitting contract** plus the node carried in the **event payload**, and the subgraph indexes resolver events from every contract rather than from a registry-derived list. So the set returned is *every contract that has ever emitted a resolver event carrying this namehash*, which any address can join. `domain.resolver` is never selected, so nothing between the fetch and the render can compare a row against the registry.

Ordering and truncation then turn one injected row into a replacement of the whole page:

[truncateToTransactions.ts#L31-L35](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/history/truncateToTransactions.ts#L31-L35)

```ts
  const kept: TimelineIndexerEvent[] = []
  for (const group of byTransaction.values()) {
    if (kept.length > 0 && kept.length + group.length > limit) break
    kept.push(...group)
  }
```

Groups are iterated in insertion order, which follows the newest-first sort at `useNameHistoryTimeline.ts:297`. Once the attacker's group fills the 100-event budget, every later group breaks the loop.

An attacker uses this to overwrite what the Explorer says about a name he has no relationship to:

* Alice owns `alice.eth`. Its real history is a registration, a wrap and a resolver setup.
* Mallory wants visitors to believe `alice.eth` resolves to an address he controls.
  * He deploys one contract. It is not a resolver, is never registered, and is never named by `setResolver`.
  * In one transaction, that contract emits 100 `AddrChanged` logs, each carrying `namehash("alice.eth")` and Mallory's address.
* The subgraph creates a `Resolver` row keyed `(chainId, Mallory's contract, namehash("alice.eth"))` and indexes all 100 events under it.
* Bob opens `/alice.eth/history`, a first-class tab in the name's sidebar that needs no wallet:
  * `fetchV1NameHistory` returns Mallory's 100 events alongside Alice's real ones, because the `where` clause matched his contract's row.
  * `adaptV1Events` stamps the viewed name onto each event (`:266`) and writes Alice's namehash into the payload (`:188`), so the detail table shows it as a decoded event parameter.
  * `dropDuplicateEthAddrEvents` (`:219-235`) removes the coin-60 twin, exactly as it would for a genuine `setAddr`.
  * The sort puts Mallory's transaction first, and `truncateToTransactions` keeps it whole and breaks. Alice's real history is rendered zero times.
  * `descriptors.ts:70-81` labels each row `Set address to` beside Mallory's address chip, and `EventRow.tsx:140-147` badges his contract `resolver`.
* Bob reads a complete, chronologically plausible history of `alice.eth` in which no row is Alice's, headed by `"Showing the most recent 100 events."` (`HistoryTimeline.tsx:123-127`).

Mallory never sets a sort key: `adaptV1Events.ts:271` takes the real block timestamp. He only chooses when to transact, and now beats every historical block. Fewer than 100 logs often suffice, since he needs only to leave less room than the real history takes.

### Impact

Any address can permanently author what the Explorer presents as any name's own history, and delete the real history to make room, for the cost of one deployment and one transaction. The victim is never involved and cannot remove the rows, which live on chain. Every later visitor sees the substituted timeline.

The affected population is every name the Explorer renders, not only v1 names. The `where` clause matches on the node string, so no `Domain` entity is required: a node whose `domain(id:)` returns `null` still returns resolver rows with their events, so a name that only ever existed in ENS v2 is injectable.

Nothing on this path reaches a signature. The name's live resolution is read from chain and is unaffected. This is a content-integrity defect.

### Attacker prerequisites

One transaction. No role, no key, no privileged address, no prior registration, no relationship to the victim name.

In the classic ENS subgraph a contract's resolver events are indexed only after a `NewResolver` names that address, which would be a real gate. This index applies no such gate: `newResolvers` filtered to the injecting contract returns nothing across every node and every block, while the identical filter over a genuine resolver returns rows.

### Recommendation

`NewResolver` is emitted only by the registry's `setResolver`, so a name's `NewResolver` events are exactly the resolvers it has ever pointed at, moved-off ones included. Select them in the same request and drop any row not in that set:

[fetchV1NameHistory.ts#L101](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/history/v1/fetchV1NameHistory.ts#L101)

```diff
+        newResolvers(where: { domain: $id }, first: $resolvers) {
+          resolverId
+        }
         resolvers(where: { domain: $id }, first: $resolvers) {
```

Then filter `resolvers[]` by the addresses parsed out of those `resolverId` values, excluding the zero address since `setResolver(node, 0)` is a legal clear. This keeps the moved-off-resolver history the doc comment protects, which `domain.resolver.events` would lose.

## Link to Proof of Concept

<https://gist.github.com/nguyen-xuan-quoc/091f5e33d1f54af17a9995d78903e23b>

## Proof of Concept

### Proof of Concept

Seven tests, run against the live Sepolia subgraph the app's own chain config names. They import `fetchV1NameHistory`, `adaptV1Events` and `truncateToTransactions` rather than reimplementing them, so what is measured is the application's behaviour. No transaction is sent to any network and no shipped file is modified.

<https://gist.github.com/nguyen-xuan-quoc/091f5e33d1f54af17a9995d78903e23b>

The gist also contains the output of the run below, so the results can be compared without executing anything. To run the tests:

1. Clone the audit repo and install:

```sh
git clone https://github.com/immunefi-team/audit-comp-ens
cd audit-comp-ens && git checkout audit-comp-ready && pnpm install --frozen-lockfile
```

2. Copy `poc-v1-history-injection.test.ts` into `apps/portal/src/features/history/`.
3. Run it:

```sh
pnpm --filter portal exec vitest run src/features/history/poc-v1-history-injection.test.ts
```

Two results cannot be checked by reading the code. The first is that the index applies no gate, so a deliberate injection needs nothing but a deploy:

```
[poc] test 7: NewResolver events naming 0x8fade66b79cc9f707ab26799354482eb93a5b7dd (control) = 10+
[poc] test 7: NewResolver events naming 0xffc48055d79b5e4282a5e00b499fd55506c3e53d = 0, on any node, ever
[poc] test 7: so the indexer applies no NewResolver gate and no address allowlist
```

The second is the displacement boundary, measured either side:

```
[poc] test 4: genuine events 9, page size 100
[poc] test 4: 101 events in one tx -> kept 101, genuine survivors 0
[poc] test 4: displacement threshold is 92 events in one tx
[poc] test 4: at 91 events, genuine survivors 9
```
