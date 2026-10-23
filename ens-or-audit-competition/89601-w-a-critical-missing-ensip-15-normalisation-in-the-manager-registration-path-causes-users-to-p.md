# 89601 w a critical missing ensip 15 normalisation in the manager registration path causes users to pay for and receive a different permanently unresolvable name than the one the app showed as av&#x20;

**Submitted on Aug 20th 2026 at 05:18:43 UTC by @daivo for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89601
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager app never applies ENS name normalisation (ENSIP-15 / `ens_normalize`) anywhere in its registration path — not on URL input, not on search input, not in validation, not before the availability check, and not before building the on-chain call. A user who enters a name containing any character that ENSIP-15 maps or rejects is told the name is available, is quoted a price, pays, and receives a **different label** on-chain. The registered name is unreachable by every ENSIP-15-conformant client (the ENS app, wallets, resolvers), and its `addr` record — which determines where third parties send funds — is written under a namehash no conformant resolver will ever query. On live Sepolia today, six of ten tested inputs already resolve to names that are **unavailable**, while the label the Manager would register is available: a user asking for `ｖｉｔａｌｉｋ` is shown "available", quoted **8.000021 USDC/year**, and sold a homograph of `vitalik.eth` — while the real `vitalik` reverts as taken.

## Vulnerability Details

**The Manager knows how to normalise, but the registration path never does it.** `normalize` from `viem/ens` appears in exactly two places in `apps/manager`, both inside the `profile` feature, and neither is on any registration code path:

```bash
grep -rn "from 'viem/ens'" apps/manager/src
# features/profile/service/profileName.ts:1      import { normalize }   ← profile only
# features/profile/service/setPrimaryName.ts:28  import { normalize }   ← profile only
# features/profile/service/profileImageRecord.ts:7  parseAvatarRecord   (not normalisation)
# features/migration/utils/dnsEncodeName.ts:2       packetToBytes       (not normalisation)
# features/migration/service/migrationApprovalNeeds.ts:1  labelhash     (not normalisation)
```

`profileName.ts` even fails closed correctly — `normalizeEthName` returns `null` when `normalize` throws — and its only consumers are `profileOwner.ts`, `ProfileActions.tsx` and `ProfileEditAction.tsx`. Nothing in `features/landing/check-availability`, `features/register-v2`, `features/shared/registration`, `features/shared/service`, `packages/transaction-manager` or `packages/smart-account` calls it or any other normaliser. So the correct primitive exists in the codebase, is used for viewing a profile, and is absent from the path that spends the user's money.

`apps/portal` normalises properly in three files (`utils/token/{nameValidation,getLabel,isNormalized}.ts`), so the two apps in this monorepo disagree about the same user input.

The repository's own review rule requires normalisation across `apps/**` — `.greptile/config.json:190`:

```json
{
  "id": "normalize-ens-names",
  "rule": "User-supplied ENS names must pass through `normalize` from `viem/ens` before being used in contract calls or queries.",
  "scope": ["apps/**/*.ts", "apps/**/*.tsx"],
  "severity": "medium"
}
```

**Reachability: registration is not behind a feature flag.** The Manager's only PostHog flags are `migration` and `migration-nft` (`src/lib/posthog/feature-flags.ts`); `/register/$name` (`src/routes/register/$name.tsx`) has no gate. The route feeds `parseName(name).label` straight into `RegistrationV2UiProvider`, and the landing search applies no filtering beyond `validateENSName` (`features/landing/check-availability/useCheckAvailability.ts:66-90`). Nothing upstream blocks the inputs below.

Every stage of the path is unnormalised:

**1. URL → name** — `apps/manager/src/utils/domain.ts:27`

```ts
export function normalizeDomainNameFromUrl(name: string): string {
  if (!name || typeof name !== 'string') return ''
  const label = name.toLowerCase().endsWith(ETH_SUFFIX)
    ? name.slice(0, -ETH_SUFFIX.length) : name
  return truncateToMaxBytes(label, MAX_DOMAIN_BYTES) + ETH_SUFFIX
}
```

**2. Search → name** — `apps/manager/src/features/shared/registration/nameUtils.ts:31`

