# 93056 w a medium migration replays attacker controlled records from a removed cached v1 resolver into the authoritative v2 resolver

**Submitted on Sep 14th 2026 at 04:20:28 UTC by @TREEIndustries for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93056
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Manager migration selects the source V1 resolver from cached subgraph data, reads profile records directly from that address, and writes the returned records into the new authoritative V2 resolver. The cached V1 domain response remains fresh for five minutes, and no authoritative `ENSRegistry.resolver(namehash)` read is performed before profile replay.

If a victim replaces a compromised V1 resolver while Manager still holds the previous resolver address, migration reads the attacker's residual address record from the removed resolver and emits `setAddr(node, 60, attacker)` against the victim's new HCA-owned V2 resolver. Migration therefore makes a record the victim had already removed authoritative again.

This finding does not require stale manager ownership or a V2 role grant. Indexed Registry owner and registrar owner can both equal the victim.

## Root cause

### Manager caches the subgraph-selected resolver

`apps/manager/src/features/migration/service/v1SubgraphClient.ts:22-48` obtains:

```graphql
resolver { address }
```

`apps/manager/src/features/migration/hooks/useV1Names.ts:16-23` treats that response as fresh for five minutes:

```typescript
resultQueryOptions({
  queryKey: qk('migration', 'v1_names', {
	address: address?.toLowerCase(),
  }),
  queryFn:
	enabled && address ? () => getV1NamesForAddress(address) : skipToken,
  staleTime: 5 * 60 * 1000,
})
```

### Manager uses the cached address as the on-chain read target

`apps/manager/src/features/migration/service/buildMigrationPlan.ts:77-86` forwards each classified cached resolver address to the profile reader:

```typescript
return fetchV1Profiles({
  names: namesToOwnedPermRes
	.filter((n) => n.v1ResolverAddress)
	.map((n) => ({
  	nodeHex: namehash(n.domain.name) as Hex,
  	v1ResolverAddress: n.v1ResolverAddress as Address,
	})),
  publicClient,
  profileKeys: preflight.profileKeys,
})
```

`apps/manager/src/features/migration/service/fetchV1Profiles.helpers.ts:62-101` then builds every `text`, `addr`, `contenthash`, and `ABI` read against `name.v1ResolverAddress`:

```typescript
contracts.push({
  address: name.v1ResolverAddress,
  abi: PERMISSIONED_RESOLVER_ABI,
  functionName: 'addr',
  args: [name.nodeHex, coinType],
})
```

No preceding call obtains the authoritative current resolver from the V1 Registry.

### Manager serializes the stale value into signed V2 calldata

The profile is passed to `buildAtomicMigrationBatches()`. Its profile-replay phase wraps record-setting calls into the new PermissionedResolver's `multicall`. For an ETH address record, the nested call is:

```
setAddr(node, 60, staleValue)
```

Gas estimation and transaction simulation do not protect the user: both reading the removed resolver and writing its stored value into the new resolver are valid calls.

## Attack scenario

Actors:

* A: former V1 manager/attacker
* B: current registrar and Registry owner/victim

1. While A legitimately manages `victim.eth`, A stores `addr(node, 60) = A` in a known V1 PublicResolver.
2. B obtains full ownership and opens the Manager migration interface. The V1 response is cached and includes the old PublicResolver address. Owner and registrant both resolve to B, so no stale-manager grant is involved.
3. B replaces the compromised V1 resolver with a safe resolver containing `addr(node, 60) = B`. The old resolver still contains A's per-node record, but it is no longer authoritative because the V1 Registry points to the safe resolver.
4. B continues the normal migration flow before the five-minute cache expires. Ordinary subgraph lag creates the same stale state.
5. Manager reads the ETH record from the cached old resolver rather than querying the Registry for the current resolver. The confirmation dialog only tells B to "Migrate the selected names and restore their records"; it does not enumerate the copied recipient addresses.
6. The old resolver returns A.
7. Manager constructs the atomic migration batch with a profile-replay call that decodes to `setAddr(node, 60, A)` against B's new V2 resolver.
8. B signs and submits the normal migration transaction.
9. After migration, authoritative V2 resolution returns A again.
10. A third party resolving `victim.eth` can send funds to A rather than B.

