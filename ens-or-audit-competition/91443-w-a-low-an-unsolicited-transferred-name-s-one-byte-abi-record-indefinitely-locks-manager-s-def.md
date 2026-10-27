# 91443 w a low an unsolicited transferred name s one byte abi record indefinitely locks manager s default dashboard

**Submitted on Sep 1st 2026 at 14:26:18 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91443
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
    * Locking up the victim from login
    * Cookie bombing, etc.

## Description

## Summary

Manager uses one broad profile query for several consumers and always enables ABI retrieval. The default dashboard invokes that query for every visible active V2 name even though each row only consumes `theme` and `avatar`.

An attacker can register an ordinary V2 name, set its canonical `PermissionedResolver` ABI record to content type `4` and the single byte `0xbf`, revoke all attacker resolver roles, and transfer the name to an arbitrary fresh victim. The victim does not approve, sign, transact, select the name, or make it primary.

Connected root navigation automatically redirects that victim to `/dashboard`. The dashboard reads the transferred name's unused ABI record. `0xbf` begins an indefinite-length CBOR map but contains no terminating break. The frozen decoder reads beyond EOF without throwing, repeatedly decodes zero values, and never exits its map loop.

I reproduced the denial in two fresh victim browsers. Each received the exact real ABI response and then could not complete a renderer evaluation or navigation for more than 10 seconds. An equal-size `0xa0` finite-map control remained responsive. Manager HTTP, sibling Manager pages, Anvil RPC, and Panoptes remained healthy throughout.

## Finding Description

### Intended behavior

Resolver-supplied ABI data is untrusted. A dashboard row that does not display ABI data should not decode it as a prerequisite for rendering. If optional structured data is decoded, malformed input must terminate with a bounded error and must not block routing or unrelated profile fields.

Receiving an ENSv2 name is ordinary protocol behavior. A recipient should be able to open Manager without first inspecting or sanitizing every record previously configured by the sender.

### Actual behavior

Manager's shared profile service always requests ABI data:

```ts
const records = yield* fromPromise(
  getRecords(client, {
    name,
    texts,
    coins,
    contentHash: true,
    abi: true,
    ignoreInvalidCoinTypes: true,
  }),
  // ...
)
```

The dashboard starts that full query for every visible V2 row:

```ts
const pageProfileRecords = useQueries({
  queries: pageRows.map(({ item, metadata, name }) => ({
    ...profileRecordsQuery(name),
    enabled: item.kind === 'v2' && !metadata.isInGrace,
  })),
  // ...
})
```

The row preview uses only two text records:

```ts
const themeColor = getTextRecordValue(params.records, 'theme')
const avatarRecord = getTextRecordValue(params.records, 'avatar')
```

Ordinary connected root navigation redirects an owner with a domain to the dashboard:

```ts
if (
  hasDomains.data &&
  hasDomains.isSuccess &&
  !hasDomains.isPaused &&
  !landing
) {
  navigate({ to: '/dashboard' })
}
```

ENSJS content type `4` dispatches the returned bytes to CBOR decoding. The effective decoder advances without an EOF check:

```ts
function readUint8(): number {
  return commitRead(1, ta[offset])
}

function readBreak(): boolean {
  if (ta[offset] !== 0xff) {
    return false
  }
  offset += 1
  return true
}
```

Its indefinite-map loop has no other exit:

```ts
case 5:
  const retObject: any = {}
  for (i = 0; i < length || (length < 0 && !readBreak()); ++i) {
    const key = decodeItem()
    retObject[key] = decodeItem()
  }
  return retObject
```

For `0xbf`, the map length is `-1`. At EOF, `readBreak()` returns false, and each out-of-bounds `readUint8()` returns `undefined` while advancing the offset. JavaScript bitwise coercion converts `undefined` into major type zero and value zero. Every iteration assigns `retObject[0] = 0`; no future offset can observe a break byte that does not exist.

## Root Cause

