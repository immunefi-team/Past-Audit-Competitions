# 91724 w a critical extending a name from the address page renews the ensip 15 normalized twin while every screen shows the raw indexer name so the fee extends someone else s registration

**Submitted on Sep 3rd 2026 at 20:53:29 UTC by @SaiyanVegeta for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91724
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Bug Description

**Brief.** In the portal's name-extension flow the display, the expiry card and the renewability gate all use the **raw** indexer name, while the `renew()` calldata is built from the **ENSIP-15-normalized** label. Reached from `/addr/$addr/names`, which applies no normalization at all, a user extending a row that reads `Alice.eth` pays to renew `alice.eth`.

**Details (verified at HEAD).**

*Signed side — ENSIP-15:*

* `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:197` — `renewNameWriteParameters(client, { name: \`${getLabel(params.name)}.eth\`, … })\`. The file's own comment at :191 says *"ensjs splits the label without normalizing, so pass a normalized 2LD name."*
* `apps/portal/src/utils/token/getLabel.ts:10-20` — `getLabel = ens_normalize` + `ens_split`, i.e. real ENSIP-15.
* The price shown beside it is normalized too (`resolvePriceInputs` → `getLabel`), as is the v1 renewability check (`useIsRenewable.ts` → `getLabel`).

*Display / expiry / authorization side — RAW, no canonicalizer:*

* `apps/portal/src/routes/addr/$addr/names.tsx` builds the rows straight from the indexer. A grep of that file for `isValidEnsName|isNormalized|ens_normalize|getLabel` returns **0** — there is no normalization gate anywhere on this entry point.
* `selectedName.name` is carried verbatim into the confirmation heading (`ExtendNameConfirmation.tsx`), the multi-name summary and confirm cards, and the wallet-facing transaction row labels (`Extend ${…}` / `Renew ${…}`).
* The expiry that fills the "new expiry" card comes from `getV2RegistrationDataQueryOptions({ name })`, which queries the indexer with the **raw** string.

*The only gate is a shape regex:*

* `apps/portal/src/features/renew/utils/nameExtension.ts:66` — `isExtendable2LD` is `/^[^.]+\.eth$/`. Uppercase, fullwidth and ignorable code points all pass it. There is no ENSIP-15 check and no on-chain existence check on the v2 path.

**Consequence:** displayed name ≠ priced name ≠ signed name ≠ gated name. And because `useIsRenewable` normalizes before the on-chain check, the safety gate is evaluated against the **twin** — the node the attacker owns and can keep renewable — rather than against the row the user ticked.

***

## Impact

The victim extends a row whose every inspectable field describes their own name: the row text, the confirmation heading, the expiry date, the price, and the wallet's transaction label. The calldata renews a different node.

1. **Fee loss (direct, bounded).** The victim pays a full renewal for a name they do not own, scaled by the duration selected.
2. **Asset loss (indirect, larger).** Their own name is **not** renewed, and the UI tells them it was — the "new expiry" is computed from *their* expiry plus the chosen duration. They get a false confirmation, no reason to re-check, and the name runs to expiry and grace, with the attacker best placed to take it.
3. **Attacker gain.** The attacker's registration is extended at the victim's expense.

No wallet compromise, no cross-origin content, no phishing site: the victim is on the genuine portal throughout.

***

## Risk Breakdown

**This is not the Manager dashboard's bulk-renew path — stated first, because that is the likeliest objection.** Different in-scope **asset** (Explorer app vs Manager app) and entirely different files: the divergence here lives in `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts` + `utils/nameExtension.ts` + `routes/addr/$addr/names.tsx`. A patch to the Manager's dashboard leaves every one of them signing exactly as today, and vice-versa. This path also carries a mechanism the dashboard one does not: the on-chain renewability check is **aimed at the normalized twin**, so the attacker — who owns that twin — controls whether the victim's row passes the gate at all.

* **The obvious refutation, addressed head-on.** The `/$name` route *is* properly gated: `apps/portal/src/routes/$name/index.tsx` computes `isValidEnsName(name)` and renders `InvalidNameMessage` before `Profile`, so `ExtendNameButton` never mounts for a non-normalized name. A crafted-link vector through that route **does not exist**, and I do not claim one. The live entry point is `/addr/$addr/names`, which has no such gate. PoC 1 asserts *both* facts in the same test so this cannot be mistaken for an oversight.
* **Attacker-controlled trigger.** The attacker registers the normalized `alice` and the raw twin `Alice` (both accepted by the real registrar — proven in PoC 2), then transfers the raw twin to the victim; an ERC-721/1155 `transferFrom` to an EOA needs no acceptance. `/addr/$addr/names` lists names by address, so the row appears with no victim action. The victim's only step is the ordinary one: tick it and extend.
* **The indexer serves the raw name honestly.** `Domain.name` and `Domain.normalizedName` are separate fields in the repo's own generated schema, and the v1 path goes through ensjs `makeNameObject` with no normalization. Not a lying-oracle assumption.
* **Not EXP-INPUT-005.** That known issue covers validators that "accept inputs that ENS normalization later rejects or transforms", and scopes itself to display confusion **"rather than incorrect resolution"**. Here the *signed* label differs from the displayed one and funds move to extend a third party's registration — the on-chain deltas are in PoC 2. The fix is not in a validator either; it is in how the row's name is canonicalized before it drives price, gate and calldata.
* **Not WEB-626.** That known issue is the *inverse* of this one: a modal that wrongly implies you are extending **someone else's** name when the name is in fact **yours** (a grace-period copy defect). Here the modal correctly-looking claim is the opposite — it implies the name is yours while the calldata extends someone else's. Its fix is modal copy in the grace-period path; this one is label canonicalization on the extension path. They cannot be patched by the same change.
* **Not QA-05.** That known issue is a functional failure — renewing immediately after registering *fails* with "this name cannot be renewed here". Nothing here fails; the transaction succeeds and extends the wrong node.
* **Severity Critical (the program's tier).** The matched impact is listed under Critical. Honest bounds: loss per event is one renewal fee, the larger harm (the victim's own name lapsing) is delayed rather than immediate, and the victim must select the row and complete the flow.

***

## Recommendation

1. Normalize (or reject) at the boundary: canonicalize the indexer-supplied `name` once in the `NameRow` builder in `routes/addr/$addr/names.tsx`, and render *that* string everywhere.
2. Or, conversely, sign `selectedName.name` unchanged and drop any row where `getLabel(name) + '.eth' !== name` — the important property is that one canonical string drives display, price, gate and calldata.
3. Add the ENSIP-15 check the `/$name` route already performs (`isValidEnsName`) to `isExtendable2LD`, so the shape regex is not the only filter.
4. Point the renewability check at the same label that will be signed, so the gate cannot be satisfied by a node the user is not renewing.

***

## References

* `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:191-197` (comment + `getLabel` in the signed calldata), `:223, :291-292, :459-460` (raw name in the transaction labels)
* `apps/portal/src/utils/token/getLabel.ts:10-20` (`ens_normalize` + `ens_split`)
* `apps/portal/src/features/renew/utils/nameExtension.ts:66` (`isExtendable2LD`, shape regex only)
* `apps/portal/src/routes/addr/$addr/names.tsx` (no normalization gate — grep returns 0)
* `apps/portal/src/routes/$name/index.tsx:442,457-460` (the route that *is* gated, for contrast)
* `apps/portal/src/features/renew/hooks/useIsRenewable.ts:41` (gate evaluated on the normalized label)
* `apps/portal/src/features/renew/components/ExtendNameConfirmation.tsx:33` (raw name in the heading)

***

## Proof of Concept

## Proof of Concept

### PoC 1 — the gate admits it and the calldata carries another label (vitest, the app's real functions, 4/4 pass)

Drop into `apps/portal/src/audit/` and run `npx vitest run src/audit/renew-label-divergence.test.ts` from `apps/portal`.

```ts
import { describe, expect, it } from 'vitest'
import { decodeFunctionData, encodeFunctionData, labelhash, parseAbi } from 'viem'
import { getLabel } from '@/utils/token/getLabel'
import { isExtendable2LD } from '@/features/renew/utils/nameExtension'
import { isValidEnsName } from '@/utils/token/isNormalized'

