# 92105 w a insight explorer prints forward match true and primary name for a default reverse record ens resolution reports as a mismatch

**Submitted on Sep 7th 2026 at 07:48:54 UTC by @gzeon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92105
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

Any address can set its own `default.reverse` record to any normalized ENS name, including one it does not own. The record is set, but there is no verified primary name until the named name's forward `addr` record points back, which is why ENSIP-19 resolution reverts with `ReverseAddressMismatch` for such a pair. The Explorer runs that check through ensjs, discards the result, and prints "True" and "Primary name" in the Forward match column of every L2 row that has no reverse record of its own, and of every row whose record read failed. One `setName` transaction makes the address page for an attacker's address claim a verified primary name of `vitalik.eth` on Optimism, Arbitrum, Base, Linea and Scroll. An address on Sepolia has already done exactly that, so the reproduction below needs no setup.

## Vulnerability Details

### The verification result is fetched and discarded

`apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-243`

```ts
  const defaultResult = await getName(l1Client, {
    address,
    coinType: DEFAULT_EVM_COIN_TYPE,
    allowMismatch: true,
  })
  const defaultName = defaultResult?.name ?? null

  return ok(resolvedResults.map((r) => ({ ...r, defaultName })))
```

`getName` returns `{ name, match }`, where `match` is the forward check the UniversalResolver performs for that coin type, and `allowMismatch: true` asks ensjs to return the name even when the check fails. Only `.name` is kept. `defaultName` is then attached to every row (`:243`), carrying no verification state of its own.

Every other record in the same hook keeps its verdict: the L1 rows set `forwardMatch: nameResult?.match` (`:103`) and the L2 rows re-derive it from `getAddressRecord` (`:167-170`).

### The table prints a passing verdict for the inherited name

`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:91-112`, the Forward match column:

```tsx
    cell: ({ row }) => {
      const name = row.original.name
      const forwardMatch = row.original.forwardMatch
      const defaultName = row.original.defaultName

      if (!name && defaultName && canInheritDefault(row.original.coinType)) {
        return (
          <div className="flex flex-row items-center gap-2">
            <Badge variant="outline" className="text-xs">
              <CheckCircle2 className="w-4 h-4" />
              <span>True</span>
            </Badge>
            <Badge variant="outline" className="text-xs">
              <SquareUser className="w-4 h-4" />
              <span>Primary name</span>
            </Badge>
            <Badge variant="outline" className="text-xs">
              Default
            </Badge>
          </div>
        )
      }
```

`forwardMatch` is read on line 93 and never used on this branch. `canInheritDefault` (`ReverseResolutionTable/columns.tsx:12-13`) accepts every row except Default and Mainnet, so the branch is taken by each of the five L2 rows whose own `name` is null (`reverse-resolution/config.ts:79-108`).

`apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:80-83` agrees:

```ts
  const isInheritingDefault = !name && !!defaultName && !isL1

  // Primary name: either matches forward resolution OR inherits default
  const isPrimaryName = forwardMatch || isInheritingDefault
```

That feeds the sidebar's success banner "This is the primary name on {label}" (`ReverseResolutionSidebar.tsx:645-650`) and its `Primary name: True` info row (`:687-693`). A third-party visitor sees the table, whose five inherited rows carry the same `True` verdict (`ReverseResolutionTable.tsx:95-104`).

### A failed record read is rendered as a pass

A row that holds its own reverse record handles a failed forward check correctly: `getAddressRecord` is wrapped in `try`/`catch` that sets `forwardMatch = false` (`useReverseResolution.ts:163-173`), and the cell renders `False` with an `XCircle` (`ReverseResolutionTable/columns.tsx:125-143`). The defect is one step earlier, in the read that decides whether the row has a record at all.

`getReverseResolution` gathers the per-network reads with `Promise.allSettled`, and on a rejected read it logs to the console and substitutes an empty result: `console.error(...)` then `return createEmptyResult(networks[index])` (`useReverseResolution.ts:220-229`). Several earlier failure paths do the same (`:120-141`). An empty result carries no name, which is precisely the condition the Forward match cell treats as "inheriting the default", so the row prints `True`.

The failure need not be arranged. In the run below the Scroll RPC refuses the cross-origin request from a local dev origin, its read throws `ContractFunctionExecutionError: HTTP request failed`, and the Scroll row still renders `True` and `Primary name`. Which network does this depends on the RPC endpoints in use and is not fixed, but any rejected read reaches the same branch. A chain whose record could not be read is presented identically to one that verified the pair, and the only trace is a console line.