No project privilege, leaked key, browser exploit, phishing, malformed input, or abnormal Manager action is required. The victim uses the normal migration flow after correcting their V1 resolver.

## Impact

The application reintroduces an attacker-selected ETH address retained from the former owner's prior legitimate authority, even though the victim already removed that resolver from authoritative ENS state. The former owner need not retain ongoing authority over the removed resolver after the victim reclaims the name. The wrong address is embedded in a signed migration transaction and becomes authoritative after migration.

This directly matches the competition's Critical incorrect-address/transaction impact, which specifically includes resolving an ENS name to an attacker-controlled address.

The report does not claim NFT theft, smart-account takeover, or permanent bricking. The victim can repair the record after discovering it, but transfers sent while the attacker address is authoritative are irreversible.

## Proof of Concept

## Proof of concept

Patch: `stale-resolver-replay-poc.patch`

Target file created:

`apps/manager/src/features/migration/service/staleResolverReplay.audit.test.ts`

Apply from the frozen competition repository:

```bash
git apply stale-resolver-replay-poc.patch
pnpm install --frozen-lockfile --ignore-scripts
pnpm --filter manager exec vitest run   src/features/migration/service/staleResolverReplay.audit.test.ts   --reporter=verbose
pnpm --filter manager typecheck
```

The test executes the real classifier, Manager profile fetcher, atomic migration builder, and ABI decoders. It proves:

1. Indexed Registry owner and registrar owner are both the victim, so `managerAddress` is null.
2. The cached domain contains the removed known PublicResolver.
3. The modeled authoritative V1 resolver contains a safe victim address.
4. Manager never calls the authoritative resolver lookup.
5. Manager calls `addr(node, 60)` on the removed resolver and receives the attacker address.
6. The profile-replay call targets the new V2 resolver.
7. The final nested calldata decodes exactly to `setAddr(node, 60, ATTACKER)`.

Verified output:

```
Test Files  1 passed (1)
Tests   	1 passed (1)
Manager typecheck exit code 0
```

A fresh focused migration regression set also passed:

```
Test Files  4 passed (4)
Tests   	54 passed (54)
Manager typecheck exit code 0
```

The Happy DOM/Reown teardown can print an aborted optional remote-config fetch after Vitest completes. It is pre-existing test-environment noise; the command exits 0.

Apply the complementary canonical patch from the `contracts/` directory of a clean contracts-v2 checkout at `97a57293f3b4279d94b571e678edb53ce62638f4` with its submodules installed:

```bash
git apply stale-resolver-production-helper-canonical.patch
forge test --match-contract StaleResolverReplayImpactTest -vv
```

The canonical patch backports the exact HCA-aware helper logic from `0717b0b6f1867f43078f19115dd5dfa4f35426ab`, adjusts the local fixture constructor, and adds the exploit test. `git apply --check` was verified against a clean `97a5729` checkout.

### Canonical contract execution with the production HCA-aware helper

