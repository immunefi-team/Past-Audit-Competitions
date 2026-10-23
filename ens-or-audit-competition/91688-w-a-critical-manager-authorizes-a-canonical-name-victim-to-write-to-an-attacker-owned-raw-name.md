# 91688 w a critical manager authorizes a canonical name victim to write to an attacker owned raw name resolver

## #91688 \[W\&A-Critical] Manager authorizes a canonical-name victim to write to an attacker-owned raw-name resolver

**Submitted on Sep 3rd 2026 at 15:29:29 UTC by @shadowHunter for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91688
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

### Description

### Description

Manager does not use one consistent identity for a name containing canonically equivalent Unicode. For the raw NFD route `café-qualification-ens06-v2.eth`, its ownership/expiry path normalizes to the NFC name `café-qualification-ens06-v2.eth`, while record resolution and profile transaction construction preserve the raw route value. Those are different on-chain names with different namehashes and different owners.

In the reproduced state, victim `0x72B935f1cE5D436fc555Eb02AFAf24F285ff38db` owns only the NFC name. Attacker `0xCcfE3e898EE111699586B099D6F93F8703bD1FA2` owns the exact raw NFD name and configured resolver `0xf1ce623b66d24b8bf746a888156a7194c6627bd3`. Manager displays the victim as Owner and enables **Edit Profile**, but displays records supplied by the attacker resolver. When the victim saves a harmless description, Manager submits transaction `0x16cf4a22099465e4dd64d9d3cd369307af19e958e2bc1898ef3a2eda3a1a4901` from the victim directly to the attacker resolver with raw node `0xdd5afe96b92070116fde5bc9409af87c97ddf377659ba903116c09fce10dc57e`. The status-1 receipt, decoded calldata, emitted attacker-contract event, and storage delta are included below.

The malicious resolver deliberately allows any caller to write. That permissiveness is attacker-controlled test setup, not the reported bug: an owner may choose resolver code for their own exact name. The bug is Manager borrowing authorization from the visually equivalent but byte-distinct victim name and applying it to the attacker name/resolver.

### Impact

An attacker can register a canonically equivalent raw name, choose arbitrary resolver code, plant attacker-controlled profile/address data, and direct a connected victim to that raw Manager URL. Manager joins the victim's canonical ownership to the attacker's resolver context. A victim who uses the enabled profile editor and approves the resulting transaction invokes attacker-selected contract code as `msg.sender`. The demonstrated call is a harmless text write, but the trust-boundary failure is the same one covered by the program's critical connected-wallet category: wrong target contract and wrong exact name are used for a transaction the UI represents as editing the victim's profile.

This PoC makes no claim of zero-click compromise, direct theft, or public-chain exploitation. One normal wallet approval is required.

### Recommendation

Normalize or reject the name once at the route boundary and carry a single canonical identity through every loader, query key, authorization decision, resolver lookup, rendered record, and transaction builder. Never combine an owner/expiry result for one exact namehash with records or a resolver for another. Before enabling a write, assert that the owner check, resolver address, displayed name, and encoded transaction node were all derived from the same canonical byte sequence. Add regression tests using NFC/NFD pairs with distinct owners and resolvers; the raw route must either be rejected/redirected before data access or remain exact consistently and must not inherit authority from the NFC name.

### Proof of Concept

## Proof of concept

### Actual attack steps

1. Attacker registers exact raw NFD name `café-qualification-ens06-v2.eth`, owns it as `0xCcfE3e898EE111699586B099D6F93F8703bD1FA2`, and selects attacker-controlled resolver `0xf1ce623b66d24b8bf746a888156a7194c6627bd3`. The victim never owns this exact name.
2. Victim owns the distinct NFC name `café-qualification-ens06-v2.eth` as `0x72B935f1cE5D436fc555Eb02AFAf24F285ff38db`.
3. Attacker sends the connected victim a Manager link containing the raw NFD spelling. It renders visually like the victim's NFC name.
4. Manager's ownership path normalizes the route and displays `0x72B935f1cE5D436fc555Eb02AFAf24F285ff38db` as Owner, while its record path resolves the exact raw name and displays the attacker address `0xCcfE3e898EE111699586B099D6F93F8703bD1FA2`.
5. Because Manager believes the connected victim owns the route, it enables **Edit Profile**.
6. Victim edits the description and clicks **Save Profile**, then approves the wallet request generated by Manager.
7. Manager sends `0x16cf4a22099465e4dd64d9d3cd369307af19e958e2bc1898ef3a2eda3a1a4901` from `0x72B935f1cE5D436fc555Eb02AFAf24F285ff38db` to `0xf1ce623b66d24b8bf746a888156a7194c6627bd3`, not to the victim's canonical resolver. Calldata is `setText(0xdd5afe96b92070116fde5bc9409af87c97ddf377659ba903116c09fce10dc57e, "description", "local-only ENS-06 victim-authorized proof")`.
8. The attacker resolver executes successfully, emits `ResolverWriteCaptured` with caller `0x72B935f1cE5D436fc555Eb02AFAf24F285ff38db`, increments `writeCount` from 0 to 1, and persists the raw-name record.

