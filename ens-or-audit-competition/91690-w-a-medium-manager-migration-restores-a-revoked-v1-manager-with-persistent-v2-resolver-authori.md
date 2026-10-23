# 91690 w a medium manager migration restores a revoked v1 manager with persistent v2 resolver authority

**Submitted on Sep 3rd 2026 at 16:09:35 UTC by @shadowHunter for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91690
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Description

Manager's V1-name query is fresh for five minutes. The returned `V1Domain` objects contain the V1 registry owner, which migration treats as the manager to restore. If V1 authority changes on chain after a plan becomes ready, `MigrationPage` can synchronously send that cached plan to the migration state machine before a names refresh occurs. The state machine captures the submitted plan for the in-flight flow; a later live query result does not replace or cancel it.

The later preflight is requested with `staleTime: 0`, but that only recomputes from the already stale domain objects. It does not bind the cached manager to a fresh V1 Registry `owner(node)` read. The plan consequently adds `grantRoles(canonicalResource, 16777216, revokedManager)` to the victim's HCA batch. The temporary HCA operator approval is cleaned up, but the role granted to the revoked address remains.

Run `dev-full-20260903-10` used `ens08-1f07c0a538.eth`. Victim `0x3deC62498b77fbd0efaC1D7d67c6eddAD38060C0` first made `0x21C267181A582c059fA2662f1Fe0dbA00bB3b9bA` its V1 registry manager. Manager's Migration page fetched that state and finished building an enabled migration plan; the victim then kept the page open and reclaimed the name on V1 in status-1 transaction `0x918882b28edbb96aae2c86323cebec6bc6cbb97bac4fedef0b632ca24ed94842`. The live V1 registry and BaseRegistrar both reported the victim; an attacker `setOwner` call was denied. The V1 names-query count remained at the settled attacker-manager baseline through the first pending wallet request. After the first migration transaction mined, Manager refreshed at block `11625830` and correctly received the victim as the live manager. That result arrived after the stale plan had been captured and did not cancel or rebuild the in-flight flow. All five expected victim transactions mined, and the decoded HCA call granted the revoked address `ROLE_SET_RESOLVER`.