A complementary Foundry PoC executes the canonical ENSv2 contracts and an app-shaped atomic `HCA.executeByOwner` batch. It uses the HCA-aware `MigrationHelper` source from contracts-v2 commit `0717b0b6f1867f43078f19115dd5dfa4f35426ab` (PR #402), which is the helper generation pinned by the frozen Manager.

The frozen app pins production Sepolia helper `0xddC597d937618849348E18Db5D631Ce747bCDeEF`. Read-only verification returned runtime size 5,557 bytes and runtime hash `0x0b8acb00c2912a8b43085e0f55f9459b956cb51be1a16a5ff7ef0346dbd18143`, exactly matching the hash enforced by the app manifest.

The helper resolves the calling factory-certified HCA to its owner before checking approval. The Manager-requested approval of `MigrationHelper` is therefore sufficient: the exact atomic batch migrates the name and then writes the stale profile through the HCA-authorized PermissionedResolver.

Verified output:

```
[PASS] test_closestCanonicalPathReplaysStaleAddressAfterMigration()
[PASS] test_exactAppShapedHcaHelperBatchReplaysStaleProfileWithProductionHelper()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The two PoCs are complementary rather than one literal production fork:

* The Manager test executes the real classifier, profile reader, plan and calldata builders with controlled RPC responses, proving the frozen application dataflow.
* The Foundry test executes canonical ENS contracts and the production-helper-shaped HCA batch, manually mirroring the generated replay calldata and using a faithful factory-certification registry.

## Why the removed resolver can still return the attacker record

Changing `ENSRegistry.records[node].resolver` changes which resolver is authoritative; it does not erase node-keyed storage in the old resolver. Calling that old resolver directly can therefore still return its prior record. The vulnerability is that Manager calls the cached resolver directly instead of resolving the authoritative current resolver through ENS Registry.

## Known-issue and duplicate screening

No matching root cause or consequence appears in the current public known-issue list.

* `WEB-806` says records on a migrated subname are not visible. This finding concerns an unwrapped `.eth` 2LD migration that successfully writes an attacker-controlled record into authoritative V2 calldata.
* `WEB-567` concerns stale avatar/header display after editing. It does not concern resolver selection, migration, transaction construction, or on-chain profile replay.
* `ensdomains/ens-app-v3` PR #1164 / commit `4a742c483aa278950f1f5b1fb1afd2f32b4e5822` is the closest public analogue and should be disclosed. It addresses ENSV1Resolver abstraction/ENSIP-10 behavior in a different legacy application, and its final Universal-Resolver read design would incidentally avoid stale pinned reads in that flow. It does not document or test the competition Manager's five-minute cached resolver pointer changing before V1-to-V2 migration, retained attacker-selected records from the removed resolver, or replay of those values into the new authoritative resolver.
* QA-01 and WEB-1257 concern role combinations and failed actions after transfers, not profile values read from a removed resolver.
* SEC-TXM-002 concerns a cached smart-account address, not a V1 resolver read target.
* The separate stale-manager-role candidate copies cached `domain.owner.id` into `grantRoles`. This finding copies records from cached `domain.resolver.address` into `setAddr`, works when ownership fields are current, and emits no manager-role grant. Fixing only one does not fix the other.

No direct or same-root-cause public duplicate was found. PR #1164 is the closest analogue but addresses resolver abstraction/ENSIP-10 behavior in `ens-app-v3`, not cached-pointer mutation and attacker-record replay in the competition Manager.

## Scope analysis

`apps/manager` is explicitly in scope. The vulnerable source-resolver query/cache, profile read, and V2 profile-replay transaction construction are all in Manager files. The ENS contracts and V1 subgraph are not alleged to be vulnerable; ordinary cache staleness and persistent resolver storage are environmental inputs the application must authenticate before constructing a security-sensitive transaction.

## Why this is a security defect rather than expected record preservation

ENS integration guidance treats the Registry/Universal Resolver result as the authoritative resolver selection. The defect is not that old resolver storage persists; it is that Manager reuses a cached pointer for security-sensitive record migration after the authoritative pointer may have changed.

## Recommended remediation

1. Immediately before any profile read, call authoritative V1 `ENSRegistry.resolver(namehash(name))` at a pinned block.
2. Use only that returned address as the source resolver.
3. If it differs from the previewed/indexed address, invalidate and rebuild the migration plan.
4. Revalidate the resolver and profile values after any intervening wallet approval or deployment prompt and immediately before final submission.
5. Do not reuse the five-minute domain cache for signed transaction construction.
6. Add a regression test in which the indexed resolver is old but the live Registry resolver is safe; no record from the old resolver may appear in the final V2 calldata.