### Complete runnable Coded PoC bundle

Google Drive: [**DOWNLOAD `ens-06.rar`**](https://drive.google.com/file/d/1EFJilgfNC5jfvlOtZ7qWw9e_sXxtJERN/view?usp=sharing)

```
Archive: ens-06.rar
Format: RAR 5
Authoritative successful run: qualification-ens06-20260903T112104Z
```

The linked archive is the complete, untruncated `audit-pocs/ens-06` workspace—not a source excerpt. It contains the malicious-resolver test contract, setup code, Docker/Anvil/Panoptes configuration, real Manager Playwright test, one-command runner, generated reports, raw transactions and receipts, decoded calldata, event and storage evidence, screenshots, video, Playwright trace, browser evidence, targeted secret-scan result, and a recursively verifiable SHA-256 manifest. No source file required by the runner is omitted.

The form's image uploads are supplementary UI evidence only. The runnable code and machine-readable transaction/state evidence are in the linked archive.

#### Which retained run is evidence

Only `artifacts/runs/qualification-ens06-20260903T112104Z` is presented as the successful PoC. It has a passed Playwright result, all four stages at exit `0`, wrapper exit status `0`, status-1 exploit receipt, confirmed post-state, clean targeted secret scan, and a sealed manifest.

For auditability, the complete workspace also retains:

* `qualification-ens06-20260903T110638Z`: an earlier failed qualification whose original decoder rejected a valid direct `setText` transaction shape. It is expressly disclosed as failed and is not relied upon.
* `dev-setup-20260903-ens06-1` and `dev-setup-20260903-ens06-2`: unsealed development setup captures. They are not submitted as proof.

The successful run's scanner checked its exact disposable victim/attacker key values, common textual variants, and ZIP members and returned `result=clean`. This is not a claim that the later outer Drive archive underwent a generic credential scan.

### Safety and revision

The PoC does not send a transaction to public Sepolia or mainnet. Anvil reads Sepolia state at fixed block `11625819`, but every setup and exploit transaction is signed and mined only on loopback. Manager, RPC, and the Panoptes relay are loopback-only; the browser blocks non-loopback HTTP/WebSocket requests and receives only the disposable victim key.

```
Program-listed upstream audited revision: 63772fd872af472ced58b009499355f3430c2a86
Competition import commit: cda79acaad59711b943fc68207ebb3f1d0ff8596
Exact competition-fork execution commit: 1c9b47f18fcddd2e864dfe385c4171061c9811ae
Branch: audit-comp-ready
Fork block: 11625819
Local chain ID: 11155111
RPC: http://127.0.0.1:18547
Manager: http://127.0.0.1:13001
Panoptes: http://127.0.0.1:15656/graphql
```

### Archive map

The principal entry points are:

```
ens-06/
  README.md
  report.md
  report.full-evidence.md
  contracts/MaliciousProfileResolver.sol
  src/evidence.ts
  src/setup.ts
  tests/ens-06.manager.spec.ts
  tests/playwright.config.ts
  panoptes/contracts.fork-11625819.json
  scripts/run.sh
  scripts/docker-compose.fork-11625819.yml
  scripts/docker-compose.loopback.yml
  scripts/assert-local-only.sh
  scripts/verify-panoptes-labels.sh
  artifacts/runs/qualification-ens06-20260903T112104Z/
    environment/run-status.txt
    environment/stage-status.tsv
    environment/secret-scan.txt
    indexer/label-preservation-proof.json
    ui/wallet-account-proof.json
    ui/state-before.json
    ui/transaction.json
    ui/receipt.json
    ui/browser-evidence.json
    ui/state-after.json
    ui/debug-trace.json
    ui/*.png
    ui/ens-06-trace.zip
    manifest/SHA256SUMS
```

`src/setup.ts` creates the byte-distinct victim NFC and attacker NFD names with real local transactions and deploys the harmless attacker resolver. `tests/ens-06.manager.spec.ts` drives the unmodified Manager UI with only the victim account, proves the wallet request remains pending before authorization, authorizes it, decodes the mined transaction, and independently checks its event and storage effects. Panoptes responses are not mocked or intercepted.

### Prerequisites

* WSL2 Ubuntu on x86-64
* Git, Node.js 22, Corepack/pnpm, Docker Engine with Compose, Foundry `cast`, Solidity compiler `solc` 0.8.30, Python 3, `curl`, `jq`, UnRAR with RAR 5 support, `openssl`, `realpath`, `sha256sum`, and `timeout`
* Enough space for a Linux-native execution worktree and the pinned Docker images
* A read-only Sepolia archive RPC capable of serving block `11625819`; the credentialless endpoint below was used by the retained run

The successful run used Node `v22.23.2`, pnpm `10.27.0`, solc `0.8.30`, Docker `29.1.3`, and Docker Compose `2.40.3`.

### Exact reproduction commands

Run in WSL Ubuntu. Set `POC_ARCHIVE` to the downloaded archive. The victim and attacker keys generated below are disposable and used only on local Anvil.

```bash
set -Eeuo pipefail

export POC_ARCHIVE='/path/to/ens-06.rar'
export BUNDLE_ROOT='/root/ens06-poc-bundle'
export EXTRACT_ROOT="$(mktemp -d)"

test "$(sha256sum "$POC_ARCHIVE" | cut -d ' ' -f 1)" = \
  '7c050b52105e24f1f8029ef01422f1bc12a201e425586b02ab92f548370e4837'
command -v unrar >/dev/null
unrar x -idq "$POC_ARCHIVE" "$EXTRACT_ROOT/"

git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-ens.git "$BUNDLE_ROOT"
git -C "$BUNDLE_ROOT" checkout --detach \
  1c9b47f18fcddd2e864dfe385c4171061c9811ae
test "$(git -C "$BUNDLE_ROOT" rev-parse HEAD)" = \
  '1c9b47f18fcddd2e864dfe385c4171061c9811ae'

mkdir -p "$BUNDLE_ROOT/audit-pocs"
cp -a "$EXTRACT_ROOT/ens-06" "$BUNDLE_ROOT/audit-pocs/"

docker pull \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd
docker pull \
  ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b

export REPO_ROOT="$BUNDLE_ROOT"
export EXECUTION_PARENT="$(mktemp -d /tmp/ens06-exec.XXXXXXXX)"
export EXECUTION_REPO="$EXECUTION_PARENT/repo"
git -C "$REPO_ROOT" worktree add --detach "$EXECUTION_REPO" \
  1c9b47f18fcddd2e864dfe385c4171061c9811ae
mkdir -p "$EXECUTION_REPO/audit-pocs"
cp -a "$REPO_ROOT/audit-pocs/ens-06" "$EXECUTION_REPO/audit-pocs/"

cd "$EXECUTION_REPO"
CI=true corepack pnpm install --frozen-lockfile --ignore-scripts \
  --filter manager... --filter @ens-apps/e2e...
corepack pnpm --filter @ens-apps/e2e exec playwright install chromium

export ENS06_EXECUTION_REPO="$EXECUTION_REPO"
export ENS06_SOLC_BIN="$(command -v solc)"
export ENS06_RUN_ID="qualification-ens06-$(date -u +%Y%m%dT%H%M%SZ)"
export ENS06_LABEL_SUFFIX="triage-$(date -u +%s)"
export SEPOLIA_FORK_URL='https://sepolia.gateway.tenderly.co'
export RPC_URL='http://127.0.0.1:18547'
export INDEXER_URL='http://127.0.0.1:15656/graphql'
export MANAGER_URL='http://127.0.0.1:13001'
export VICTIM_PRIVATE_KEY="0x$(openssl rand -hex 32)"
export ATTACKER_PRIVATE_KEY="0x$(openssl rand -hex 32)"

bash "$REPO_ROOT/audit-pocs/ens-06/scripts/run.sh"
```

`SEPOLIA_FORK_URL` supplies read-only state to local Anvil. It is not passed to Manager/the browser and never receives a signed transaction. A different read-only archive endpoint can be substituted. Never use funded or production keys.

The runner refuses non-loopback application URLs, verifies Anvil and chain ID, byte-compares execution-side PoC files, performs normal local registrations, waits for unmodified Panoptes to preserve both exact labels, starts the audited Manager, pauses at the actual wallet request, authorizes it, verifies the mined receipt/event/storage, scans the successful run for the disposable keys, and seals its manifest.

### Expected terminal result

Fresh account, resolver, suffix, and transaction values will differ. Success ends with:

```
[ENS-06] Loopback URL policy passed.
[ENS-06] Safety gate passed: loopback-only Anvil with local Sepolia chain ID.
[ENS-06] Deploying normal local registrations and attacker resolver.
[ENS-06] Panoptes raw-label preservation proof passed.
[ENS-06] Starting audited Manager on loopback with EOA debug transport.
[ENS-06] Running the real Manager connected-wallet flow.

Running 1 test using 1 worker
  1 passed

[ENS-06] Captured after chain state.
[ENS-06] ENS-06 confirmed and artifacts sealed in <new run directory>.
```

### Decisive output from the successful retained run

All hashes below are local-Anvil transaction hashes and are verified from the included raw transactions, receipts, trace, event, and state—not from a public block explorer. Unicode is written with escapes and UTF-8 hex so form normalization cannot change the identity being proved.

```
Victim V:   0x72B935f1cE5D436fc555Eb02AFAf24F285ff38db
Attacker A: 0xCcfE3e898EE111699586B099D6F93F8703bD1FA2

Victim NFC name:       caf\u00e9-qualification-ens06-v2.eth
Attacker raw NFD name: cafe\u0301-qualification-ens06-v2.eth
Canonical UTF-8: 0x636166c3a92d7175616c696669636174696f6e2d656e7330362d76322e657468
Raw UTF-8:       0x63616665cc812d7175616c696669636174696f6e2d656e7330362d76322e657468
Canonical namehash: 0x29ad2e8f97964d8be79eae38d0944986438942a975d06b2d81c498b7528b6f65
Raw namehash/node:  0xdd5afe96b92070116fde5bc9409af87c97ddf377659ba903116c09fce10dc57e

Canonical owner: V
Canonical resolver: 0x06afC34EcCAeB32Ac705333E0B8656DB5f7A57f7
Raw owner: A
Raw attacker-selected resolver: 0xf1ce623b66d24b8bf746a888156a7194c6627bd3
Victim owns exact raw name: false

Canonical negative control displays attacker address: false
Raw route displays Owner: V
Raw route displays Main receiving address: A
Edit Profile enabled on raw route: true

Browser-exposed accounts: [V]
Attacker private key injected into browser: false

Wallet request kind: eth_sendTransaction
Pending request count: 1
Block before request: 11625832
Block while pending: 11625832
Transaction mined before victim authorization: false

Transaction: 0x16cf4a22099465e4dd64d9d3cd369307af19e958e2bc1898ef3a2eda3a1a4901
From: V
To: attacker resolver 0xf1ce623b66d24b8bf746a888156a7194c6627bd3
Chain ID: 11155111
Receipt status: 1

Decoded function: setText
Decoded node: 0xdd5afe96b92070116fde5bc9409af87c97ddf377659ba903116c09fce10dc57e
Decoded key: description
Decoded value: local-only ENS-06 victim-authorized proof

Emitted event: ResolverWriteCaptured
Event caller: V
Event node: exact attacker-owned raw namehash
writeCount: 0 -> 1
lastCaller: V
lastSelector: 0x10f13a8c
lastNode: exact attacker-owned raw namehash
storedDescription: local-only ENS-06 victim-authorized proof

Playwright: 1 passed (1.6m)
Browser evidence result: confirmed
Runner exit status: 0
Successful-run targeted key scan: clean
```

The permissive resolver is attacker-controlled setup, not the vulnerability. The attacker is entitled to choose code for the exact raw name they own. The defect is that Manager borrows the victim's authorization from a different normalized NFC name, enables editing, then constructs a victim-authorized transaction to the raw name's attacker-selected resolver and raw node.

This is materially beyond the public `EXP-INPUT-005` display-confusion description: the victim never owns the exact raw name, yet the real Manager flow produces a mined status-1 victim-to-attacker-contract transaction and attacker-controlled storage effect.

### Verify the successful retained evidence

```bash
cd "$BUNDLE_ROOT/audit-pocs/ens-06"
bash ./scripts/generate-manifest.sh --verify \
  ./artifacts/runs/qualification-ens06-20260903T112104Z
```

The expected result is one `OK` line per successful-run file and exit status `0`. The recorded statuses are:

```
stack-up       0
setup          0
browser        0
evidence       0
Playwright     passed
wrapper        0
```

### Uploaded image evidence

1. `ENS-06-01-canonical-negative-control.png` — victim-owned canonical NFC control; the attacker address is absent.
2. `ENS-06-02-mixed-victim-owner-attacker-address.png` — decisive raw-route view: victim shown as Owner with **Edit Profile** enabled, while the About record and Main receiving address come from the attacker resolver.
3. `ENS-06-03-profile-save-complete.png` — post-save raw profile displays `local-only ENS-06 victim-authorized proof` while the attacker main address remains.

The images establish the visible UI sequence. The included pending-request evidence, mined status-1 transaction/receipt, decoded raw node, emitted event, debug trace, and exact storage delta establish the connected-wallet security impact.