```ts
export const normalizeQuery = (query: string): string => {
  const trimmed = query.trim().toLowerCase()
  if (!trimmed) return ''
  return trimmed.endsWith('.eth') ? trimmed : `${trimmed}.eth`
}
```

`toLowerCase()` is **not** ENSIP-15 normalisation. It does not fold ligatures (`ﬁ` → `fi`), fullwidth forms (`ａ` → `a`), circled letters (`ⓐ` → `a`), superscripts (`²` → `2`), Roman numerals (`ⅸ` → `ix`), or ignorable characters (soft hyphen U+00AD → removed), and it does not reject the labels ENSIP-15 forbids.

**3. Validation** — `nameUtils.ts:validateENSName` and `features/register-v2/utils/name-parser.ts:parseName` are hand-rolled ASCII blocklists (`/[&*@#$%^()[\]{}|\\:;"'<>?,=+~\`!]/\`, a space check, an ASCII-dot check, a codepoint-length check). No Unicode property checks, no normalisation.

**4. Availability — also unnormalised.** `features/shared/service/nameChainContractService.ts:28` calls `getAvailable(publicClient, { name })`. ensjs `dist/actions/public/v2/getAvailable.js`:

```js
const labels = name.split('.');
const leafLabel = labels[0];
// eth-2ld → readContract(ethRegistrar, 'isAvailable', [leafLabel])
```

The raw label goes straight to the registrar.

**5. The on-chain arguments.** `packages/transaction-manager/src/machines/registration/registration.actors.ts`

```ts
// :141  generateCommitment
const cleanName = name.replace('.eth', '')
readContract({ functionName: 'makeCommitment',
               args: [cleanName, ownerAddress, secret, zeroAddress,
                      resolverAddress, duration, REFERER_ADDRESS] })

// :223  encodeRegisterCall
args: [ name.replace('.eth', ''), owner, secret, zeroAddress,
        resolverAddress, duration, paymentToken, REFERER_ADDRESS ]
```

`packages/smart-account/src/providers/rhinestone/registration-calls.ts` (`buildRevealBatch`):

```ts
const name = `${params.label}.eth`
const node = namehash(name)            // viem namehash performs NO ENSIP-15 mapping
...
setAddr(node, COIN_TYPE_ETH, params.wallet)
register(params.label, params.wallet, ...)
setNameWithHCA(params.wallet, params.setPrimaryName)
```

So the unnormalised label reaches `ETHRegistrar.makeCommitment`, `ETHRegistrar.register`, every `setAddr`/`setText`, and the primary-name call.

## Impact Details

**A — the user pays for a name they do not receive.** The purchase UI reports availability, price and total for the string the user typed. The transaction registers a *different* label. On the current Sepolia deployment this is not hypothetical: `ｖｉｔａｌｉｋ`, `ｅｎｓ`, `ⓔⓝⓢ`, `ａｂｃ`, `ⓐbc` and `ab­cd` (soft hyphen) all normalise to names that are **not available**, while the labels the Manager would register **are** available. The registrar quotes a real, payable price for them (read from Sepolia, see PoC Part 1):

```
"ｖｉｔａｌｉｋ"    isAvailable=true    getRegisterPrice =   8.000021 USDC/yr
"ⓔⓝⓢ"        isAvailable=true    getRegisterPrice = 640.000005 USDC/yr
"vitalik"    getRegisterPrice REVERTS — the real name is taken
```

`ⓔⓝⓢ` is billed at the **3-character premium tier** — 640 USDC/yr — because the registrar counts three codepoints. So the victim is charged the premium rate for a homograph of `ens.eth` while the genuine label reverts as unavailable. The user pays the full charge — the registration rent plus the quoted network cost, both debited under one permit — and receives an asset that is not the one advertised.

**B — the received name is permanently inaccessible.** No ENSIP-15-conformant client can reach it: any app that normalises input will resolve the user's typed string to the *other* name. The purchased name cannot be looked up, sent to, or used as a primary name by anyone using standard tooling. This matches the in-scope impact *"leaving a name permanently inaccessible after the flow."*

**C — records are written under an unreachable node.** `setAddr(namehash(unnormalised), COIN_TYPE_ETH, wallet)` writes the `addr` record — the record that determines **where funds are sent** when a third party resolves the name — under a node no conformant resolver will query. The record the user believes they set is not the record that will ever be read.

**D — the flow contradicts itself: one branch normalises, the other does not.** The registration flow sets the primary name through two different branches, and they disagree about the very transform this report is about:

* **EOA branch** — `registrationUi.machine.ts:577` passes the raw label to `submitPrimaryNameForward`, which runs it through `withEthSuffix` → `normalize()` (`profile/service/setPrimaryName.ts:237`). The user registers `ｖｉｔａｌｉｋ` and their reverse record is set to **`vitalik.eth` — a name they do not own**. Bidirectional resolution therefore fails and they end up with no working primary name at all.
* **HCA branch** — `registrationUi.machine.ts:509-510` bundles `asEthName(event.label)` **raw** into `setNameWithHCA`, writing the unnormalised name.
* For labels ENSIP-15 *rejects* (ZWJ, ideographic full stop), `normalize()` throws, so the EOA branch **hard-errors after the user has already paid**.

Part 2b of the PoC demonstrates all three against the production code. This is the clearest statement of the defect: the same feature normalises where it does not matter and skips it where the money is spent.

**E — homograph registration, not homograph display.** An attacker can register `ｖｉｔａｌｉｋ`, `ⓔⓝⓢ`, `ｅｎｓ` and similar as **real on-chain names** that render near-identically to high-value names. The Manager also renders names without the `escapeUnicode` helper that `apps/portal` uses, so the deceptive form is what a victim sees in the Manager UI.

**Loss per event:** the full registration charge (rent + network cost) for an unusable asset, repeated per affected registration, plus the downstream misdirection risk in C.

## Relationship to known issues

The closest published entry is **EXP-INPUT-005** — *"ENS name validators accept inputs that ENS normalization later rejects or transforms — homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot."* This report is distinct because EXP-INPUT-005 is explicitly scoped to *"causing homograph-style display confusion in the UI **rather than incorrect resolution**"*, and presupposes that "ENS normalization later" runs at all. **On this path it never runs.** The Manager's only two `normalize` call sites are in `features/profile`, unreachable from registration; ensjs's `getAvailable` does not normalise; and viem's `namehash` does not normalise. There is no later stage left — the unnormalised label therefore reaches the registrar and the resolver — an on-chain argument defect and incorrect resolution, which is precisely the half EXP-INPUT-005 excludes.

## References

* `apps/manager/src/features/shared/registration/nameUtils.ts:31` (`normalizeQuery`)
* `apps/manager/src/utils/domain.ts:27` (`normalizeDomainNameFromUrl`)
* `apps/manager/src/features/register-v2/utils/name-parser.ts` (`parseName`)
* `apps/manager/src/features/shared/service/nameChainContractService.ts:28`
* `apps/manager/src/features/register-v2/state/registrationUi.machine.ts:509-510,577` (the two primary-name branches)
* `apps/manager/src/features/profile/service/setPrimaryName.ts:237` (`withEthSuffix` — the branch that DOES normalise)
* `apps/manager/src/routes/register/$name.tsx` (no feature-flag gate)
* `packages/transaction-manager/src/machines/registration/registration.actors.ts:141,223`
* `packages/smart-account/src/providers/rhinestone/registration-calls.ts` (`buildRevealBatch`)
* `.greptile/config.json` — rule `normalize-ens-names`
* ENSIP-15 (ENS Name Normalization): <https://docs.ens.domains/ensip/15>
* `apps/portal/src/utils/token/getLabel.ts` — the Explorer's correct handling, for contrast

## Proof of Concept

## Proof of Concept

**Part 1 — the divergence exists on live Sepolia (read-only, no keys, no writes).**

1. Take the Manager's own transforms verbatim from `apps/manager/src/features/shared/registration/nameUtils.ts`: `normalizeQuery(q) = q.trim().toLowerCase()` + `.eth` suffix, and `toLabel(n) = n.endsWith('.eth') ? n.slice(0,-4) : n`.
2. For an input string, compute two labels: the one the Manager sends (`toLabel(normalizeQuery(input))`) and the ENSIP-15 one (`toLabel(ens_normalize(normalizeQuery(input)))`).
3. Query `ETHRegistrar.isAvailable(label)` at `0xa88553F454b77203B0D036A05c894d555EAAa2Cc` on Sepolia for both.
4. Observed at block **11523539**:

```
typed        sent on-chain    ENSIP-15     sent avail  norm avail
ｖｉｔａｌｉｋ      "ｖｉｔａｌｉｋ"        "vitalik"    true        false   <== DIVERGENT
ｅｎｓ          "ｅｎｓ"            "ens"        true        false   <== DIVERGENT
ⓔⓝⓢ          "ⓔⓝⓢ"            "ens"        true        false   <== DIVERGENT
ａｂｃ          "ａｂｃ"            "abc"        true        false   <== DIVERGENT
ⓐbc          "ⓐbc"            "abc"        true        false   <== DIVERGENT
ab­cd        "ab­cd"          "abcd"       true        false   <== DIVERGENT
ﬁre          "ﬁre"            "fire"       true        true
```

In every DIVERGENT row the name the user believes they are buying is **not obtainable**, while the label the Manager will register **is**.

5. The registrar also quotes a real price for those labels. Reading `getRegisterPrice(label, 31536000, USDC)` at the same address:

```
"ｖｉｔａｌｉｋ"    isAvailable=true    price =   8.000021 USDC/yr
"ⓔⓝⓢ"        isAvailable=true    price = 640.000005 USDC/yr   (3-char premium tier)
"ab­cd"      isAvailable=true    price =   8.000021 USDC/yr
"vitalik"    getRegisterPrice REVERTS — the genuine name is taken
```

So the quoted total the user is shown and charged is real, and `ⓔⓝⓢ` is billed at the premium rate for a homograph of `ens.eth`.

Reproduce with the script below (Node 22; run from the repo root after `pnpm install`). It is read-only, uses no keys, and deliberately uses a public keyless RPC rather than any committed provider key.

```js
// Run from the repository root: `node F-001-onchain.cjs` (Node 22, after `pnpm install`).
// Both packages are resolved through `apps/portal`, the workspace package that
// already depends on each of them, so no pnpm store path is hardcoded.
const { createRequire } = require('node:module')
const rq = createRequire(require('node:path').join(process.cwd(), 'apps/portal/package.json'))
const { createPublicClient, http, parseAbi } = rq('viem')
const { sepolia } = rq('viem/chains')
const { ens_normalize } = rq('@adraffy/ens-normalize')

const RPC = 'https://ethereum-sepolia-rpc.publicnode.com'
const REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const abi = parseAbi(['function isAvailable(string label) view returns (bool)'])

// Manager transform, verbatim
const normalizeQuery = (q) => { const t = q.trim().toLowerCase(); return !t ? '' : (t.endsWith('.eth') ? t : `${t}.eth`) }
const toLabel = (n) => (n.toLowerCase().endsWith('.eth') ? n.slice(0, -4) : n)

// pairs: what the user types -> the label the Manager sends on-chain
const probes = ['ﬁre','oﬃce','ⓐbc','x²yz','ⅸabc','ａｂｃ','ab­cd','ｖｉｔａｌｉｋ','ｅｎｓ','ⓔⓝⓢ']

;(async () => {
  const c = createPublicClient({ chain: sepolia, transport: http(RPC) })
  console.log('block:', await c.getBlockNumber(), '\n')
  console.log('typed'.padEnd(12), 'sent on-chain'.padEnd(16), 'ENSIP-15'.padEnd(12), 'sent avail'.padEnd(11), 'norm avail')
  console.log('='.repeat(72))

  for (const typed of probes) {
    const sent = toLabel(normalizeQuery(typed))
    let norm
    try { norm = toLabel(ens_normalize(normalizeQuery(typed))) } catch { norm = null }
    if (norm === null || norm === sent) continue

    const read = async (l) => {
      try { return await c.readContract({ address: REGISTRAR, abi, functionName: 'isAvailable', args: [l] }) }
      catch (e) { return 'ERR' }
    }
    const [aSent, aNorm] = [await read(sent), await read(norm)]
    console.log(
      typed.padEnd(12), JSON.stringify(sent).padEnd(16), JSON.stringify(norm).padEnd(12),
      String(aSent).padEnd(11), String(aNorm),
      aSent === true && aNorm === false ? '   <== DIVERGENT: user buys a DIFFERENT name' : '')
  }
})().catch((e) => console.error('FATAL', e.shortMessage || e.message))
```

**Part 2 — the unnormalised label really reaches the on-chain call.**

1. Save the file below as `packages/smart-account/src/F-001.test.ts`.

```ts
/**
 * Proof of concept — missing ENSIP-15 normalisation in the Manager registration path.
 *
 * Save as: packages/smart-account/src/F-001.test.ts
 * Run:     cd packages/smart-account && pnpm vitest run src/F-001.test.ts
 *
 * Calls the PRODUCTION encoder `buildRevealBatch` from
 * providers/rhinestone/registration-calls.ts with the label the Manager produces
 * for the user input "\uFB01re", and asserts that the unnormalised label — not its
 * ENSIP-15 form — is what reaches ETHRegistrar.register and setAddr.
 */
import { describe, expect, it } from 'vitest'
import { namehash } from 'viem'
import { buildRevealBatch } from './providers/rhinestone/registration-calls'

// Verbatim from apps/manager/src/features/shared/registration/nameUtils.ts
const normalizeQuery = (query: string): string => {
  const trimmed = query.trim().toLowerCase()
  if (!trimmed) return ''
  return trimmed.endsWith('.eth') ? trimmed : `${trimmed}.eth`
}
const toLabel = (name: string): string =>
  name.toLowerCase().endsWith('.eth') ? name.slice(0, -4) : name

const HCA = '0x00000000000000000000000000000000000000aa' as const
const WALLET = '0x00000000000000000000000000000000000000bb' as const
const RESOLVER = '0x00000000000000000000000000000000000000cc' as const

describe('F-001: unnormalised label reaches the on-chain arguments', () => {
  const userTyped = 'ﬁre'          // U+FB01 LATIN SMALL LIGATURE FI + "re"

  it('the label the user believes they are buying differs from the one registered', () => {
    const managerName = normalizeQuery(userTyped)     // "ﬁre.eth"
    const managerLabel = toLabel(managerName)         // "ﬁre"
    // ENSIP-15 result. Hardcoded because @adraffy/ens-normalize is a dependency
    // of apps/portal, not of packages/smart-account, so it cannot be imported
    // here. Part 4 of this report derives the same value from the workspace's
    // own @adraffy/ens-normalize@1.11.1.
    const ensip15 = 'fire.eth'

    expect(managerName).not.toBe(ensip15)
    expect(managerLabel).toBe('ﬁre')
    expect(ensip15).toBe('fire.eth')
  })

  it('register() is encoded with the unnormalised label', () => {
    const managerLabel = toLabel(normalizeQuery(userTyped))

    const calls = buildRevealBatch({
      chainId: 11155111,
      hca: HCA,
      resolver: RESOLVER,
      resolverDeployed: true,
      label: managerLabel,
      wallet: WALLET,
      secret: `0x${'11'.repeat(32)}`,
      price: 1_000_000n,
      duration: 31536000n,
    })

    // step 3 = ETHRegistrar.register — its calldata must contain the raw label
    const registerCall = calls[1]!            // [0]=approve, [1]=register (resolverDeployed)
    const rawHex = Buffer.from(managerLabel, 'utf8').toString('hex')
    expect(registerCall.data.toLowerCase()).toContain(rawHex)

    // and it must NOT contain the ENSIP-15 label
    const normHex = Buffer.from('fire', 'utf8').toString('hex')
    expect(registerCall.data.toLowerCase()).not.toContain(normHex)
  })

  it('setAddr is keyed on namehash(unnormalised), not namehash(normalised)', () => {
    const managerLabel = toLabel(normalizeQuery(userTyped))

    const calls = buildRevealBatch({
      chainId: 11155111,
      hca: HCA,
      resolver: RESOLVER,
      resolverDeployed: true,
      label: managerLabel,
      wallet: WALLET,
      secret: `0x${'11'.repeat(32)}`,
      price: 1_000_000n,
      duration: 31536000n,
    })

    const setAddrCall = calls.find(
      (c) => c.to === RESOLVER && c.data.startsWith('0x8b95dd71'),
    )
    expect(setAddrCall).toBeDefined()

    const wrongNode = namehash(`${managerLabel}.eth`)
    const rightNode = namehash('fire.eth')
    expect(wrongNode).not.toBe(rightNode)
    expect(setAddrCall!.data.toLowerCase()).toContain(wrongNode.slice(2).toLowerCase())
    expect(setAddrCall!.data.toLowerCase()).not.toContain(rightNode.slice(2).toLowerCase())

    console.log('\n  node written  :', wrongNode, `(namehash of "${managerLabel}.eth")`)
    console.log('  node expected :', rightNode, '(namehash of "fire.eth")\n')
  })
})
```

2. Run `cd packages/smart-account && pnpm vitest run src/F-001.test.ts`.
3. The test calls the **production** encoder `buildRevealBatch` from `providers/rhinestone/registration-calls.ts` with the label the Manager produces for the input `ﬁre`, and asserts:
   * the `register` calldata contains the raw label `ﬁre` (UTF-8 `efac817265`);
   * it does **not** contain `fire` (`66697265`);
   * `setAddr` is keyed on `namehash("ﬁre.eth")` and **not** `namehash("fire.eth")`.
4. Result: `Test Files 1 passed (1) · Tests 3 passed (3)`, with

```
node written   0xca9171a6195c2f6859b3f17f275b97fafb6ef1e265632c6c58d36da1085d7209   namehash("ﬁre.eth")
node expected  0xa6678c963886deac4f63785e38c53659f3e06c5485f69ea7bc374d0933c313c1   namehash("fire.eth")
```

**Part 2b — the flow normalises on one branch and not the other.**

1. Save the file below as `apps/manager/src/F-001b.test.ts`.

```ts
/**
 * Proof of concept — the SAME registration flow normalises on one branch and
 * not on the other, so the primary name written on-chain disagrees with the
 * label that was actually registered.
 *
 * Save as: apps/manager/src/F-001b.test.ts
 * Run:     cd apps/manager && pnpm vitest run src/F-001b.test.ts
 *
 * Both call sites are production code reached from `/register/$name`:
 *   EOA branch — registrationUi.machine.ts:577 -> submitPrimaryNameForward
 *   HCA branch — registrationUi.machine.ts:509 -> buildRevealBatch(setPrimaryName)
 * Only the transaction dispatcher is mocked, so the encoded calldata is real.
 */
import { describe, expect, it, vi } from 'vitest'
import { decodeFunctionData } from 'viem'
import { defaultReverseRegistrarSetNameSnippet } from '@ensdomains/ensjs/contracts'

const captured: string[] = []

vi.mock('@ens-apps/transaction-manager', async (importOriginal) => {
  const actual =
    await importOriginal<typeof import('@ens-apps/transaction-manager')>()
  return {
    ...actual,
    transactionManager: {
      startTransaction: (tx: { request: { data: `0x${string}` } }) => {
        captured.push(tx.request.data)
        return 'tx-1'
      },
    },
  }
})

import { submitPrimaryNameForward } from './features/profile/service/setPrimaryName'
import { buildRevealBatch } from '@ens-apps/smart-account'

// Verbatim from register-v2/state/registrationUi.machine.ts:159
const asEthName = (label: string) => `${label}.eth`

const OWNER = '0x00000000000000000000000000000000000000bb' as const

describe('F-001b: the two registration branches disagree about normalisation', () => {
  it('EOA branch: setName is encoded with the ENSIP-15 name, not the registered label', () => {
    captured.length = 0
    const registeredLabel = 'ｖｉｔａｌｉｋ' // exactly what ETHRegistrar.register receives

    submitPrimaryNameForward({
      name: asEthName(registeredLabel),
      signer: { type: 'eoa', walletClient: {} } as never,
      accountAddress: OWNER,
      publicClient: {} as never,
      chainId: 11155111,
    })

    const { args } = decodeFunctionData({
      abi: defaultReverseRegistrarSetNameSnippet,
      data: captured[0] as `0x${string}`,
    })

    console.log('\n  registered label      :', JSON.stringify(registeredLabel))
    console.log('  setName() argument    :', JSON.stringify(args?.[0]))

    // The reverse record points at a name the user does NOT own.
    expect(args?.[0]).toBe('vitalik.eth')
    expect(args?.[0]).not.toBe(asEthName(registeredLabel))
  })

  it('HCA branch: setNameWithHCA is encoded with the RAW label', () => {
    const registeredLabel = 'ｖｉｔａｌｉｋ'
    const calls = buildRevealBatch({
      chainId: 11155111,
      hca: '0x00000000000000000000000000000000000000aa',
      resolver: '0x00000000000000000000000000000000000000cc',
      resolverDeployed: true,
      label: registeredLabel,
      wallet: OWNER,
      secret: `0x${'11'.repeat(32)}`,
      price: 1_000_000n,
      duration: 31536000n,
      setPrimaryName: asEthName(registeredLabel),
    })

    const raw = Buffer.from(asEthName(registeredLabel), 'utf8').toString('hex')
    const norm = Buffer.from('vitalik.eth', 'utf8').toString('hex')
    const blob = calls.map((c) => c.data).join('').toLowerCase()

    console.log('  HCA setNameWithHCA arg: RAW label present =', blob.includes(raw))
    console.log('  HCA setNameWithHCA arg: ENSIP-15 present  =', blob.includes(norm), '\n')

    expect(blob).toContain(raw)
    expect(blob).not.toContain(norm)
  })

  it('EOA branch HARD-FAILS after payment for labels ENSIP-15 rejects', () => {
    captured.length = 0
    const registeredLabel = 'ab‍cd' // U+200D ZWJ — register() accepts it, normalize() throws

    expect(() =>
      submitPrimaryNameForward({
        name: asEthName(registeredLabel),
        signer: { type: 'eoa', walletClient: {} } as never,
        accountAddress: OWNER,
        publicClient: {} as never,
        chainId: 11155111,
      }),
    ).toThrow()
  })
})
```

2. Run `cd apps/manager && pnpm vitest run src/F-001b.test.ts`.
3. Both call sites are production code reached from `/register/$name`; only the transaction dispatcher is mocked, so the encoded calldata is real.
4. Result: `Test Files 1 passed (1) · Tests 3 passed (3)`, with

```
  registered label      : "ｖｉｔａｌｉｋ"
  setName() argument    : "vitalik.eth"
  HCA setNameWithHCA arg: RAW label present = true
  HCA setNameWithHCA arg: ENSIP-15 present  = false
```

The EOA branch writes a reverse record for `vitalik.eth`, which the user does not own; the HCA branch writes the raw `ｖｉｔａｌｉｋ.eth`; and the third case shows the EOA branch throwing outright for a ZWJ label that `ETHRegistrar.register` accepts — after the user has paid.

**Part 3 — end-to-end user path.**

1. Open the Manager at `/register/ｖｉｔａｌｉｋ` (or type `ｖｉｔａｌｉｋ` into search).
2. `normalizeDomainNameFromUrl` / `normalizeQuery` produce `ｖｉｔａｌｉｋ.eth`; `validateENSName` accepts it (no character in it is on the ASCII blocklist).
3. `checkRealNameAvailability` → `getAvailable(client, { name: 'ｖｉｔａｌｉｋ.eth' })` → `isAvailable("ｖｉｔａｌｉｋ")` → **true**. The UI shows the name as available and quotes a price.
4. The user proceeds and signs. `generateCommitment` calls `makeCommitment("ｖｉｔａｌｉｋ", …)` and `encodeRegisterCall` / `buildRevealBatch` calls `register("ｖｉｔａｌｉｋ", …)`.
5. The user has paid for and received the label `ｖｉｔａｌｉｋ`. Any conformant client that normalises their typed string resolves `vitalik.eth` instead — a different name, confirmed unavailable in Part 1.

**Part 4 — the full differential corpus.** The script below reimplements the Manager's `normalizeQuery` / `toLabel` / `validateENSName` verbatim and diffs the label they produce against `ens_normalize`. **15 of 18 probes diverge, and 13 of those 15 pass the Manager's own validator** — only the leading-dot and trailing-dot probes are caught (`INVALID_FORMAT`), which are precisely the two cases EXP-INPUT-005 already names. Every homoglyph, fullwidth, ligature, circled, superscript, Roman-numeral, soft-hyphen, ZWJ and NFC-form probe is accepted and sent on-chain unchanged.

```js
// Run from the repository root: `node norm-diff.cjs` (Node 22, after `pnpm install`).
const { createRequire } = require('node:module')
const rq = createRequire(require('node:path').join(process.cwd(), 'apps/portal/package.json'))
const { ens_normalize } = rq('@adraffy/ens-normalize')

// --- verbatim reimplementation of apps/manager/src/features/shared/registration/nameUtils.ts
const normalizeQuery = (query) => {
  const trimmed = query.trim().toLowerCase()
  if (!trimmed) return ''
  return trimmed.endsWith('.eth') ? trimmed : `${trimmed}.eth`
}
const toLabel = (name) =>
  name.toLowerCase().endsWith('.eth') ? name.slice(0, -4) : name

const INVALID = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
const validateENSName = (name) => {
  const trimmed = name.trim()
  if (!trimmed) return null
  const hasEth = trimmed.toLowerCase().endsWith('.eth')
  const label = hasEth ? trimmed.slice(0, -4).trim() : trimmed.trim()
  if (trimmed.includes(' ')) return 'INVALID_FORMAT'
  if (trimmed.includes('..')) return 'INVALID_FORMAT'
  if (label.includes('.')) return 'INVALID_FORMAT'
  if ([...label].length > 0 && [...label].length < 3) return 'TOO_SHORT'
  if (label.length > 0 && INVALID.test(label)) return 'INVALID_CHARACTER'
  return null
}

const cases = [
  ['ASCII control',        'vitalik'],
  ['ligature fi',          'ﬁre'],                 // ﬁre
  ['ligature ffi',         'oﬃce'],                // oﬃce
  ['Kelvin sign',          'Kelvin'],              // Kelvin
  ['capital sharp s',      'straẞe'],              // straẞe
  ['Turkish dotted I',     'İstanbul'],            // İstanbul
  ['fullwidth latin',      'ｖｉｔａｌ'], // ｖｉｔａｌ
  ['ideographic stop',     'vitalik。eth'],         // vitalik。eth
  ['fullwidth stop',       'vitalik．eth'],         // vitalik．eth
  ['halfwidth ideo stop',  'vitalik｡eth'],         // vitalik｡eth
  ['zero-width joiner',    'ab‍cd'],
  ['soft hyphen',          'ab­cd'],
  ['NFC combining',        'café'],               // café decomposed
  ['roman numeral',        'Ⅸabc'],                // Ⅸabc
  ['circled latin',        'Ⓐbc'],                 // Ⓐbc
  ['superscript',          'x²yz'],                // x²yz
  ['leading dot',          '.vitalik'],
  ['trailing dot',         'vitalik.'],
]

const pad = (s, n) => (s + ' '.repeat(n)).slice(0, n)
const show = (s) => JSON.stringify(s)

console.log('label passes manager validator? | manager label sent on-chain | ens_normalize(name) | AGREE?')
console.log('='.repeat(120))
let diverge = 0
for (const [desc, input] of cases) {
  const v = validateENSName(input)
  const managerName = normalizeQuery(input)      // what the UI treats as the name
  const managerLabel = toLabel(managerName)      // what register()/makeCommitment() receive
  let normalized
  try { normalized = ens_normalize(managerName) } catch (e) { normalized = 'THROWS: ' + String(e.message).slice(0, 40) }
  // The on-chain 2LD the manager actually creates:
  const managerOnChain = managerLabel + '.eth'
  const agree = managerOnChain === normalized
  if (!agree) diverge++
  console.log(
    pad(desc, 20), '|',
    pad(v ?? 'accepted', 16), '|',
    pad(show(managerOnChain), 28), '|',
    pad(show(normalized), 26), '|',
    agree ? 'yes' : '*** NO ***'
  )
}
console.log('='.repeat(120))
console.log(`divergences: ${diverge}/${cases.length}`)
```

## Suggested remediation

Route every user-supplied name through `normalize` from `viem/ens` (or `ens_normalize`) at the single boundary where a name enters the app — `normalizeDomainNameFromUrl` and `normalizeQuery` — and **fail closed** when `ens_normalize` throws, instead of relying on an ASCII blocklist. This is what `.greptile/config.json`'s `normalize-ens-names` rule already mandates and what `apps/portal` already does.