The in-scope Manager automatically uses a complete profile query for a lightweight dashboard row and unconditionally selects `abi: true`, even though the consumer only needs `theme` and `avatar`. This makes synchronous decoding of an unused attacker-controlled ABI record a prerequisite for the default dashboard.

The terminal sink is an unchecked-EOF indefinite-map loop in the frozen CBOR decoder. Manager's automatic overfetching turns that primitive into a no-click renderer denial for a recipient of an unsolicited name.

## Impact Explanation

The attack uses one byte of malicious record data. For both fresh attack browsers:

```
malicious ABI value:                           0xbf
payload length:                               1 byte
RPC request body:                            60,744 bytes
RPC response body:                           45,862 bytes
ABI call position:                           175 of 176

attempt 1 exact response after start:          1,845.587 ms
attempt 1 pending before health probes:       10,501.299 ms
attempt 1 total pending until watchdog kill:  11,198.780 ms

attempt 2 exact response after start:          1,693.147 ms
attempt 2 pending before health probes:       10,501.431 ms
attempt 2 total pending until watchdog kill:  10,855.117 ms
```

The `0xa0` equal-size controls reached the same dashboard and had maximum heartbeat drifts of 74.7ms and 68ms. While each poisoned renderer was stuck, sibling Manager pages loaded in 667.575ms and 334.550ms; Manager HTTP responded in 11.938ms and 5.488ms; Anvil RPC and Panoptes also remained responsive.

For every control and attack browser, the unmodified dashboard issued its normal read-only `getNamesForAddress` POST to `https://v1-graphql.ens.dev/subgraph`. The harness passed the exact browser request through unchanged, recorded its 1,236-byte body, and replayed that exact body independently for parseable evidence. Each request contained the corresponding victim address and each replay returned HTTP 200 with `{"data":{"domains":[]}}`. No response was fabricated or supplied to Manager; unrelated external fonts, widgets, and telemetry remained blocked.

The renderer has no natural exit from the loop. Closing or externally terminating the affected tab ends that process, but opening the same normal dashboard with the poisoned owned name reproduces the denial.

## Likelihood Explanation

The attacker needs only ordinary V2-owner operations: register a name, call the standard `setABI(node, 4, 0xbf)` entrypoint, revoke its own resolver roles, and transfer the ERC-1155 name.

The victim takes no action to receive or configure the name. The PoC proves nonce zero, zero ETH balance, no attacker approval, no primary-name assignment, no wallet signature or transaction request, and no retained attacker authority over the resolver. The poisoned name is index zero on the victim's default first dashboard page. Visiting ordinary connected `/` is sufficient to reach the sink automatically.

The payload is one byte, the RPC response is 45,862 bytes, and the attack requires no flood, elevated-gas simulation, large response, browser extension, or compromised service.

## Attack Path / Reproduction Path

{% stepper %}
{% step %}

## The attacker registers an active ENSv2 name through the ordinary V2 flow.

{% endstep %}

{% step %}

## The registration creates the canonical `PermissionedResolver` proxy.

{% endstep %}

{% step %}

## The attacker calls `setABI(node, 4, 0xbf)`.

{% endstep %}

{% step %}

## The attacker reads back content type `4` and exact bytes `0xbf`.

{% endstep %}

{% step %}

## The attacker revokes its complete resolver role bitmap.

{% endstep %}

{% step %}

## The attacker transfers the ENSv2 ERC-1155 token to a fresh victim.

{% endstep %}

{% step %}

## The victim sends no transaction, grants no approval, and does not make the name primary.

{% endstep %}

{% step %}

## Panoptes indexes the victim owner, resolver, and default first-page position.

{% endstep %}

{% step %}

## The connected victim visits Manager's ordinary root route.

{% endstep %}

{% step %}

## Manager automatically redirects to `/dashboard`.

{% endstep %}

{% step %}

## The first dashboard page automatically invokes `profileRecordsQuery(name)`.

{% endstep %}

{% step %}

## `getProfileRecords()` sets `abi: true` even though the row does not use ABI data.

{% endstep %}

{% step %}

