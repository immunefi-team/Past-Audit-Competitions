# 91314 w a medium explorer builds a user s own name list from an ensjs query whose default relation set includes resolvedaddress so any name an attacker points at a victim s address is listed a&#x20;

**Submitted on Aug 31st 2026 at 14:49:38 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91314
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Explorer has three surfaces that answer "which names does this address have?":

| surface                                | source                                          | membership rule                        |
| -------------------------------------- | ----------------------------------------------- | -------------------------------------- |
| `/addr/$addr/names`, dashboard "Names" | `useV1NamesForAddress` (V1)                     | **ensjs default**                      |
| same, V2 half                          | `useV2NamesWithRolesForAddress`                 | `domains(where: { owner })`            |
| search box, "Names you own"            | `useV1NamesForAddress` + `useV2NamesForAddress` | **ensjs default** / `where: { owner }` |

`useV1NamesForAddress` forwards its parameter object to `@ensdomains/ensjs`'s `getNamesForAddress` verbatim. Every one of its three call sites passes only `{ address }`. ensjs therefore applies its own default relation set — and that set includes `resolvedAddress: true`.

`resolvedAddress` is not an ownership relation. In the ENS V1 subgraph it is written by `handleAddrChanged`: it is the address a name's `addr` record points at. Whoever controls a name sets it, unilaterally, to any address they like. The address on the receiving end never consented, was never notified, and cannot refuse.

So one `setAddr` puts a name the attacker owns into a stranger's name list.

The two surfaces built the same way in the same codebase both get this right, which is what makes the omission a defect rather than a product decision: the V2 reader in the very same directory says so in a comment (`// Only include names the address owns`), and the Manager's V1 reader — querying the *same subgraph* — writes the ownership disjunction out by hand and deliberately leaves `resolvedAddress` out of it.

The row that arrives carries `relation = { owner: false, registrant: false, resolvedAddress: true }`. The names table destructures `relation` two lines from where it would have to be checked, reads only `registrant`/`wrappedOwner`/`owner`, and never looks at `resolvedAddress` — not to filter the row, not to label it. Selection then drops `relation` entirely, so nothing downstream can tell the row apart from a name the victim really holds.

## Root cause

### 1. The reader inherits a library default it never states

`apps/portal/src/features/dashboard/hooks/useV1NamesForAddress.ts:18-31`

```ts
const getV1NamesForAddress = ResultFn(async function* (
  params: GetNamesForAddressParameters,
) {
  const client = yield* safeGetClient()

  const names = yield* await fromPromise(
    ensjs_getNamesForAddress(client, params),   // <- params forwarded verbatim
    ...
  )
  return ok(names)
})
```

All three call sites pass an address and nothing else:

* `apps/portal/src/routes/addr/$addr/names.tsx:95-97` (route loader) and `:172`
* `apps/portal/src/features/dashboard/components/NameList.tsx:113`
* `apps/portal/src/features/dashboard/hooks/useSearchResults.ts:124`

### 2. ensjs's default includes a relation that is not ownership

`@ensdomains/ensjs@e96662c`, `dist/actions/subgraph/getNamesForAddress.js:63-73`

```js
const getNamesForAddress = async (client, { address, filter: _filter, ... }) => {
    const filter = {
        owner: true,
        registrant: true,
        resolvedAddress: true,   // <-
        wrappedOwner: true,
        allowExpired: false,
        ...
        ..._filter,
    };
```

which becomes `where: { or: [{owner}, {registrant}, {resolvedAddress}, {wrappedOwner}] }`. The PoC captures that exact `where` clause off the app's own query rather than asserting it from the source.

### 3. Two readers in the same codebase disagree

`apps/portal/src/features/dashboard/hooks/useV2NamesWithRolesForAddress.ts:57` and `:82`

```graphql
domains(where: { owner: $account }) { ... }
```

```ts
  // Only include names the address owns, attaching role info where available
```

`apps/manager/src/features/migration/service/v1SubgraphClient.ts:55-59` — against the same V1 subgraph:

```ts
  const baseFilters: Record<string, unknown>[] = [
    {
      or: [{ owner: addr }, { registrant: addr }, { wrappedOwner: addr }],
    },
```

### 4. The row is neither filtered nor labelled

`apps/portal/src/routes/addr/$addr/names.tsx:177-191`