const renewAbi = parseAbi([
  'function renew(string label, uint64 duration, address paymentToken, bytes32 referrer)',
])
const FUTURE = new Date(Date.now() + 90 * 864e5)

const ROWS = [
  { kind: 'mixed case',  name: 'Nick.eth' },
  { kind: 'fullwidth',   name: 'Ｎick.eth' },
  { kind: 'soft hyphen', name: 'ni­ck.eth' },
] as const

describe('portal renew: displayed raw, signed normalized', () => {
  it.each(ROWS)('$kind — the names table admits it and signs another label', (row) => {
    // the /$name route WOULD reject it — that entry point is genuinely gated
    expect(isValidEnsName(row.name)).toBe(false)
    // ...but the names-table gate is a shape regex only, so it passes
    expect(isExtendable2LD({ name: row.name, isV2: true, expiryDate: FUTURE })).toBe(true)

    const displayedLabel = row.name.replace(/\.eth$/, '')
    const signedLabel = getLabel(row.name)
    expect(signedLabel).not.toBe(displayedLabel)
    expect(labelhash(signedLabel)).not.toBe(labelhash(displayedLabel))

    const data = encodeFunctionData({
      abi: renewAbi, functionName: 'renew',
      args: [signedLabel, 31557600n,
        '0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238',
        '0x0000000000000000000000000000000000000000000000000000000000000000'],
    })
    expect(decodeFunctionData({ abi: renewAbi, data }).args?.[0]).toBe(signedLabel)
    expect(decodeFunctionData({ abi: renewAbi, data }).args?.[0]).not.toBe(displayedLabel)
  })

  it('a canonical row is unaffected (no false positive)', () => {
    expect(getLabel('nick.eth')).toBe('nick')
    expect(isValidEnsName('nick.eth')).toBe(true)
  })
})
```

Result: `Test Files 1 passed · Tests 4 passed`. Note the first two assertions: the gated route rejects the name **and** the names-table gate admits it, in the same test.

### PoC 2 — the exact harm, end-to-end on the real deployed contracts (Sepolia fork)

A single self-contained Node script, run against `anvil --silent --port 8545 --chain-id 11155111 --fork-url <any Sepolia RPC>`. It embeds byte-exact copies of `getLabel` and the `isExtendable2LD` shape gate, registers both names against the **real** deployed v2 registrar (commit → wait `MIN_COMMITMENT_AGE` → register), then approves USDC as the victim and sends the `renew()` the portal would encode, reading `getExpiry` on both nodes before and after. All writes are served by the local fork; nothing is sent to Sepolia. The full run output follows; **the script is available on request** (the submission form does not accept `.mjs` attachments). The decisive lines are:

```js
// EXACT copy of apps/portal/src/utils/token/getLabel.ts
const getLabel = (name) => {
  const normalized = ens_normalize(name)
  const first = ens_split(normalized)[0]
  if (!first || first.error) throw first?.error
  return String.fromCodePoint(...first.input)
}
// EXACT copy of the only gate, features/renew/utils/nameExtension.ts:66
const isExtendable2LDShape = (name) => /^[^.]+\.eth$/.test(name)