## The actual Multicall response returns content type `4` and `0xbf`.

{% endstep %}

{% step %}

## The CBOR decoder enters the nonterminating indefinite-map loop.

{% endstep %}

{% step %}

## Queued evaluation and navigation cannot execute.

{% endstep %}

{% step %}

## A new browser visiting the same dashboard repeats the lock because ownership, resolver, and ABI state remain unchanged.

{% endstep %}
{% endstepper %}

## External Preconditions

* The target uses the frozen Manager source at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` or equivalent affected code.
* The attacker can register, configure, and transfer one ENSv2 name.
* The victim can receive the ENSv2 ERC-1155 token.
* The victim has an ordinary connected Manager session and visits the root route.

## Internal Preconditions

* The transferred name is active and appears on the default first dashboard page.
* Its resolver remains assigned after transfer.
* The ABI record remains content type `4` with bytes `0xbf`.
* The root route redirects the connected owner to `/dashboard`.
* `MyNamesList` launches the broad profile query.
* `getProfileRecords()` unconditionally sets `abi: true`.
* The effective CBOR decoder retains the unchecked EOF behavior.

## Affected Code

Primary in-scope paths at frozen commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`:

* [`profileRecords.ts#L82-L169`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileRecords.ts#L82-L169) uses the shared query and unconditional `abi: true`.
* [`MyNamesList.tsx#L221-L265`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/dashboard/components/MyNamesList.tsx#L221-L265) automatically queries every visible active V2 row.
* [`nameRowProfileRecords.ts#L18-L32`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/dashboard/components/nameRowProfileRecords.ts#L18-L32) consumes only theme and avatar.
* [`NamesTable.tsx#L95-L109`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/dashboard/components/NamesTable.tsx#L95-L109) defines the default owned/name-ascending view.
* [`DashboardPage.tsx#L38-L144`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/dashboard/pages/DashboardPage.tsx#L38-L144) renders the default table.
* [`routes/index.tsx#L58-L126`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/index.tsx#L58-L126) redirects an ordinary connected owner to the dashboard.

Effective frozen decode path:

* frozen ENSJS `src/utils/coders/getAbi.ts:96-145` dispatches content type `4` to CBOR.
* effective `@ensdomains/address-encoder/src/utils/cbor.ts:49-64,101-120,122-138,187-205,273-304` contains the unchecked EOF reads, indefinite-map loop, and unreachable trailing-byte check.
* the relevant CBOR source in the frozen ENSJS encoder 1.1.1 and Manager's direct encoder 1.1.4 is byte-identical with SHA-256 `d76bd2f5bc1cf9f9e3d97d7562f94d5a957555a39a58a8aaf89fa59802db1190`.

## Recommended Mitigation

Use a dashboard-specific record query that requests only the text records the row consumes and does not request ABI data. Optional ABI decoding on profile routes should also reject truncated indefinite-length CBOR before iteration and execute behind a bounded failure path so one resolver value cannot block navigation. Add a regression test in which `ABI(node, 4)` returns `0xbf` and assert that the dashboard remains responsive while the malformed optional record is ignored.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/24084a36216c9800d0c642f5740054fe/c6a5319e8b1ae49e053212783faf87ea5ba958aa>

## Proof of Concept

### Description

This full E2E PoC runs the unmodified frozen production Manager, the deployed ENS V2 contracts on a fixed Sepolia fork, real VerifiableFactory-created `PermissionedResolver` proxies, real Panoptes, and real Chromium. It registers ordinary attack and equal-size-control names, writes one-byte ABI records `0xbf` and `0xa0`, revokes the attacker resolver roles, and transfers each ERC-1155 name to a distinct code-empty, zero-nonce victim. Neither victim signs, transacts, approves the attacker, selects a primary name, or opens the transferred name.

For each browser, ordinary connected root navigation automatically reaches the default dashboard. The harness captures and decodes the exact Manager Multicall request and response. Both one-byte `0xa0` controls stay responsive. Both fresh browsers that receive `0xbf` complete the real ABI RPC response and then leave renderer evaluation and navigation pending for more than ten seconds, while a sibling Manager page, Manager HTTP, Anvil, and Panoptes stay responsive. The target Manager, resolver, indexer, RPC response, decoder, and browser are not mocked or patched.

The browser also sends Manager's exact normal V1 inventory POST unchanged. The harness records the request, proves that it is the read-only `getNamesForAddress` operation for the victim, and independently replays the same body to prove the real V1 inventory is empty. The replay is evidence only and is never returned to Manager. Every unrelated external request is blocked.

The runner fails closed on repository revision, authored-file hashes, fork anchor, image digests, resolver/ownership/authority state, indexer placement, wallet method allowlist, exact ABI response, control responsiveness, two independent attack measurements, sibling health, and final state invariants. It preserves the generated evidence directory and automatically removes only the PoC Manager process, named Docker Compose containers, and disposable local fork volume.

### Where to place files and their names

The Gist clones these ten files under the frozen repository root:

```
poc/manager-cbor-abi-main-thread-denial/
├── README.md
├── SOURCE_SHA256SUMS
├── browser-child.mts
├── docker-compose.poc.yml
├── orchestrate.mts
├── package.json
├── panoptes-contracts.json
├── run.sh
├── setup.mts
└── verify-final.mts
```

### Command to run

Run from an empty parent directory with Git, Docker/Compose, Node.js 22, and `shasum` installed:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git audit-comp-ens
cd audit-comp-ens
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/24084a36216c9800d0c642f5740054fe.git \
  poc/manager-cbor-abi-main-thread-denial
git -C poc/manager-cbor-abi-main-thread-denial checkout --detach \
  c6a5319e8b1ae49e053212783faf87ea5ba958aa

(cd poc/manager-cbor-abi-main-thread-denial && \
  shasum -a 256 -c SOURCE_SHA256SUMS)

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
pnpm --filter e2e exec playwright install chromium

POC_LOG_DIR=/tmp/ens-report9-evidence \
bash poc/manager-cbor-abi-main-thread-denial/run.sh
```

The default read-only archive endpoint is `https://sepolia.gateway.tenderly.co`. Set `SEPOLIA_FORK_URL=https://your-sepolia-archive.example` on the final command to use another archive-capable Sepolia endpoint.

### Logs proving the bug

```
ENS24_EFFECTIVE_DECODER_BINDING ... managerEnsjsAddressEncoderVersion=1.1.1 ... managerDirectAddressEncoderVersion=1.1.4 ... allRelevantCborSourcesMatchExpectedSha256=true
ENS24_NOT_VICTIM_PRIMARY ... victimEnsjsPrimaryRecord=null ... victimReverseName=null
ENS24_PANOPTES_DEFAULT_FIRST_PAGE ... index=0 ... totalOwned=1
ENS24_REAL_TRANSFER_PANOPTES_DELIVERY_READY {"result":"ENS24_REAL_TRANSFER_PANOPTES_DELIVERY_READY",..."scenarios":{"attack":{"name":"000-ens24-cbor-attack-1788257877.eth",..."abiBytes":"0xbf",..."attackerCanRewrite":false,...}}}
ENS24_V1_INVENTORY_REQUEST_VALIDATED {"scenario":"attack","browserRequestContinued":true,"replayStatus":200,"operationName":"getNamesForAddress","requestIncludesVictim":true,"readOnlyQuery":true,"domainCount":0,"requestSha256":"bd2245a4dbedc352c811856499fbfd81f0cdb00767d22d37c20f8902644ae00b","replayResponseSha256":"076863efce4b880e77053e5c81775a4c5204f668006e4daf92d2dd0817930911"}
ENS24_BROWSER_ABI_RPC_RESPONSE {"scenario":"attack","status":200,"requestBytes":60744,"responseBytes":45862,"responseSha256":"d324c76e1bfdc712e3978feeac1aacb1055a521a2567030b21655cb31ba4b619","responseFinishedAfterStartMs":1845.587208032608,"managerPagePathnameAtMatch":"/dashboard"}
ENS24_EQUAL_SIZE_CONTROL_DASHBOARD_PASS {"result":"ENS24_EQUAL_SIZE_CONTROL_DASHBOARD_PASS",..."abiBytes":"0xa0",..."maxHeartbeatDriftMs":74.69999998807907,...}
ENS24_ATTACK_MAIN_THREAD_DENIAL_PROVED {"result":"ENS24_ATTACK_MAIN_THREAD_DENIAL_PROVED","tag":"attack-one",..."abiBytes":"0xbf",..."rendererNavigationEvaluatePending":true,..."blockedForMsBeforeHealth":10501.29941701889,"blockedForMs":11198.78045898676,..."exit":{"code":null,"signal":"SIGKILL"}}
ENS24_EQUAL_SIZE_CONTROL_DASHBOARD_PASS {"result":"ENS24_EQUAL_SIZE_CONTROL_DASHBOARD_PASS",..."abiBytes":"0xa0",..."maxHeartbeatDriftMs":68,...}
ENS24_V1_INVENTORY_REQUEST_VALIDATED {"scenario":"attack","browserRequestContinued":true,"replayStatus":200,"operationName":"getNamesForAddress","requestIncludesVictim":true,"readOnlyQuery":true,"domainCount":0,"requestSha256":"464dc1f4df030f3ce33f6a0e9a4a2db015b3c47e046b5e44148dda513ada25c8","replayResponseSha256":"076863efce4b880e77053e5c81775a4c5204f668006e4daf92d2dd0817930911"}
ENS24_BROWSER_ABI_RPC_RESPONSE {"scenario":"attack","status":200,"requestBytes":60744,"responseBytes":45862,"responseSha256":"d324c76e1bfdc712e3978feeac1aacb1055a521a2567030b21655cb31ba4b619","responseFinishedAfterStartMs":1693.146958053112,"managerPagePathnameAtMatch":"/dashboard"}
ENS24_ATTACK_MAIN_THREAD_DENIAL_PROVED {"result":"ENS24_ATTACK_MAIN_THREAD_DENIAL_PROVED","tag":"attack-two",..."abiBytes":"0xbf",..."rendererNavigationEvaluatePending":true,..."blockedForMsBeforeHealth":10501.431459009647,"blockedForMs":10855.117417037487,..."exit":{"code":null,"signal":"SIGKILL"}}
ENS24_SIBLINGS_HEALTHY_WHILE_ATTACK_LIVES {"attempt":"attack-two","managerHttpStatus":200,..."rpcStatus":200,..."panoptesStatus":200,..."siblingManagerStatus":200,...}
ENS24_E2E_CONTROL_ATTACK_RECOVERY_REPEAT_PASS ... controlMaxDriftMs=74.7 ... freshControlAfterKillMaxDriftMs=68
ENS24_FINAL_CHAIN_BROWSER_INVARIANTS_PASS
ENS24_POC_PASS content-type-4 one-byte CBOR denial reproduced twice
ENS24_CLEANUP_PASS containers and fork state removed
```

The fresh September 1 evidence receipts are hash-bound as follows:

```
80088bc6bcb60a7d92a39f4ea17bf721b83a0cc34e54b788451bff23185f95a9  browser-orchestrator-receipt.json
ad2f766da5e124635914c02dac21efb8019c8618364361e49bb4c26c15ae2fa6  final-invariants.json
025db3f584d77b3ad952ea4306dd5d860ca814fcf311021dc5bec394f5fa8b53  delivery-state.json
```

Running the command generates the complete independent receipt set under `/tmp/ens-report9-evidence`, including the three files above, the exact attack and control RPC bodies, both watchdog receipts, setup/orchestrator/final logs, and the production Manager build/runtime logs.

**Full source:** [**immutable ten-file PoC Gist**](https://gist.github.com/aydarkhusaenov/24084a36216c9800d0c642f5740054fe/c6a5319e8b1ae49e053212783faf87ea5ba958aa)