```ts
    const v1Names: NameRow[] = (v1NamesQuery.data || []).map(
      ({ name, expiryDate, relation }) => ({
        name,
        expiryDate: expiryDate?.date ?? null,
        roleBitmap: null,
        v1Roles: {
          owner: relation.registrant || relation.wrappedOwner,
          manager: relation.owner || relation.wrappedOwner,
        },
        protocolVersion: 'ENSv1',
      }),
    )
```

`relation.resolvedAddress` appears nowhere in `apps/portal/src`.

### 5. Nothing downstream can recover the distinction

`apps/portal/src/features/renew/utils/nameExtension.ts:11-23` — `getSelectedNames` maps a row to `{ name, isV2, expiryDate }`, dropping `relation`. `isExtendable2LD` (same file, `:61-74`) checks only the `.eth` 2LD shape and the grace window. The last gate, `useRenewableNames` → `ETHRenewerV1.isRenewable`, is a liveness check on the *name*, never a check on the *caller*.

## A note on the renewability gate

`apps/portal/src/features/renew/hooks/useIsRenewable.ts:27-32` (shipped) claims:

> `ETHRenewerV1`, which only renews RESERVED (premigrated) or in-grace names, so a name with no reservation returns `false`.

Measured against the deployed Sepolia `ETHRenewerV1` (`0x4ad56feb…`) on a fork at block 11601093, that is not what it does. `isRenewable` is `true` for every registered V1 name that is not past its 90-day grace window, **including names whose expiry is years away**:

| label       | `nameExpires` | state (now = 1788150336) | `isRenewable` |
| ----------- | ------------- | ------------------------ | ------------- |
| `world`     | 2549940852    | live, \~24 years out     | `true`        |
| `ens`       | 1848126588    | live                     | `true`        |
| `test`      | 1816590960    | live                     | `true`        |
| `alice`     | 1785054756    | in grace                 | `true`        |
| `crypto`    | 1769696772    | past grace               | `false`       |
| `wallet123` | 0             | unregistered             | `false`       |

This matters twice over. It removes the "attacker must let their name lapse into grace" precondition — any ordinary live name works, so the attack is repeatable at will and at scale. And it means a reviewer reading the codebase would have believed the Extend path was far narrower than it is. The comment is corrected as part of the fix.

## Impact

Renewal is permissionless on both ENS protocols, and that is correct: the public `/$name` page offers Extend for any name to any visitor, by design. **This finding is not that anyone can renew a stranger's name.** It is that the Explorer manufactures the belief that a stranger's name is *yours*, in the one place a user has no reason to doubt it, and then offers them the button.

The attacker's cost is one `setAddr` per victim. Their gain is a renewal term paid for by someone else. The victim's loss is the rent, in USDC or DAI, irreversibly. The attacker can point one name at thousands of addresses, or pick a label that resembles a name the victim genuinely holds to raise the click-through rate. Nothing is minted, nothing is transferred, and the victim takes no action outside the normal workflow of reviewing their own names.

Secondary: every surface that counts or lists "the user's names" is inflated by rows the user does not own — including the search dropdown that says *"Names you own"* in those words (`apps/portal/src/features/dashboard/components/SearchResultsList.tsx:289,292,315`).

## Fix

The principle: **membership of "your names" must be a relation the code states, and a row that carries no ownership relation must not be representable on a surface that claims ownership.**

{% stepper %}
{% step %}

## State the relation set, in full

`OWNERSHIP_RELATIONS` is written out with every key — including `resolvedAddress: false` — rather than as a single override, so a future ensjs release cannot widen the set by defaulting some new relation to `true`.
{% endstep %}

{% step %}

## Make the omission unrepresentable

The exported parameter type is no longer ensjs's. It is `Omit<GetNamesForAddressParameters, 'filter'>` plus a `filter` whose ownership keys have been removed, so no call site can re-enable `resolvedAddress` — by intent or by omission — and the compiler says so.
{% endstep %}

{% step %}

## Hold the invariant at the boundary too

`hasOwnershipRelation` drops any row that carries no ownership relation after the query returns. The filter should already have excluded it; this is what keeps the guarantee true if the filter ever stops being honoured (an ensjs change, an indexer that ignores a filter key).
{% endstep %}

{% step %}

## Correct the two comments that misdescribe the renewability gate

`useIsRenewable` and `useCanExtend` both asserted that `ETHRenewerV1` renews only reserved or in-grace names. The table above shows it does not. Both now say what the check actually is — a liveness check on the name, never an authorisation check on the caller — and point at where ownership *is* established.
{% endstep %}
{% endstepper %}

