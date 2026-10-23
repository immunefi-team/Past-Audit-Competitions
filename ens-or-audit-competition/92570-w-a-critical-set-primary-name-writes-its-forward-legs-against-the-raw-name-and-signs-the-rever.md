# 92570 w a critical set primary name writes its forward legs against the raw name and signs the reverse leg against the normalized name so one click points the victim s reverse record at a name t&#x20;

**Submitted on Sep 10th 2026 at 17:47:22 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92570
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

Choosing a primary name in the Manager dashboard produces a sequence of transactions. The name the user clicked is carried through some of them raw and through the last one normalized. When those two forms differ, one user action writes address and resolver records on one name and claims a *different* name as the wallet's primary.

The claimed name is the ENSIP-15 normalized form, which the victim may not own. If an attacker owns it, the victim's wallet reverse-resolves to the attacker's name.

## Where the legs diverge

The selected name is the **raw** indexer field. `apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx:81`:

```ts
const label = resolveDomainLabel(domain)
```

and `apps/manager/src/features/dashboard/utils.ts:42-46`:

```ts
export const resolveDomainLabel = (domain: {...}): string =>
  domain.name ?? domain.normalizedName ?? domain.id
```

That value is rendered in the row (`:116`), used for the selection state (`:82`) and passed out by `onSelectName(label)` (`:96`) to become `selectedName`.

**Forward legs — raw.** `selectedName` goes straight into the record write and the resolver setup:

* `:229` — `saveRecords({ name: selectedName, … })`
* `:278` — `setupControlledResolver({ name: selectedName, … })`

**Reverse leg — normalized.** `:469` calls `submitPrimaryName({ name: selectedName, … })`, which reaches `apps/manager/src/features/profile/service/setPrimaryName.ts:48-49`:

```ts
// Normalized claim string: a non-canonical name would fail the bidirectional
// check at resolution time and read as "no primary name".
const withEthSuffix = (name: string) =>
  normalize(name.endsWith('.eth') ? name : `${name}.eth`)
```

consumed at `:140` (`const cleanName = withEthSuffix(name)`) and encoded at `:163` (`args: [ownerAddress, cleanName]`). The transaction description at `:218` also uses `cleanName`.

So the same click writes `setAddr` on `namehash(raw)` and `setName(normalize(raw))`.

## The comment shows this is a known distinction, applied to only one leg

The comment above `withEthSuffix` states the reason for normalizing: a non-canonical name "would fail the bidirectional check at resolution time and read as 'no primary name'." The authors knew non-canonical names reach this code and normalized the reverse leg deliberately. The forward legs, in the same dialog, were left raw.

The result is that the normalization intended to make the claim resolve correctly is exactly what makes it claim a different name.

## Attack path

1. The attacker owns `treasury.eth`.
2. The attacker registers the literal label `Treasury` at the registrar with `owner` set to the victim's address. This is a distinct registration — `keccak256("Treasury")` is not `keccak256("treasury")` — and the victim genuinely owns it.
3. The indexer records the row as `{ name: "Treasury.eth", normalizedName: "treasury.eth" }`. The repository's own fixtures model exactly this shape: `apps/manager/src/features/dashboard/v2NameRoles.test.ts:43` uses `makeDomain({ name: 'Alaska.eth', normalizedName: 'alaska.eth' })`, and `utils.test.ts:193-194` uses `{ name: 'Display Name', normalizedName: 'normalized' }`.
4. The victim opens the primary-name dialog. The row renders as `Treasury.eth`, a name they own.
5. The victim selects it and confirms.
6. The forward legs write the address record on `namehash("Treasury.eth")`. The reverse leg signs `setName("treasury.eth")` — the attacker's name.

Resulting state: the victim's reverse record names `treasury.eth`. If the attacker points that name's ETH address record at the victim's wallet, the ENSIP-19 bidirectional check passes, and every ENS-aware interface renders the victim's address as the attacker's name.

## Impact

The application encodes a name into the signed transaction that is not the name the user selected or was shown at any point in the flow. That is the wallet-argument impact: the transaction the victim approves carries an argument the application substituted.

The consequence is an on-chain identity the victim did not choose, attached to their own wallet, and readable by every ENS consumer.

## Scope correction

I want to be exact about the ceiling here rather than argue past it.

No funds and no NFT move in this transaction. The victim keeps `Treasury.eth` and keeps every right to it, and they can set a different primary name afterwards. The loss is that their wallet's public identity is set to a name under the attacker's control until they notice and correct it, and that the forward and reverse halves of one action describe two different names.

I am filing this under the wallet-argument impact because the substitution is real and is in the signed calldata, but I recognise the program may weigh the consequence lower than the mechanism, and I would rather state that than overclaim.

## Why this is not an already-known issue

The known-issues list contains an entry for name validators accepting homoglyphs, bidirectional controls and other deceptive characters, rated Medium, whose stated impact is display confusion rather than incorrect resolution. This does not depend on visual confusability — plain mixed-case ASCII reproduces it — and no validator is involved. The defect is that one flow uses two different derivations of one selected name.

It is not the transfer seam of report #89960 and not the renewal-route parser split of report #89456: different flow, different files, different builder, and the two strings here are two fields of one indexer row rather than two client-side parsers. It is also not in the resolver-role family, since no role is granted or revoked — the question is only which node each leg addresses.

## Recommended fix

Normalize once, at row construction, and use that single string for the row label, the forward legs and the reverse leg alike. Alternatively, refuse the row when `domain.name` and `domain.normalizedName` disagree, since in that state the application cannot know which name the user means.

