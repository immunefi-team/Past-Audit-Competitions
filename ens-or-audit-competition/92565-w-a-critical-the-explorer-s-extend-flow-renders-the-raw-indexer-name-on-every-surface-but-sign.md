# 92565 w a critical the explorer s extend flow renders the raw indexer name on every surface but signs the ensip 15 normalized label so a renewal reached from the address names table pays to exte&#x20;

**Submitted on Sep 10th 2026 at 17:28:00 UTC by @Tradi3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92565
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief

One parameter, two different names. In the Explorer's Extend flow every surface renders the raw name from the V2 indexer, while the calldata label is that name put through ENSIP-15. For a name normalization changes, the wallet signs a renewal of a different registration than the one displayed, and the stablecoin payment goes to extending it.

U+00AD SOFT HYPHEN is the sharpest case: ENSIP-15 deletes it, raw handling keeps it, and it has no glyph. A row named treasury\[U+00AD].eth is pixel-identical to treasury.eth on screen while the wallet signs renew("treasury", ...).

## Vulnerability details

The signing leg. apps/portal/src/features/renew/hooks/useRenewalTransactions.ts, buildRenewIntent:

```
// ensjs splits the label without normalizing, so pass a normalized 2LD name.
const writeParams = renewNameWriteParameters(params.publicClient, {
  name: `${getLabel(params.name)}.eth`,
  duration: BigInt(params.duration),
  ...
})
```

getLabel is apps/portal/src/utils/token/getLabel.ts:10-18 and runs ens\_normalize plus ens\_split. The comment shows the normalization is deliberate on this side.

The display legs, all raw. The row's name comes straight from the V2 indexer at apps/portal/src/features/dashboard/hooks/useV2NamesWithRolesForAddress.ts:56-64, GraphQL domains(where: { owner: $account }) { name expiryDate }. That raw string is what renders in the table at routes/addr/$addr/names.tsx:180-201, in MultiNameSummaryCard.tsx:41,50, MultiNameConfirmCard.tsx:26,35,66, ExtendNameSettings.tsx:34-35, ExtendNameConfirmation.tsx:32-33,41, and in the transaction id and description at useRenewalTransactions.ts:212,215.

No surface anywhere in the flow renders the string that is actually signed.

## The guard exists on one entry point and not the other

The profile page does validate. apps/portal/src/routes/$name/index.tsx:442 computes isValidEnsName(name) and returns InvalidNameMessage before Profile renders at line 499, so the ExtendNameButton on that page (line 375) is never reached for a non-normalized name. isValidEnsName('treasury\[U+00AD].eth') is false.

The address names table does not. apps/portal/src/routes/addr/$addr/names.tsx contains no isValidEnsName, isNormalized, normalize or getLabel call at all - grep returns zero hits. Its only filters are getSelectedNames and isExtendable2LD from features/renew/utils/nameExtension.ts:17-24, and the crafted row passes both. Both modal paths from that table reach the same builder: single through ExtendNameModal, multi through MultiNameExtendModal and planMultiRenewSteps into buildRenewTransaction.

So the flow is guarded where it does not matter and unguarded where the row actually comes from.

## Preconditions, stated precisely

The table queries domains(where: { owner: $account }) and the code comments "Only include names the address owns" (useV2NamesWithRolesForAddress.ts:55). So the row must be a registration the victim actually holds. "The attacker registers the twin" is not sufficient on its own, because the twin would not appear in the victim's table. The victim comes to hold a non-normalized registration two ways:

Route 1. They registered it themselves. The Manager's registration path applies no ENSIP-15 at any step, so a crafted registration link hands them vi\[U+00AD]talik.eth, whose Explorer row renders as vitalik.eth.

Route 2. The attacker gives them one. The attacker registers the twin and transfers the ERC-1155 to the victim while keeping the canonical name. The victim's table shows a row displayed as treasury.eth; renewing it signs renew("treasury"), which extends the attacker's registration. The victim's stablecoins fund the attacker's name and the name the victim actually holds still expires.

Route 2 requires nothing of the victim beyond renewing a name they believe they own, which is the ordinary workflow. That is why the impact is fund misdirection rather than a display defect.