## Proof of Concept

### What it runs against

A **local Anvil fork of Ethereum Sepolia**, pinned at block `11601093`, chain id `11155111`, on `http://127.0.0.1:18546`. **No transaction is ever sent to a live network or a public testnet, and no request is ever made to the real ENS subgraph** — the file throws at import time if the RPC is not `127.0.0.1`/`localhost`, and the one subgraph call is served by a local stub that also asserts what was asked for.

```bash
anvil --fork-url <sepolia-rpc> --fork-block-number 11601093 \
      --port 18546 --chain-id 11155111 --accounts 10 --balance 10000

pnpm install
POC_RPC_URL=http://127.0.0.1:18546 pnpm --filter portal exec vitest run --reporter=verbose \
  src/features/dashboard/poc-v1-names-ownership-relation.poc.test.ts
```

Save the file as `apps/portal/src/features/dashboard/poc-v1-names-ownership-relation.poc.test.ts`. It needs no other new file, no fixture and no configuration change.

### What it deliberately does, and does not, do

* It runs against the **unmodified shipped source** at commit `1c9b47f`.
* It **imports the app's own modules**: the query under test is the app's real `getV1NamesForAddressQueryOptions`, and the downstream gates are the app's real `getSelectedNames` and `isExtendable2LD`. The only seam that is mocked is `@/lib/wagmi/helpers`, so the app's client points at the fork.
* It **does not assert the subgraph filter from the source text.** It runs the app's query, captures the GraphQL body the app actually sends, and reads the `where` clause out of it.
* It **reads the shipped source at runtime** (`readFileSync`) to assert the vulnerable expression and the two correct sibling readers are present verbatim, so no step of the argument rests on a paraphrase.
* It **uses the real deployed contracts** through the fork: `BaseRegistrarImplementation`, the V1 `ENSRegistry`, the V1 `PublicResolver`, `ETHRenewerV1` and the payment token, every address read from `@ensdomains/ensjs/chain` rather than hardcoded.
* It **does not stop at calldata.** The victim's USDC balance, the attacker's on-chain expiry, and the attacker's continued registrant status are all read back off the chain after the renewal.
* Impersonation is used **once**, only to hand Mallory a live V1 name on the fork; every subsequent action is Mallory or the victim acting for themselves.
* The one thing it cannot run is the ENS V1 subgraph itself, which is an out-of-scope third-party asset. That link is covered from both ends instead: test 1 proves the app *asks* for the `resolvedAddress` relation and *returns* a row matched by it, and test 2 performs on-chain the exact `setAddr` that `handleAddrChanged` turns into `Domain.resolvedAddress`.

**Assertion polarity:** every `expect` states the *vulnerable* behaviour. Against the shipped source all four tests **pass** — that is the vulnerability. Against the patched source tests 0 and 1 **fail** — that is the fix. Tests 2 and 3 assert facts about the deployed contracts and about helpers the fix does not touch, and pass either way; they are what establishes the impact.

### Output — against the unmodified shipped source

```
$ POC_RPC_URL=http://127.0.0.1:18546 pnpm --filter portal exec vitest run --reporter=verbose \
    src/features/dashboard/poc-v1-names-ownership-relation.poc.test.ts

 RUN  v4.1.5 /Users/bp/x/ens/1/apps/portal

stdout | 2. on the fork, one `setAddr` makes the victim a live name's resolvedAddress, and renewal takes the victim's money

  world.eth registrant       : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 (Mallory)
  its addr record points at  : 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC (the victim)
  ETHRenewerV1.isRenewable   : true
  victim USDC spent          : 8000021
  expiry 2549940852 -> 2581476852 (+31536000s)

 ✓ 0. the shipped reader passes no ownership filter, while its V2 sibling and the
      Manager both scope by ownership                                          1ms
 ✓ 1. the app's own query asks the subgraph to OR on resolvedAddress, and returns
      a name the address does not own                                          2ms
 ✓ 2. on the fork, one `setAddr` makes the victim a live name's resolvedAddress,
      and renewal takes the victim's money                                     39ms
 ✓ 3. that row survives every gate the names table applies before it offers
      Extend                                                                   0ms

 Test Files  1 passed (1)
      Tests  4 passed (4)
```

`0x70997970…` is Mallory; `0x3C44CdDd…` is the victim. The victim spent 8.000021 USDC. `world.eth` gained a year. Its registrant did not change.