## Proof of Concept

**Revision note.** The competition Resources tab names the audited revision as `63772fd872af472ced58b009499355f3430c2a86`. This repository's `audit-comp-ready` branch imports that revision at commit `cda79ac`, and its head is `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, which the scope page's asset links point at. `git diff cda79ac 1c9b47f` touches only `apps/manager/package.json`, `workers/api-worker/package.json`, `packages/dqa-server/package.json` and `pnpm-lock.yaml` — a Hono version bump. No source file in any in-scope asset differs, so every line number below is identical at either revision.

Tested against the competition repository at `audit-comp-ready` HEAD `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

The test imports `resolveDomainLabel` from production source — that is the function producing the string the dialog renders and passes to `onSelectName`. `withEthSuffix` is module-local in `setPrimaryName.ts`, so its two lines are transcribed verbatim and quoted with their line numbers.

No chain, wallet, RPC or local infrastructure is needed beyond the repository's own dependencies. The whole test is below, so nothing outside this report is required.

Save as:

```
apps/manager/src/features/dashboard/poc-primary-name-legs-split.test.ts
```

Run from the repository root:

```
pnpm install --frozen-lockfile
pnpm --filter manager exec vitest run src/features/dashboard/poc-primary-name-legs-split.test.ts --reporter=verbose
```

Observed result on the audited revision:

```
✓ the row the victim clicks is the RAW name
✓ the reverse leg signs the NORMALIZED name instead
✓ the forward leg writes addr() on the raw node, a different node entirely
✓ so one click sets the reverse record to a name the victim may not own
✓ control: a canonical row keeps both legs on one name

Test Files  1 passed (1)
Tests       5 passed (5)
```

Full exact test:

```ts
import { describe, expect, it } from 'vitest'
import { namehash, normalize } from 'viem/ens'
import { resolveDomainLabel } from '@/features/dashboard/utils'

// One "Set as Primary" click produces two legs that address two different names.
//
//  selectedName = resolveDomainLabel(domain)          ChoosePrimaryNameDialog.tsx:81  -> RAW
//  forward leg : saveRecords({ name: selectedName })  ChoosePrimaryNameDialog.tsx:229 -> RAW
//  forward leg : setupControlledResolver({ name: selectedName })              :278    -> RAW
//  reverse leg : submitPrimaryName({ name: selectedName })                    :469
//                  -> setPrimaryName.ts:140  const cleanName = withEthSuffix(name)
//                  -> setPrimaryName.ts:163  args: [ownerAddress, cleanName]  -> NORMALIZED
//
// `resolveDomainLabel` is imported from production. `withEthSuffix` is
// module-local, so its two lines are transcribed verbatim from
// setPrimaryName.ts:48-49.
const withEthSuffix = (name: string) =>
  normalize(name.endsWith('.eth') ? name : `${name}.eth`)

type Domain = {
  id: string
  name?: string | null
  normalizedName?: string | null
}

// Indexer row for a name registered with the literal label "Treasury".
const domain: Domain = {
  id: '0x01',
  name: 'Treasury.eth',
  normalizedName: 'treasury.eth',
}

describe('set primary name splits its two legs across two names', () => {
  it('the row the victim clicks is the RAW name', () => {
    expect(resolveDomainLabel(domain)).toBe('Treasury.eth')
  })

  it('the reverse leg signs the NORMALIZED name instead', () => {
    const selectedName = resolveDomainLabel(domain)
    expect(withEthSuffix(selectedName)).toBe('treasury.eth')
    expect(withEthSuffix(selectedName)).not.toBe(selectedName)
  })

  it('the forward leg writes addr() on the raw node, a different node entirely', () => {
    const selectedName = resolveDomainLabel(domain)
    expect(namehash(selectedName)).not.toBe(namehash(withEthSuffix(selectedName)))
  })

  it('so one click sets the reverse record to a name the victim may not own', () => {
    const selectedName = resolveDomainLabel(domain)
    const displayedAndOwned = selectedName // Treasury.eth
    const signedIntoReverseRecord = withEthSuffix(selectedName) // treasury.eth
    expect(signedIntoReverseRecord).not.toBe(displayedAndOwned)
  })

  it('control: a canonical row keeps both legs on one name', () => {
    const clean: Domain = { id: '0x02', name: 'mine.eth', normalizedName: 'mine.eth' }
    const selectedName = resolveDomainLabel(clean)
    expect(withEthSuffix(selectedName)).toBe(selectedName)
    expect(namehash(selectedName)).toBe(namehash(withEthSuffix(selectedName)))
  })
})
```

Steps to reproduce in the application

1. Attacker owns `treasury.eth`.
2. Attacker registers the literal label `Treasury` at the registrar with `owner` set to the victim's address. The victim now owns `Treasury.eth`, a separate registration.
3. Victim opens the Manager dashboard and the "Choose primary name" dialog. The row renders as `Treasury.eth`.
4. Victim selects that row and confirms, approving the wallet requests.
5. Inspect the final transaction before signing: the reverse call carries `"treasury.eth"`, not `"Treasury.eth"`. The preceding record write carries `namehash("Treasury.eth")`.
6. After it mines, read the victim's reverse record. It names `treasury.eth` — the attacker's registration.
7. If the attacker then sets `treasury.eth`'s ETH address record to the victim's wallet, the bidirectional check passes and ENS-aware interfaces render the victim's wallet as the attacker's name.