Registrability is verified on chain, not assumed. Against a local Anvil fork of Sepolia the real registrar's isAvailable returns false for canonical vitalik and true for vi\[U+00AD]talik, vitalik\[U+00AD] and \[U+00AD]vitalik. The deployed contract accepts labels ENSIP-15 deletes.

## Impact details

Two rows, both Critical on this program's impact list:

6288, malicious interactions with an already-connected wallet, modifying transaction arguments or parameters. The label argument in the signed renew call is not the label any surface displayed.

6285, causing the application to display or pre-fill an incorrect name or transaction detail so that a user sends funds to an unintended recipient. The table shows one name, the wallet signs another, and stablecoins move to extend the registration the victim did not intend.

## Relationship to other reports in this competition

This belongs to the same family as the raw-versus-normalized reports already filed here, and I would rather state that than have it read as concealed. The separations are real: a different app (Explorer, not Manager), a different data source (the V2 indexer GraphQL row rather than a profile cache or dashboard table), a different builder (buildRenewIntent), and a different entry point (the address names table). It is also structurally different from the Manager reports, where two columns of one row disagree; here a single parameter is passed through two different functions on its way to the screen and to the calldata.

It is not the known deceptive-characters entry either. That known issue is about validators accepting confusables with display-confusion impact. Here the character is deleted by the specification rather than confused with another, the flow's own guard already rejects it on the sibling route, and the consequence is a payment against a different registration.

## Recommended fix

Normalize once, where the row enters the app, and render and sign the same string. Either apply getLabel to the indexer row in useV2NamesWithRolesForAddress before it becomes a NameRow, or add the isValidEnsName gate that routes/$name/index.tsx:442 already uses to routes/addr/$addr/names.tsx so a non-normalized row cannot be selected for extension. Displaying the exact label that will be signed on the confirmation card would also close it.

## Proof of Concept

Tested against audit-comp-ready at 1c9b47f18fcddd2e864dfe385c4171061c9811ae. That branch commit only bumps Hono over the upstream audited revision 63772fd872af472ced58b009499355f3430c2a86; git diff confirms every Explorer renewal source cited above is unchanged between the two.

The test drives the application's own getLabel and ensjs's renewNameWriteParameters, then decodes the resulting calldata. It needs no chain, no wallet and no local stack beyond the repository's own dependencies. It opens with a control - for a canonical name the displayed label IS the signed label - so the divergence cannot be dismissed as an artefact of the test.

Save as apps/portal/src/features/renew/poc-extend-displays-raw-signs-normalized.test.ts and run:

```
pnpm install --frozen-lockfile
cd apps/portal && ./node_modules/.bin/vitest run src/features/renew/poc-extend-displays-raw-signs-normalized.test.ts --reporter=verbose
```

Observed on the audited revision: 6 passed.

```
control: for a canonical name the displayed label IS the signed label
a soft hyphen (U+00AD) row renders identically but signs a different label
the two labels are two separate registrations
no entry point renders the string it signs
guard boundary: the profile page rejects it, the names table does not
same split for a mapped-away fullwidth label

Test Files  1 passed (1)
Tests       6 passed (6)
```

Full test source:

```ts
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { ethRegistrarRenewSnippet } from '@ensdomains/ensjs-abi/v2/ethRegistrar'
import { renewNameWriteParameters } from '@ensdomains/ensjs/wallet'
import {
  decodeFunctionData,
  encodeFunctionData,
  labelhash,
  namehash,
  type Address,
} from 'viem'
import { sepolia } from 'viem/chains'
import { describe, expect, it } from 'vitest'
import { getSelectedNames, isExtendable2LD } from '@/features/renew/utils/nameExtension'
import type { NameRow } from '@/features/names/components/NamesTable/columns'
import { getLabel } from '@/utils/token/getLabel'
import { isValidEnsName } from '@/utils/token/isNormalized'

const USDC = '0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238' as Address
const ONE_YEAR = 31_536_000n

/** Exactly how the production hook builds the renew calldata. */
const buildProductionRenewCall = (selectedName: string) => {
  const client = {
    chain: extendChainWithEns(sepolia),
    account: {
      address: '0x000000000000000000000000000000000000dEaD' as Address,
      type: 'json-rpc' as const,
    },
  } as unknown as Parameters<typeof renewNameWriteParameters>[0]

  const writeParams = renewNameWriteParameters(client, {
    // verbatim from useRenewalTransactions.ts:197
    name: `${getLabel(selectedName)}.eth`,
    duration: ONE_YEAR,
    paymentToken: USDC,
    contract: 'ensEthRegistrar',
  })

  return encodeFunctionData({
    abi: writeParams.abi,
    functionName: writeParams.functionName,
    args: writeParams.args,
  })
}

const decodeRenewLabel = (data: `0x${string}`) => {
  const { args } = decodeFunctionData({
    abi: ethRegistrarRenewSnippet,
    data,
  }) as { args: readonly [string, bigint, Address, `0x${string}`] }
  return args[0]
}

/** A row exactly as the indexer query + names.tsx build it. */
const indexerRow = (name: string): NameRow =>
  ({
    name,
    expiryDate: new Date(Date.now() + 10 * 24 * 60 * 60 * 1000),
    roleBitmap: '0',
    v1Roles: null,
    protocolVersion: 'ENSv2',
  }) as unknown as NameRow

describe('Explorer Extend: displayed name vs signed renew label', () => {
  it('control: for a canonical name the displayed label IS the signed label', () => {
    const row = indexerRow('treasury.eth')
    const selected = getSelectedNames({ 0: true }, [row])
    expect(selected[0]?.name).toBe('treasury.eth')
    expect(decodeRenewLabel(buildProductionRenewCall(selected[0]!.name))).toBe(
      'treasury',
    )
  })

  it('a soft hyphen (U+00AD) row renders identically but signs a different label', () => {
    const displayedRowName = 'treasury\u00ad.eth'
    const selected = getSelectedNames({ 0: true }, [indexerRow(displayedRowName)])

    expect(selected).toHaveLength(1)
    expect(selected[0]?.name).toBe(displayedRowName)
    expect(isExtendable2LD(selected[0]!)).toBe(true)
    expect(displayedRowName.normalize()).toBe(displayedRowName)

    const signedLabel = decodeRenewLabel(
      buildProductionRenewCall(selected[0]!.name),
    )
    expect(signedLabel).toBe('treasury')
    expect(signedLabel).not.toBe(displayedRowName.split('.')[0])
  })

  it('the two labels are two separate registrations', () => {
    expect(labelhash('treasury\u00ad')).not.toBe(labelhash('treasury'))
    expect(namehash('treasury\u00ad.eth')).not.toBe(namehash('treasury.eth'))
  })

  it('no entry point renders the string it signs', () => {
    const displayedRowName = 'treasury\u00ad.eth'
    const signedName = `${getLabel(displayedRowName)}.eth`
    expect(signedName).not.toBe(displayedRowName)
    expect(displayedRowName.includes(signedName)).toBe(false)
  })

  it('guard boundary: the profile page rejects it, the names table does not', () => {
    const twin = 'treasury\u00ad.eth'
    // routes/$name/index.tsx:442 -> InvalidNameMessage, so the single-name
    // Extend button on the profile page is unreachable for this input.
    expect(isValidEnsName(twin)).toBe(false)
    // The names table applies no such gate: the row survives, is selectable and
    // is extendable, and its name is what the modal displays.
    const selected = getSelectedNames({ 0: true }, [indexerRow(twin)])
    expect(selected).toHaveLength(1)
    expect(isExtendable2LD(selected[0]!)).toBe(true)
  })

  it('same split for a mapped-away fullwidth label', () => {
    const twin = '\uFF54\uFF52\uFF45\uFF41\uFF53\uFF55\uFF52\uFF59.eth'
    expect(isValidEnsName(twin)).toBe(false)
    expect(isExtendable2LD(getSelectedNames({ 0: true }, [indexerRow(twin)])[0]!)).toBe(
      true,
    )
    expect(decodeRenewLabel(buildProductionRenewCall(twin))).toBe('treasury')
    expect(twin.split('.')[0]).not.toBe('treasury')
  })
})
```