const RAW  = 'Nick' + ts                 // the indexer row the victim sees
const NORM = getLabel(RAW + '.eth')      // what buildRenewIntent signs

// victim owns RAW, attacker owns NORM; the victim ticks the row and confirms:
const renewData = encodeFunctionData({ abi: REG, functionName: 'renew',
                                       args: [NORM, DUR, MOCK_USDC, REFERRER] })
await wc(VICTIM).sendTransaction({ to: ETH_REGISTRAR, data: renewData })  // VICTIM pays
// then read getExpiry(RAW) and getExpiry(NORM) before/after
```

```
indexer row (displayed) : "Nick465935.eth"   <- victim owns this
getLabel() -> SIGNED    : "nick465935"       <- attacker owns this
  isExtendable2LD shape gate admits the row : true
  displayed label != signed label           : true
  labelhash differs                         : true

registrar isAvailable(raw mixed-case label): true
registered "Nick465935.eth" -> VICTIM
registered "nick465935.eth" -> ATTACKER

what the confirmation card shows as the expiry of the row:  1790885193 (the VICTIM's node)
calldata decodes to     : renew(label="nick465935", ...)
  transaction row label : "Extend Nick465935.eth"   <- the raw name, verbatim

=== END-TO-END RESULT (real renew tx on real forked contracts) ===
{
  "victim_row_expiry_delta":     "0",
  "attacker_twin_expiry_delta":  "2419200",   // +28 days
  "victim_usdc_spent":           "613701"
}

ASSERTIONS:
  registrar accepted the raw mixed-case label   : true
  shape gate admitted the un-normalized row     : true
  signed label != displayed label               : true
  ATTACKER twin extended                        : true
  VICTIM's own row NOT renewed                  : true
  VICTIM paid                                   : true
```

Every link is demonstrated rather than narrated: the registrar accepts the raw label, the shape gate admits the row, the confirmation card shows the victim's own expiry, the calldata carries the twin, and on-chain the attacker's registration gains the paid 28 days while the victim's gains nothing and the victim's balance falls.