### The sibling view models the state this one is missing

The Explorer already models an errored verification, one directory over. Forward resolution declares `unverifiable` as a first-class status, documented as "the L1 verification path itself errored (broken verifier, gateway failure, …) so no verdict is possible" (`forward-resolution/components/hooks/useReverseMatch.ts:54-55`), returns it with the error message on a rejected read (`:289-299`), and renders it as a warning: danger styling (`AddressResolution/AddressResolutionSidebar.tsx:90`), a `TriangleAlert` icon (`:253`) and the label `Unverifiable` (`:261`).

The two views are not mirror images, and the point is narrower than that. It is that no failure in the forward view is allowed to produce a passing verdict: an errored check becomes `Unverifiable`, and a supplementary L2 read that fails degrades to `null` without touching the status (`:210-217`). The reverse table has no equivalent state, so a read that fails there resolves to the branch that prints `True`.

### The same record is reported both ways in one table

The Default row holds the record itself, keeps the real `match` bit, and reports `False`. The five inherited rows report the opposite verdict for the same record and carry a `Default` tag. The PoC prints all seven rows.

### The repo specifies the sidebar's rule, not this column's verdict

`computeDisplayNameState` documents inheritance as intended and a committed test pins it: "Primary name: Either has forwardMatch OR is inheriting default" (`computeDisplayNameState.ts:39`), with a worked example of `defaultName: 'vitalik.eth'`, `forwardMatch: false` yielding `isPrimaryName: true` (`:58-66`), asserted at `computeDisplayNameState.test.ts:90-104`.

It governs a different surface. `computeDisplayNameState` has one consumer, `ReverseResolutionSidebar.tsx`, and it decides `isPrimaryName`. The table every visitor sees does not call it: `ReverseResolutionTable/columns.tsx` inlines its own cell, destructures `forwardMatch` at `ReverseResolutionTable/columns.tsx:93`, and never reads it on the inherit branch at `:96-112`, which emits the `True` badge in the **Forward match** column unconditionally. No rule anywhere in the repo says a forward match may be reported without one being computed, and no test covers that cell for this case.

The documented rule governs `isPrimaryName` on the sidebar. What this report disputes is a column headed "Forward match" asserting a check that never passed.

### Why the record is attacker-controlled

The read goes through ensjs `getName`, which calls the UniversalResolver (`useReverseResolution.ts:236-240`). The write goes to the Sepolia `DefaultReverseRegistrar` the Explorer pins at `reverse-resolution/config.ts:34-35` and uses on its own write path (`AddressResolutionSidebar.tsx:545-553`):

```ts
export const DEFAULT_REVERSE_REGISTRAR_ADDRESS =
  '0x4f382928805ba0e23b30cfb75fc9e848e82dfd47' as Address
```

Its verified Sepolia ABI exposes `setName(string name)`, nonpayable, writing `msg.sender`'s own record, alongside controller and signature-assisted setters that write a supplied address's record. Setting your own reverse record to an arbitrary string is intended. Forward verification is what makes the pair meaningful, and it is the check this view drops. The name must survive ensjs normalization, since `getName` returns `null` for an unnormalized result unless `allowUnnormalized` is set, so the attacker picks any normalized name.

## Impact Details

The Reverse Resolution tab reports, per chain, whether an address's reverse record forward-verifies. Of the seven rows it draws, five answer `True` and label the pair `Primary name` for a record that never verified, while the two that run a chain-specific check report `False`. An eligible L2 row whose record read failed prints the same `True`. An attacker binds a name they do not own to their address in that view for every visitor, persistently, for one `setName` transaction, and nothing is required of the name's real owner.

The program names this shape itself. The bullet's first listed instance is "resolving or rendering an ENS name to an attacker-controlled address", and that is what the table does: it renders `vitalik.eth` against an address the attacker controls, under a label asserting a check was run. The transfer is signed in the user's wallet, so what the application contributes is the rendering the bullet describes. The record behind it is written by one permissionless `setName` call and needs nothing from the victim.

The false binding is confined to this view. The other two primary-name surfaces read different sources: the address overview uses wagmi `useEnsName` (`routes/addr/$addr/index.tsx:39-41`) and `usePrimaryName` keeps the L1 name only when `match` is true (`features/profile/hooks/usePrimaryName.ts:39-43`).

## Distinction from known issues

QA-06 is a deleted or non-resolving subname still shown as primary, which is stale state for a pair that once verified. Here the pair never verified. WEB-1301 concerns owner and expiry metadata on the address view, not the forward-match column.

## References

* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-243`, `:103`, `:167-170`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:12-13`, `:91-112`
* `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:80-83`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionSidebar.tsx:324-326`, `:645-650`, `:687-693`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionTable.tsx:95-104`
* `apps/portal/src/features/reverse-resolution/config.ts:34-35`, `:79-108`
* `apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx:545-553`
* Sepolia `DefaultReverseRegistrar` `0x4f382928805ba0e23b30cfb75fc9e848e82dfd47`, verified ABI includes `setName(string)`
* ENSIP-19 default reverse resolution, ENSIP-3 forward verification

Present from repository import `cda79ac` (`fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86`) through audited commit `1c9b47f`.

## Proof of Concept

## Proof of Concept

Every step is a read. Nothing is written to any chain, and the record used already exists: `0xc9eec1b174a646d7c282820afe94acfba6c00a12` set its `default.reverse` to `vitalik.eth`, a name it does not own and which has no v2 registration that could point back at it.

Requires Node 22, pnpm 10, and `cast` (Foundry) for step 2.

**Step 1. Check out the audited commit and start the Explorer.**

```bash
git clone https://github.com/immunefi-team/audit-comp-ens && cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
pnpm install --frozen-lockfile
pnpm --filter portal dev --port 3001
```

**Step 2. Confirm the reverse record on chain.** Read it straight from the registrar the Explorer pins at `reverse-resolution/config.ts:34-35`:

```bash
cast call 0x4f382928805ba0e23b30cfb75fc9e848e82dfd47 "nameForAddr(address)(string)" \
  0xc9eec1b174a646d7c282820afe94acfba6c00a12 --rpc-url https://ethereum-sepolia-rpc.publicnode.com
```

```
"vitalik.eth"
```

**Step 3. Check the indexer for a registration behind the name.** `vitalik.eth` has no v2 registration on Sepolia. This is supporting evidence rather than proof of the pair's status; the authoritative verdict is the application's own, read in step 6, where the two rows that run the check report `False`:

```bash
curl -s -X POST https://staging-graphql.ens.dev/ -H 'content-type: application/json' \
  -d '{"query":"query { domains(first: 3, where: {name: \"vitalik.eth\"}) { name resolvedAddress { id } } }"}'
```

```
{"data":{"domains":[]}}
```

**Step 4. Open the address page in the running Explorer.**

```
http://127.0.0.1:3001/addr/0xc9eec1b174a646d7c282820afe94acfba6c00a12/reverse-resolution
```

**Step 5. Read the Reverse resolution table.** Paste this in the browser console to extract the rows from the rendered DOM rather than transcribing them:

```js
[...document.querySelectorAll('table tr')].map(r => [...r.querySelectorAll('th,td')].map(c => c.innerText.replace(/\n+/g,' | ').trim()).join('  ||  '))
```

```
[
  "Network  ||  Name  ||  Forward match",
  "Default  ||  vitalik.eth  ||  False",
  "Mainnet  ||  vitalik.eth  ||  False",
  "Optimism  ||  vitalik.eth | Default  ||  True | Primary name | Default",
  "Base  ||  vitalik.eth | Default  ||  True | Primary name | Default",
  "Arbitrum  ||  vitalik.eth | Default  ||  True | Primary name | Default",
  "Linea  ||  vitalik.eth | Default  ||  True | Primary name | Default",
  "Scroll  ||  vitalik.eth | Default  ||  True | Primary name | Default"
]
```

**Step 6. Compare the two answers.** The Default and Mainnet rows run a chain-specific forward check and report `False`, which is correct for this pair. The five L2 rows report `True` in the Forward match column and label the pair `Primary name`, for the same record, in the same table.

**Step 7 (optional, environment-dependent). Observe the same badge on an errored read.** Filter the browser console on `getReverseResolution`. Any network whose read was rejected logs there while its row still prints `True`. On this run the Scroll endpoint refused the cross-origin request and threw `ContractFunctionExecutionError: HTTP request failed`; which endpoint fails depends on the RPCs your environment reaches, so treat the specific network as incidental and the branch as the point. To force it deterministically, block one L2 RPC host and reload.

## Suggested fix

Verify the inherited name against each row's own coin type before treating it as that row's primary. Per ENSIP-19 an L2 row's forward check reads `addr(node, <row coin type>)`, which `getL2ReverseRecord` already does for rows that have their own record (`useReverseResolution.ts:163-170`). Carrying the default read's single `match` bit onto every row would still be wrong, since it answers only the default coin type. Until an inherited row has its own verdict, it should render `False`, or a distinct state that does not claim forward verification.