The attacker-only process then sent status-1 transaction `0x5493b338ca86a8015ad461ef1003d7349b9eb3f3acf3b8917a9919158934e4d6` directly to V2 Registry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`. It called `setResolver` (`0xbc7b6d62`), installed malicious resolver `0xd974497d7c051569e6e053b8c1950d55b8ca662c`, and changed `addr(namehash(ens08-1f07c0a538.eth))` to the attacker. V2 owner remained `0x3deC62498b77fbd0efaC1D7d67c6eddAD38060C0` before and after the attack.

## Scope eligibility caveat

The enumerated scope includes `apps/manager`, and the cache, stale-plan reuse, absence of transaction-bound authority validation or in-flight reconciliation, role-grant construction, and submission path demonstrated here are all in that tree. One in-scope Manager adapter, `service/classifyNames.ts`, delegates the low-level `managerAddress` classification to `@ens-apps/migration`; `packages/migration` was not separately listed in the supplied scope URLs.

This report therefore does **not** submit `packages/migration` as the vulnerable asset. It submits the Manager security boundary: an in-scope flow starts from cached authority and creates a persistent grant without binding the plan to live V1 authority at the submission boundary. Even a later correct refresh does not cancel or reconstruct the captured plan. Because a triager could interpret the imported classifier dependency narrowly, written scope confirmation is advisable before submission. That caveat does not alter the reproduced on-chain behavior.

The local V1 GraphQL response is a faithful controllable snapshot used to prove Manager's cache semantics; the finding is not an allegation that the production indexer is vulnerable.

## Preconditions

1. The victim owns an unwrapped V1 `.eth` registration and is the registrant.
2. Another address is its V1 registry manager when Manager first fetches the victim's V1 names.
3. The victim opens Manager's Migration page while that manager is current and lets the migration estimate/plan become ready.
4. The victim then revokes that manager on chain within the five-minute query freshness window and, without refreshing or leaving the ready Migration page, approves the displayed migration transactions.
5. The revoked manager later uses the incorrectly restored V2 role. No victim key or active victim session is available to the attacker.

## Impact

The victim's expected revocation is undone across the migration boundary. A revoked manager receives durable V2 resolver authority because the victim's wallet signs a plan constructed from stale data. The attacker can substitute a resolver and redirect the victim's ENS resolution while ownership surfaces can continue to show the victim. Depending on how the name is used, this can divert funds sent to the name, redirect application identity, or leave the migrated name operationally controlled by the revoked party.

The proof does not rely on stealing the victim key, injecting the attacker key into the browser, impersonating the victim for migration, or mutating contract storage. It demonstrates a status-1 victim migration and a distinct status-1 attacker transaction against deployed V2 code.

## Severity and in-scope impact

Proposed severity: **Critical**, under unauthorized widening of account/name authority and hijacking an ENS name during a Manager migration.

The affected asset is the enumerated in-scope `apps/manager` migration flow. After the victim has revoked a V1 registry manager, Manager can reuse the old manager from its five-minute query cache and encode a persistent V2 `ROLE_SET_RESOLVER` grant to that address. The revoked address can then install its own resolver in a separately signed transaction and make the victim's name resolve to the attacker while V2 ownership still reports the victim.

## Recommendation

Treat indexer-provided manager fields only as discovery hints. For every selected name, read the authoritative current V1 manager from chain immediately before final plan construction and again at the transaction-submission boundary. Bind that value, block number, node, and selected-name revision to the plan; if any value differs, invalidate the plan and require the user to review a newly rendered manager grant.

Set security-sensitive V1 authority data to `staleTime: 0`, or explicitly invalidate/refetch it after focus, chain/account changes, and any ownership or manager transaction. A fresh preflight must fetch fresh authority rather than reclassify old `V1Domain` objects. Display every persistent grantee and exact role in the final wallet-confirmation UI. Add a regression test that changes the V1 registry owner after the initial names response and asserts that migration aborts before any grant or requests a fresh snapshot.

## Proof of Concept

## Proof of Concept

## Steps to reproduce - actual user and attacker flow

1. Victim V owns `ens08-1f07c0a538.eth` as a normal unwrapped V1 `.eth` NFT.
2. V assigns address A as the name's V1 Registry manager. The BaseRegistrar NFT remains owned by V.
3. V connects the same wallet to Manager, opens the dashboard, chooses **Upgrade Names**, and waits on the Migration page until **Upgrade 1 name** is enabled. Every `getNamesForAddress` response used to build that ready plan records V as registrant and A as registry owner; React Query marks the data fresh for 300,000 ms.
4. Keeping that ready Migration page open, V revokes A through another normal wallet/application flow with a signed, mined `BaseRegistrar.reclaim(tokenId, V)` transaction. Confirm that V1 Registry `owner(node)` and `BaseRegistrar.ownerOf(tokenId)` now both return V, and that a call from A attempting to set the V1 owner is rejected.
5. Without waiting five minutes, refreshing, or leaving the Migration page, V presses **Upgrade 1 name**. Manager synchronously submits the already-ready plan whose cached manager is A.
6. Before the first wallet authorization, confirm no transaction has mined and the names-query count is still the settled cache baseline. Authorize the first request. After it mines, observe a live names refresh return V as the current manager while the already-started migration continues unchanged.
7. V approves the remaining four normal migration requests. Together the five transactions deploy the deterministic HCA, approve MigrationHelper for the V1 NFT, temporarily approve the HCA in V2 Registry, execute the HCA batch, and remove the temporary HCA approval.
8. Decode the HCA transaction. Its four inner calls deploy the deterministic resolver, authorize V, call the canonical MigrationHelper, and call `V2Registry.grantRoles(canonicalResource, ROLE_SET_RESOLVER, A)`.
9. Confirm the migration succeeds, V remains the V2 owner, the HCA is owned by V, temporary HCA approval is false, and A now has `ROLE_SET_RESOLVER` despite being revoked before migration.
10. From a process that receives only A's key, have A send `V2Registry.setResolver(rawLabelId, attackerResolver)`. Confirm the receipt is status 1, the resolver is attacker-controlled, `addr(node)` returns A, and V remains the nominal name owner.

## Complete runnable PoC bundle

Google Drive: [**DOWNLOAD `ens-08.rar`**](https://drive.google.com/file/d/13JWiq_IHjZZOb8j6u762mM2SBO0ILhVQ/view?usp=sharing)

Before submitting, replace the link above and confirm that it opens in a signed-out/private browser window without requesting access.

```
Archive: ens-08.rar
Sole authoritative evidence run: dev-full-20260903-10
```

The linked archive is the complete triager bundle for `audit-pocs/ens-08`. It contains all runnable harness sources, the generated full report, Docker/Anvil/Panoptes/Mockestrator configuration, the real Manager Playwright test, attacker-only post-migration action, one-command runner, and the sole qualifying sealed run. That run includes raw transactions and receipts, decoded calldata, debug traces, state snapshots, screenshots, video, Playwright trace/report, browser request evidence, service logs, source snapshot, secret scan, and a recursively verifiable SHA-256 manifest.

The RAR is genuine RAR 5 and was fully extracted during submission QA. All 150 archived files match the current pruned ENS-08 tree byte-for-byte. Some retained Playwright paths are up to 269 characters, so extract with WSL/Linux `unrar` as shown below.

The form's three PNG uploads are supplementary UI evidence. The runnable code and machine-readable authorization, transaction, trace, role, and resolution evidence are in the linked archive.

## Authoritative run and integrity

`dev-full-20260903-10` is a sealed, exit-0 full Manager/HCA run completed at `2026-09-03T13:55:09Z`. Preflight, stack-up, setup, browser, attacker, and evidence-complete stages all exited `0`; Playwright passed its only test; the victim migration and separate attacker action both completed; services were cleaned up; and the run manifest verifies.

The retained run has 120 SHA-256 manifest entries plus the manifest itself. Its secret scan checked disposable-key values, case and prefixless variants in 118 ordinary files and 2,360 ZIP members, with result `clean`. Private keys are intentionally absent from retained evidence.

## Safety and revision

This PoC does not send any transaction to public Sepolia or mainnet. Anvil reads historical Sepolia state at fixed block `11625819`, but every setup, revocation, migration, and attacker transaction is mined only on local Anvil. Manager, RPC, Panoptes, Mockestrator, the PostHog stub, and the relay bind to loopback or an internal Docker network. The browser blocks non-loopback requests.

The browser process receives only a newly generated disposable victim key. The separate attacker process receives only the disposable attacker key and has no victim or fixture-controller key. Recorded Anvil balance/impersonation methods are restricted to establishing the fresh historical-fork fixture and reserving its V2 label; they are not used for the V1 revocation, five Manager migration transactions, attacker `setResolver` transaction, or final state reads.

```
Program-listed upstream audited revision: 63772fd872af472ced58b009499355f3430c2a86
Competition import commit: cda79acaad59711b943fc68207ebb3f1d0ff8596
Exact competition-fork execution commit: 1c9b47f18fcddd2e864dfe385c4171061c9811ae
Branch: audit-comp-ready
Fixed fork block: 11625819
Local chain ID: 11155111
Anvil client: anvil/v1.7.1
RPC: http://127.0.0.1:28546
Manager: http://127.0.0.1:23008
Panoptes relay: http://127.0.0.1:25655/graphql
Mockestrator: http://127.0.0.1:23007
PostHog stub: http://127.0.0.1:22998
```

The later competition-fork commits alter Hono dependency metadata and the lockfile, not the cited Manager source.

## Archive map

```
ens-08/
  README.md
  report.md
  contracts/MaliciousResolver.sol
  panoptes/contracts.fork-11625819.json
  scripts/run.sh
  scripts/assert-local-only.sh
  scripts/docker-compose.fork-11625819.yml
  scripts/docker-compose.loopback.yml
  scripts/generate-manifest.sh
  scripts/scan-secrets.py
  src/setup.ts
  src/attacker-action.ts
  tests/ens-08.manager.spec.ts
  tests/playwright.config.ts
  artifacts/runs/dev-full-20260903-10/
    environment/run-status.txt
    environment/stage-status.tsv
    environment/secret-scan.txt
    environment/setup.json
    ui/browser-evidence.json
    ui/state-after-v1-revocation.json
    ui/manager-migration-transactions.json
    ui/state-after-migration.json
    attacker/attacker-action.json
    transactions/*.json
    ui/*.png
    ui/ens-08-trace.zip
    manifest/sha256.txt
```

## Prerequisites

* WSL2 Ubuntu on x86-64
* Git, Node.js 22, Corepack/pnpm, Docker Engine with Compose, Python 3, `curl`, `jq`, `unrar`, `unzip`, `openssl`, `realpath`, `setsid`, `sha256sum`, and `timeout`
* Enough disk space for a Linux-native repository checkout, dependencies, pinned images, and generated evidence
* A read-only Sepolia archive RPC capable of serving block `11625819`; the credentialless endpoint below was used by the retained run

The retained environment used Ubuntu 24.04.1, Node `v22.23.2`, pnpm `10.27.0`, Docker `29.1.3`, Docker Compose `2.40.3`, Python `3.12.3`, solc `0.8.30`, and the pinned Foundry image below.

## Exact reproduction commands

Run these in WSL Ubuntu. Set `POC_ARCHIVE` to the downloaded RAR. The runner creates fresh disposable keys and a unique local name; never supply funded or production keys.

```bash
set -Eeuo pipefail

export POC_ARCHIVE='/path/to/ens-08.rar'
export EXTRACT_ROOT="$(mktemp -d)"
export ENS08_EXECUTION_REPO='/root/ens08-poc-exec'

test "$(sha256sum "$POC_ARCHIVE" | cut -d ' ' -f 1)" = \
  '4421d92598af61e5c34d0bca19406fcd77e2526da471d27d097da3e55b70eb4f'
unrar x -idq "$POC_ARCHIVE" "$EXTRACT_ROOT/"

git clone --branch audit-comp-ready --single-branch \
  https://github.com/immunefi-team/audit-comp-ens.git \
  "$ENS08_EXECUTION_REPO"
git -C "$ENS08_EXECUTION_REPO" checkout --detach \
  1c9b47f18fcddd2e864dfe385c4171061c9811ae
test "$(git -C "$ENS08_EXECUTION_REPO" rev-parse HEAD)" = \
  '1c9b47f18fcddd2e864dfe385c4171061c9811ae'

cd "$ENS08_EXECUTION_REPO"
corepack enable
corepack pnpm install --frozen-lockfile
corepack pnpm --filter @ens-apps/e2e exec playwright install chromium

docker pull \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd
docker pull \
  ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b
docker pull \
  ghcr.io/ensdomains/mockestrator@sha256:d30d9328d774e15a77f107e02733379078b1806d72598d4ffa151d8b507550ba
docker pull \
  ethereum/solc@sha256:b116bf835554d40c501feab0b2c943a8c5eec003b804bcc5b326b85c93da00c2

mkdir -p /root/ens5-poc-tools
docker create --name ens08-foundry-tools \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd true
docker cp ens08-foundry-tools:/usr/local/bin/cast /root/ens5-poc-tools/cast
docker rm ens08-foundry-tools
docker create --name ens08-solc-tools \
  ethereum/solc@sha256:b116bf835554d40c501feab0b2c943a8c5eec003b804bcc5b326b85c93da00c2 --version
docker cp ens08-solc-tools:/usr/bin/solc /root/ens5-poc-tools/solc-0.8.30
docker rm ens08-solc-tools
chmod 0755 /root/ens5-poc-tools/cast /root/ens5-poc-tools/solc-0.8.30

# run.sh copies the extracted PoC into this ignored execution-tree location.
mkdir -p "$ENS08_EXECUTION_REPO/audit-pocs/ens-08"

export PATH="/root/ens5-poc-tools:/usr/local/bin:/usr/bin:/bin"
export ENS08_SOLC_BIN='/root/ens5-poc-tools/solc-0.8.30'
export SEPOLIA_FORK_URL='https://sepolia.gateway.tenderly.co'
export RPC_URL='http://127.0.0.1:28546'
export MANAGER_URL='http://127.0.0.1:23008'
export INDEXER_URL='http://127.0.0.1:25655/graphql'
export ORCHESTRATOR_URL='http://127.0.0.1:23007'
export POSTHOG_URL='http://127.0.0.1:22998'
export ENS08_RUN_ID='triager-ens08-01'
export ENS08_LABEL_SUFFIX='ens08-triager-01'

cd "$EXTRACT_ROOT/ens-08"
bash ./scripts/run.sh
```

`SEPOLIA_FORK_URL` supplies read-only historical state only to local Anvil. It is not passed to Manager/the browser and never receives a signed transaction. A different read-only archive endpoint can be substituted. Do not retain endpoint credentials in the output.

The runner fails closed unless it is in WSL, every application endpoint is loopback, the RPC identifies as Anvil on chain `11155111`, the execution checkout is at the expected commit, the pinned images are cached, and required files are present. A successful return occurs only after the browser flow, separate attacker action, secret scan, cleanup, manifest creation, and manifest verification finish.

## Expected terminal result

Fresh account, name, resolver, transaction, and run-directory values will differ. Decisive success markers include:

```
[ENS-08] Loopback URL policy passed.
source_bom_scan=clean
[ENS-08] Panoptes GraphQL is ready.
[ENS-08] Environment captured.
[ENS-08] Creating the dynamic V1/V2 local fixture.
[ENS-08] Running the real Manager migration with a victim-only browser.
  1 passed
"result": "confirmed"
policy=local-only
public_broadcast=false
[ENS-08] Captured after chain state.
./ui/state-after-v1-revocation.json: OK
./ui/v1-subgraph-requests.json: OK
[ENS-08] ENS-08 confirmed and artifacts sealed in ...
```

## Decisive output from retained run r10

All transaction hashes below are local-Anvil hashes verified by included raw transactions, receipts, decoded calldata, debug traces, and post-state. They are not public block-explorer references.

```
Run: dev-full-20260903-10
Mode: full-manager-hca
Runner exit status: 0
All six recorded stages: exit 0
Playwright: 1 expected, 0 unexpected

Name: ens08-1f07c0a538.eth
Node: 0x2d95307352205746ee62a354af8059e33b9d68bfad2a8c8caa6e315d398cb177
Victim V: 0x3deC62498b77fbd0efaC1D7d67c6eddAD38060C0
Revoked manager/attacker A: 0x21C267181A582c059fA2662f1Fe0dbA00bB3b9bA

Cached getNamesForAddress baseline: 3 responses
Manager in all cached responses: A
Registrant in all cached responses: V
React Query freshness window: 300000 ms

Victim reclaim tx: 0x918882b28edbb96aae2c86323cebec6bc6cbb97bac4fedef0b632ca24ed94842
Reclaim block/status: 11625829 / success
Live V1 Registry owner after reclaim: V
BaseRegistrar NFT owner after reclaim: V
Attacker V1 setOwner after reclaim: denied/reverted
Names-query count after reclaim: 3

First wallet request: eth_sendTransaction
Cache age at first authorization: 8488 ms
Block before/while request pending: 11625829 / 11625829
Transaction mined before authorization: false
Names-query count before first authorization: 3

Victim migration transactions, all chain 11155111 and status 1:
1. deploy HCA: 0x7da3b70e26d5beae842c8b5358b2528523e796e358e2db9dc0f724cc04961227
2. approve MigrationHelper: 0x78bca5d3db836c8e40a7b6b9c9ae27b2e8855fe3db3917f40349209b43260434
3. setApprovalForAll(HCA,true): 0x9557b4bf1d39020f2e434551c280d118fb1310efaf918a4165aa045a3f0a8482
4. HCA executeByOwner: 0x7bb26d5018492b3065c29c97ea1f69fc4272416286a8df6dfc76d5d76eff88bb
5. setApprovalForAll(HCA,false): 0x3d01011ab9c352898d21b47b17c2ed0f7e1d6f77f5067c9e82ad499d9b59076b

First migration tx block: 11625830
First post-start live refresh block/manager: 11625830 / V
Total names-query count after migration: 5
Migration series final block: 11625834
Late correct refresh cancelled/rebuilt stale plan: false

HCA inner call sequence:
deployProxy -> authorizeNameRoles -> migrate -> grantRoles
Decoded persistent grant:
grantRoles(
  101934850449695891349183366677573289118705451354913795396078663943230635114496,
  16777216,
  A
)
ROLE_SET_RESOLVER: 16777216 (0x1000000)
Attacker role before/after migration: absent / present

Deterministic HCA: 0x6134eb9BF5Ab5CE2e0213C72D9f2444b275Cf51c
HCA owner after migration: V
V2 name owner after migration: V
Temporary HCA operator approval after cleanup: false
Legitimate resolver before attacker action: 0x22582710111252ad6e855A5F3371D9E2fAAadFCc

Separate attacker-only tx: 0x5493b338ca86a8015ad461ef1003d7349b9eb3f3acf3b8917a9919158934e4d6
Attacker tx block/status: 11625835 / success
From: A
To V2 Registry: 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
Decoded function: setResolver
Selector: 0xbc7b6d62
Attacker resolver: 0xd974497d7c051569e6e053b8c1950d55b8ca662c
Deployed-runtime simulation before send: succeeded
Final Registry resolver: attacker resolver
Final addr(node): A
resolvesToAttacker: true
V2 owner before/after attack: V / V

Secret scan: clean
Manifest: 120 SHA-256 entries, verification passed
```

The first migration transaction caused a live refresh that correctly returned V, but it arrived only after `MigrationPage` had synchronously submitted the cached plan to the state machine. That in-flight plan retained A and ultimately encoded the persistent `grantRoles(..., ROLE_SET_RESOLVER, A)` call. A then exercised that role without the victim key or session.

## Scope boundary demonstrated

The submitted vulnerable asset is the enumerated `apps/manager` boundary: its five-minute authority cache, synchronous stale-plan submission, lack of transaction-bound live-authority validation/in-flight reconciliation, and persistent role-grant construction and submission. This report does not submit `packages/migration` as the vulnerable asset, although one Manager adapter imports its low-level classifier. Written scope confirmation is advisable because that package was not separately enumerated.

The controlled local V1 GraphQL response supplies faithful before/after manager snapshots to exercise Manager's cache semantics. It is not a claim that the production indexer is vulnerable. The actual Manager page, migration state machine, wallet requests, HCA calls, deployed V2 contracts, role state, attacker transaction, and final resolution are exercised and recorded.

## Verify the retained evidence

```bash
cd "$EXTRACT_ROOT/ens-08/artifacts/runs/dev-full-20260903-10"
sha256sum --check manifest/sha256.txt
```

Expected: one `OK` line for each of the 120 manifest entries and exit status `0`.

## Uploaded image evidence

1. `ENS-08-01-stale-plan-after-revocation.png` -- after V's status-1 V1 reclaim returned both live ownership surfaces to V, the still-open Manager page continued to offer **UPGRADE 1 NAME** from the cached A-manager plan. `state-after-v1-revocation.json` supplies the chain/query proof.
2. `ENS-08-02-stale-plan-reaches-wallet.png` -- Manager advanced the stale plan to migration step 1/5 and the first wallet request. It was captured while authorization was pending, before any migration transaction mined; `browser-evidence.json` records the unchanged block and query count.
3. `ENS-08-03-migration-completed.png` -- Manager displayed **Your name has been upgraded!** after all five victim transactions succeeded. The paired decoded transaction and post-state artifacts, rather than the screenshot alone, prove the revoked A received `ROLE_SET_RESOLVER` and later redirected resolution.

The images establish the visible stale-flow sequence. The included signed transactions, status-1 receipts, decoded HCA grant, debug traces, role delta, separate attacker-only `setResolver`, and final resolver/`addr(node)` state establish the security impact.
