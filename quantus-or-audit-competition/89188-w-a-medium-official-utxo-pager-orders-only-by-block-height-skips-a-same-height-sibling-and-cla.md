# 89188 w a medium official utxo pager orders only by block height skips a same height sibling and claim never sees that note

**Submitted on Aug 17th 2026 at 17:59:27 UTC by @Omisanin0 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89188
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-apps/tree/audit-comp-ready/quantus\\_sdk>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief

The official Encrypted load and miner/mobile `claimRewards` path pages wormhole transfers 300 at a time with `order_by: {block: {height: asc}}` and `offset += 300`. Height is not unique. When three or more transfers share a height that sits on a 300-row boundary, Postgres/Hasura returns one sibling twice and omits another. I ran the shipped `WormholeUtxoService.getTransfersToMany` against local Postgres using that exact SQL. 2439 rows came back, 2438 unique. `0000620674-d7dfe-000003` was duplicated. `0000620674-d7dfe-000011` (2.59 QUAN) was missing. The same engine with `order_by: [{block.height: asc}, {id: asc}]` returned all 2439. The official disk cache then writes `cachedUpToBlock = tip-180` and later only asks `height_gt` that cutoff, so the missed finalized height is never asked again. `forceReload` deletes the cache and walks the same query, so it misses the same id. `claimRewards` calls `getUnspentTransfers` and never clears the cache. Official Encrypted Available / claim is short that note for as long as the app is the only exit.

## Vulnerability Details

Wormhole notes are keyless. The only official spend path is: list inbound `transfer` rows, compute nullifiers, prove an exit. If a row never enters that list, the secret still owns the leaf and the official client cannot spend it.

The list is built here:

```dart
// quantus_sdk/lib/src/services/wormhole_utxo_service.dart:128-130
static const int _transferPageSize = 300;
static const int _nullifierBatchSize = 300;
static const int _reorgDepth = 180;
```

```dart
// :296-303
query TransfersToAddresses(...) {
  transfers: transfer(
    where: { to: { id: {_in: $tos } }, block: { height: {_gt: $afterBlock } } }
    order_by: {block: {height: asc}}
    limit: $limit
    offset: $offset
  ) { id block { height } ... leafIndex transferCount }
}
```

No `id`. No `leaf_index`. Hasura compiles the nested `order_by` to a `LEFT JOIN LATERAL` on `block.height` and `ORDER BY b.height ASC NULLS FIRST LIMIT 300 OFFSET n`. Postgres is explicit that `LIMIT`/`OFFSET` without a unique `ORDER BY` does not partition the table. Adjacent pages can overlap one tied row and drop another.

`_fetchAllTransfers` is a plain offset walk (`:366-394`):

```dart
offset = 0
while (true) {
  page = _queryTransfers(limit: 300, offset: offset, afterBlock: afterBlock)
  all.addAll(page)
  if (page.length < 300) break
  offset += 300
}
```

First claim / Encrypted `load` starts with an empty cache, so `afterBlock = 0` and the whole history is walked that way.

### What the official walk does to a 300-boundary tie

A dest with 2439 inbound rows and three transfers at height 620674 sitting at unique-order positions 598, 599, 600:

| id                        | height |                    amount |    leaf |  TC |
| ------------------------- | -----: | ------------------------: | ------: | --: |
| `0000620674-d7dfe-000003` | 620674 | 2590000000000 (2.59 QUAN) | 1439401 | 598 |
| `0000620674-d7dfe-000007` | 620674 |             2590000000000 | 1439402 | 599 |
| `0000620674-d7dfe-000011` | 620674 |             2590000000000 | 1439403 | 600 |

Height-only `OFFSET 300` walk (official `getTransfersToMany`, local Postgres, 3/3):

* fetched 2439
* unique 2438
* DUP `0000620674-d7dfe-000003`
* MISS `0000620674-d7dfe-000011`

Same engine, `ORDER BY height, id`: 2439/2439, miss present.

The official service then maps transfers to UTXOs by nullifier. A duplicate id is the same TC, so it collapses to one UTXO. A missing id never becomes a UTXO. Their own log on this run:

```
getUnspentUtxos: 2438 unspent out of 2439 total
```

### Why reload and the next claim do not heal it

After the first walk the service writes the on-disk cache (`:555-562`):

```dart
final all = [...caches[address]!.transfers, ...newByAddress[address]!];
final safe = all.where((t) => t.blockHeight <= safeCutoff).toList();
await _saveTransferCache(..., _TransferCache(cachedUpToBlock: safeCutoff, transfers: safe));
```

`safeCutoff = chainHeight - 180`. Height 620674 is far below that. The missed row's height is treated as already complete.

The next `getTransfersToMany` only asks `height_gt: cachedUpToBlock`. It never re-reads 620674. The comment above that write (`:522-525`) says the filter "correctly fetches only what we don't already have." That is true only if the first walk was a partition. It was not.

`claimRewards` (`wormhole_send_service.dart:250-258`) calls `getUnspentTransfers` and does not call `forceReload` or `clearCachesForAddresses`. Miner claim and mobile redeem both go through that method.

Encrypted pull-to-refresh does `discardCachedState` then `load`, which is `clearCachesForAddresses` plus the same `getUnspentUtxos`. That is a new empty-cache walk of the same height-only query. It misses the same id again.

### Live dests (same query, official pin)

I am not putting those queries in `run.sh`. The program forbids testing on public nets. Independently of the local e2e, the verbatim official query against `https://sub2.quantus.com/v1/graphql` did the same thing:

* dest `qzpmV14n66LkzgdHyDo2WvfxY5K2sZGcF1T6upuKiiNv2sHcA`: 2439/2438, DUP `…-000003`, MISS `…-000011` (2.59 QUAN), 3/3. Unique order complete. `is_deposit_only = true` (wormhole dest, expected).
* dest `qzo3oXnC19kndaqoH5tEZEXC8KwrqC1HjfQG8ghRfWmSbwwSo`: later 9442/9441, DUP `…-000015`, MISS `0000697569-cfb10-000027` (2.52 QUAN, leaf 1601620, TC 3600). Unique order complete. The miss survived +85 new inbound rows.

Planck tip at that check was 905417, `safeCutoff = 905237`. Both missed heights are finalized.

Not every same-height group on a 300-boundary skips. dest1 has four such groups in unique order and only height 620674 actually dups/skips. The class is OFFSET plus a non-unique `ORDER BY`, not "every tie is dropped."

Encrypted `load` also batches every scanned HD address into one `_in` query. The 300-boundary is then on the combined stream, so a dest with fewer than 300 transfers of its own can still lose a sibling.

## Impact Details

Selected in-scope impact (Apps / shared SC, same mapping as the official Encrypted-send fee report):

* High: Temporary freezing of funds for at least 24 hours

Not Direct theft. Not permanent freeze / hardfork. Not a chain Critical.

Local run of the shipped `WormholeUtxoService` (no mock of the walk):

|                             |   Unique `{height,id}` | Official height-only walk |
| --------------------------- | ---------------------: | ------------------------: |
| Rows fetched                |                   2439 |                      2439 |
| Unique ids                  |                   2439 |                      2438 |
| `…-000003`                  |                   once |                     twice |
| `…-000011` (2.59 QUAN)      |                present |                    absent |
| `getUnspentTransfers` UTXOs |                   2439 |                      2438 |
| Spendable                   | 2443.770000000000 QUAN |    2441.180000000000 QUAN |
| Dest credited that note     |                    n/a |      never, official path |

After cache write, a second official `getTransfersToMany` asks `height_gt: 905237` and gets 0 new rows. Miss stays gone. `clearCachesForAddresses` plus a new walk misses the same id.

A phone-only or miner-app user cannot change the GraphQL `order_by`. Recovery is an app or SDK release. That is not a same-day fix. 24 hours is a floor.

Pre-deployment: there is no public circulating supply to multiply by a spot price. Funds at risk are every Encrypted / miner wormhole dest whose inbound count crosses a 300-boundary on a same-height group. Measured on the official pin (not in `run.sh`): 2.59 QUAN + 2.52 QUAN on two dests. This local run freezes 2.59 QUAN.

## References

* `quantus_sdk/lib/src/services/wormhole_utxo_service.dart` 128-130 (`_transferPageSize`, `_reorgDepth`), 290-313 (query), 366-394 (`_fetchAllTransfers`), 522-562 (cache `height_gt: cachedUpToBlock`)
* `quantus_sdk/lib/src/services/wormhole_send_service.dart` 145-163, 239-280 (`claimRewards` -> `getUnspentTransfers`, no cache clear)
* `quantus_sdk/lib/src/services/encrypted_account_service.dart` 190-246 (`forceReload` = `discardCachedState` + same `getUnspentUtxos`)
* `quantus_sdk/lib/src/constants/app_constants.dart` 20 (official pin `https://sub2.quantus.com/v1/graphql`)
* `miner-app/lib/features/withdrawal/claim_rewards_dialog.dart` 126-131
* `mobile-app/lib/v2/screens/settings/redeem_progress_screen.dart` 54-59
* Postgres `LIMIT`/`OFFSET` requires a unique `ORDER BY` to partition: <https://www.postgresql.org/docs/current/queries-limit.html>

Fix on the wallet side: `order_by: [{block: {height: asc}}, {id: asc}]` (or `leaf_index`), or replace OFFSET with keyset pagination. After that, bump the transfer-cache version or drop `wormhole_cache_*.json` so a client that already wrote `cachedUpToBlock` re-walks old heights.

## Proof of Concept

Local only. Quantus is Substrate plus a Flutter SDK, so there is no Hardhat/Foundry fork. This is the local-fork equivalent: shipped `WormholeUtxoService` against local Postgres running the Hasura `LATERAL` SQL for height-only `order_by`.

### How to run

From the repo root. Dart >= 3.8. Homebrew `postgresql@14` (`initdb`, `pg_ctl`, `psql`). Python 3 stdlib. No env vars.

```bash
./poc/F-3041/run.sh
```

Expected last line: `F-3041 E2E PASS`.

{% stepper %}
{% step %}

## Verify the shipped service and prepare local infrastructure

Symlink `sdk/lib/src/services/wormhole_utxo_service.dart` to the shipped file. `cmp` it. Fail if `order_by` is no longer height-only or if `_runClaimFlow` starts clearing the cache.

`initdb` a throwaway cluster on `127.0.0.1:55432`. Load `seed.sql`: 2439 dest rows, three 2.59 QUAN siblings at height 620674 in unique-order slots 598-600.
{% endstep %}

{% step %}

## Start local GraphQL and RPC

Start a 127.0.0.1 GraphQL/RPC. Transfer pages are `psql` of the `LATERAL` SQL above. No Python sort. Nullifier queries return empty (nothing spent). `chain_getHeader` returns tip 905417 so `safeCutoff = 905237`.
{% endstep %}

{% step %}

## Run the official empty-cache transfer walk

Call official `getTransfersToMany([dest])` on an empty cache.
{% endstep %}

{% step %}

## Walk with unique ordering for ground truth

Walk the same tables with `order_by: [{height: asc}, {id: asc}]` for ground truth.
{% endstep %}

{% step %}

## Verify cache behavior

Call official `getTransfersToMany` again (cache + `height_gt: 905237`).
{% endstep %}

{% step %}

## Verify force reload behavior

Official `clearCachesForAddresses` then walk again (`forceReload`).
{% endstep %}

{% step %}

## Verify the claim entry point

Official `getUnspentTransfers` (the `claimRewards` entry).
{% endstep %}
{% endstepper %}

`sdk/lib/src/services/wormhole_utxo_service.dart` is a symlink. Pagination, cache merge, and `getUnspentTransfers` are the shipped class. Local pieces are only the 127.0.0.1 transport and a temp cache directory.

### Run output (this tree)

```
========== STEP 1  official getTransfersToMany (empty cache) ==========
  fetched           2439
  unique            2438
  dups              [0000620674-d7dfe-000003]
  miss 0000620674-d7dfe-000011  false
  safeCutoff        905237

========== STEP 2  unique order_by on the same Postgres ==========
  fetched           2439
  unique            2439
  miss present      true
  missed row        height=620674 amount=2590000000000
  missed value      2.590000000000 QUAN

========== STEP 3  official getTransfersToMany again (cache + height_gt) ==========
  rows              2439
  miss recovered?   false

========== STEP 4  official clearCachesForAddresses + walk (forceReload) ==========
  unique            2438
  miss after reload false

========== STEP 5  official getUnspentTransfers (claimRewards entry) ==========
  official UTXOs    2438
  unique-order rows 2439
  official spendable 2441.180000000000 QUAN
  unique-order total 2443.770000000000 QUAN
  frozen            2.590000000000 QUAN

[WormholeUtxo] getUnspentUtxos: 2438 unspent out of 2439 total
F-3041 E2E PASS
```

### Files

All of this lives in-tree. Paste below is the host that produced the log.

`poc/F-3041/run.sh`

```bash
#!/usr/bin/env bash
# F-3041 local e2e. Does NOT talk to public testnet or mainnet.
set -euo pipefail
ROOT="$(cd "$(dirname "$0")/../.." && pwd)"
POC="$ROOT/poc/F-3041"
SDK="$POC/sdk"
OFFICIAL="$ROOT/../Quantus5-audit/src/quantus-apps/quantus_sdk/lib/src/services/wormhole_utxo_service.dart"
if [[ ! -f "$OFFICIAL" ]]; then
  OFFICIAL="/Users/nikostojak/Quantus5-audit/src/quantus-apps/quantus_sdk/lib/src/services/wormhole_utxo_service.dart"
fi
if [[ ! -f "$OFFICIAL" ]]; then
  echo "FAIL  official wormhole_utxo_service.dart not found"
  exit 1
fi
export F3041_OFFICIAL="$OFFICIAL"
export F3041_PGHOST=127.0.0.1
export F3041_PGPORT=55432
export F3041_PGDATABASE=f3041
export PATH="/opt/homebrew/opt/postgresql@14/bin:/opt/homebrew/bin:${PATH}"
export PGHOST="$F3041_PGHOST"
export PGPORT="$F3041_PGPORT"
export PGDATABASE="$F3041_PGDATABASE"

LINK="$SDK/lib/src/services/wormhole_utxo_service.dart"
mkdir -p "$(dirname "$LINK")"
ln -sfn "$OFFICIAL" "$LINK"
if ! cmp -s "$LINK" "$OFFICIAL"; then
  echo "FAIL  symlink is not the official file"
  exit 1
fi

if grep -q 'sub2.quantus.com' "$POC/local_graphql.py"; then
  echo "FAIL  local_graphql.py mentions the public pin"
  exit 1
fi

PGDATA="$POC/.pgdata"
pg_ctl -D "$PGDATA" status >/dev/null 2>&1 && pg_ctl -D "$PGDATA" stop -m fast >/dev/null || true
rm -rf "$PGDATA"
echo "==> Local Postgres $PGHOST:$PGPORT (initdb)"
initdb -D "$PGDATA" --auth=trust --no-instructions --locale=C >/dev/null
cat >> "$PGDATA/postgresql.conf" <<EOF
port = $F3041_PGPORT
listen_addresses = '127.0.0.1'
unix_socket_directories = '$PGDATA'
EOF
pg_ctl -D "$PGDATA" -l "$POC/.pg.log" start
cleanup() {
  kill "$SERVER_PID" 2>/dev/null || true
  pg_ctl -D "$PGDATA" stop -m fast >/dev/null 2>&1 || true
}
trap cleanup EXIT
for i in 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15; do
  pg_isready -h 127.0.0.1 -p "$F3041_PGPORT" >/dev/null 2>&1 && break
  sleep 0.2
done
createdb -h 127.0.0.1 -p "$F3041_PGPORT" "$F3041_PGDATABASE"
psql -h 127.0.0.1 -p "$F3041_PGPORT" -d "$F3041_PGDATABASE" -v ON_ERROR_STOP=1 -f "$POC/seed.sql" >/dev/null
echo "    seeded 2439 dest rows + 3 siblings at height 620674"

echo
echo "==> Local GraphQL/RPC on 127.0.0.1 (Hasura LATERAL SQL)"
python3 "$POC/local_graphql.py" &
SERVER_PID=$!
for i in 1 2 3 4 5 6 7 8 9 10; do
  python3 -c 'import socket;s=socket.create_connection(("127.0.0.1",4350),1);s.close()' 2>/dev/null && break
  sleep 0.2
done

echo
echo "==> Official WormholeUtxoService (symlink, no Flutter mock of the walk)"
if ! command -v dart >/dev/null 2>&1; then
  echo "FAIL  dart not on PATH"
  exit 1
fi
cd "$SDK"
dart pub get >/dev/null
dart run bin/poc.dart
```

`poc/F-3041/seed.sql`

```sql
-- Local indexer state for F-3041. Same shape as the official dest book:
-- 2439 inbound rows, three 2.59 QUAN siblings at height 620674 occupying
-- unique-order positions 598-600 (the OFFSET 300 boundary).
DROP TABLE IF EXISTS transfer;
DROP TABLE IF EXISTS block;

CREATE TABLE block (
    id INTEGER PRIMARY KEY,
    height INTEGER NOT NULL
);

CREATE TABLE transfer (
    id TEXT PRIMARY KEY,
    block_id INTEGER NOT NULL,
    from_id TEXT NOT NULL,
    to_id TEXT NOT NULL,
    amount TEXT NOT NULL,
    leaf_index TEXT NOT NULL,
    transfer_count TEXT NOT NULL
);

INSERT INTO block (id, height)
SELECT g, g FROM generate_series(100000, 630000) g;

INSERT INTO transfer (id, block_id, from_id, to_id, amount, leaf_index, transfer_count)
SELECT
    CASE
        WHEN g IN (598, 599, 600)
            THEN '0000620674-d7dfe-' || lpad((3 + 4 * (g - 598))::text, 6, '0')
        ELSE 'h' || to_char(
            CASE WHEN g < 598 THEN 100000 + g ELSE 620675 + (g - 601) END,
            'FM0000000000'
        ) || '-synth-' || lpad(g::text, 6, '0')
    END,
    CASE
        WHEN g IN (598, 599, 600) THEN 620674
        WHEN g < 598 THEN 100000 + g
        ELSE 620675 + (g - 601)
    END,
    'qzpmFROM000000000000000000000000000000000000000000',
    'qzpmV14n66LkzgdHyDo2WvfxY5K2sZGcF1T6upuKiiNv2sHcA',
    CASE WHEN g IN (598, 599, 600) THEN '2590000000000' ELSE '1000000000000' END,
    CASE
        WHEN g IN (598, 599, 600) THEN (1439401 + (g - 598))::text
        ELSE g::text
    END,
    CASE
        WHEN g IN (598, 599, 600) THEN (598 + (g - 598))::text
        ELSE g::text
    END
FROM generate_series(0, 2438) g;

ANALYZE block;
ANALYZE transfer;
```

`poc/F-3041/local_graphql.py`

```python
#!/usr/bin/env python3
"""Local Hasura-shaped GraphQL + RPC. 127.0.0.1 only.

Transfer pages are executed as the SQL Hasura emits for
`order_by: {block: {height: asc}}` (LATERAL block.height, NULLS FIRST,
LIMIT/OFFSET). No Python sort. No planted skip list.
"""

from __future__ import annotations

import json
import os
import subprocess
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

HOST = "127.0.0.1"
GQL_PORT = 4350
RPC_PORT = 19944
PGHOST = os.environ.get("F3041_PGHOST", "127.0.0.1")
PGPORT = os.environ.get("F3041_PGPORT", "55432")
PGDATABASE = os.environ.get("F3041_PGDATABASE", "f3041")
TIP = 905417


def _psql(sql: str) -> str:
    env = os.environ.copy()
    env.update(
        {
            "PGHOST": PGHOST,
            "PGPORT": PGPORT,
            "PGDATABASE": PGDATABASE,
            "PGUSER": env.get("USER", "postgres"),
            "PGCONNECT_TIMEOUT": "5",
        }
    )
    return subprocess.check_output(
        ["psql", "-A", "-t", "-v", "ON_ERROR_STOP=1", "-c", sql],
        env=env,
        text=True,
    ).strip()


def _transfers(tos: list[str], limit: int, offset: int, after: int, unique: bool) -> list[dict]:
    dests = ",".join("'" + t.replace("'", "''") + "'" for t in tos)
    order = "b.height ASC NULLS FIRST, t.id ASC" if unique else "b.height ASC NULLS FIRST"
    sql = f"""
    SELECT coalesce(json_agg(row_to_json(x)), '[]') FROM (
      SELECT
        t.id,
        t.amount,
        t.leaf_index AS "leafIndex",
        t.transfer_count AS "transferCount",
        t.from_id,
        t.to_id,
        b.height
      FROM transfer t
      LEFT JOIN LATERAL (
        SELECT height FROM block WHERE id = t.block_id LIMIT 1
      ) b ON TRUE
      WHERE t.to_id IN ({dests}) AND b.height > {int(after)}
      ORDER BY {order}
      LIMIT {int(limit)} OFFSET {int(offset)}
    ) x;
    """
    raw = _psql(sql)
    rows = json.loads(raw) if raw else []
    out = []
    for r in rows:
        out.append(
            {
                "id": r["id"],
                "amount": r["amount"],
                "leafIndex": r["leafIndex"],
                "transferCount": r["transferCount"],
                "toHash": "",
                "block": {"height": r["height"]},
                "from": {"id": r["from_id"]},
                "to": {"id": r["to_id"]},
            }
        )
    return out


def handle_graphql(body: dict) -> dict:
    query = body.get("query") or ""
    variables = body.get("variables") or {}
    if "wormhole_nullifier" in query or "wormholeNullifiers" in query:
        return {"data": {"wormholeNullifiers": []}}
    tos = list(variables.get("tos") or [])
    limit = int(variables.get("limit") or 300)
    offset = int(variables.get("offset") or 0)
    after = int(variables.get("afterBlock") or 0)
    unique = "id: asc" in query or "{id: asc}" in query
    return {"data": {"transfers": _transfers(tos, limit, offset, after, unique)}}


def handle_rpc(body: dict) -> dict:
    if body.get("method") == "chain_getHeader":
        return {
            "jsonrpc": "2.0",
            "id": body.get("id", 1),
            "result": {"number": hex(TIP)},
        }
    return {"jsonrpc": "2.0", "id": body.get("id", 1), "error": {"message": "unknown method"}}


class _Handler(BaseHTTPRequestHandler):
    kind = "gql"

    def log_message(self, fmt, *args):
        return

    def _reject_nonlocal(self) -> bool:
        host = (self.headers.get("Host") or "").split(":")[0]
        if host not in ("127.0.0.1", "localhost", ""):
            self.send_error(403, "local only")
            return True
        return False

    def do_POST(self):
        if self._reject_nonlocal():
            return
        n = int(self.headers.get("Content-Length") or 0)
        raw = self.rfile.read(n)
        try:
            body = json.loads(raw.decode())
        except json.JSONDecodeError:
            self.send_error(400, "bad json")
            return
        if self.kind == "rpc":
            out = handle_rpc(body)
        else:
            out = handle_graphql(body)
        payload = json.dumps(out).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(payload)))
        self.end_headers()
        self.wfile.write(payload)


class GqlHandler(_Handler):
    kind = "gql"


class RpcHandler(_Handler):
    kind = "rpc"


def main() -> None:
    gql = ThreadingHTTPServer((HOST, GQL_PORT), GqlHandler)
    rpc = ThreadingHTTPServer((HOST, RPC_PORT), RpcHandler)
    print(f"F-3041 local GraphQL  http://{HOST}:{GQL_PORT}/v1/graphql", flush=True)
    print(f"F-3041 local RPC      http://{HOST}:{RPC_PORT}", flush=True)
    import threading

    threading.Thread(target=rpc.serve_forever, daemon=True).start()
    gql.serve_forever()


if __name__ == "__main__":
    main()
```

`poc/F-3041/sdk/pubspec.yaml`

```yaml
# Named quantus_sdk so the official wormhole_utxo_service.dart
# `package:quantus_sdk/...` imports resolve. That file is a symlink
# to the shipped SDK. Transport/platform only are local (127.0.0.1,
# temp cache dir). The pagination/cache/claim methods are official.
name: quantus_sdk
description: Official WormholeUtxoService against a local Hasura-SQL indexer.
publish_to: none
environment:
  sdk: ">=3.8.0 <4.0.0"
dependencies:
  convert: ^3.1.2
  crypto: ^3.0.6
  http: ^1.6.0
  ss58: ^1.3.1
  flutter:
    path: ../stubs/flutter
  path_provider:
    path: ../stubs/path_provider
```

`poc/F-3041/sdk/bin/poc.dart`

```dart
/// F-3041 — official WormholeUtxoService against local Hasura-SQL Postgres.
///
/// The pagination/cache/claim methods are the shipped
/// `wormhole_utxo_service.dart` (symlink). Local pieces are only:
///   * 127.0.0.1 GraphQL/RPC (Hardhat-style fork transport)
///   * temp-dir cache path
///
///   cd poc/F-3041/sdk && dart pub get && dart run bin/poc.dart

import 'dart:convert';
import 'dart:io';

import 'package:http/http.dart' as http;
import 'package:quantus_sdk/src/services/wormhole_utxo_service.dart';

const dest = 'qzpmV14n66LkzgdHyDo2WvfxY5K2sZGcF1T6upuKiiNv2sHcA';
const missId = '0000620674-d7dfe-000011';
const dupId = '0000620674-d7dfe-000003';
const missAmount = 2590000000000;
const unit = 1000000000000;
const localGraphql = 'http://127.0.0.1:4350/v1/graphql';

Never die(String msg) {
  stderr.writeln('FAIL  $msg');
  exit(1);
}

String unitStr(BigInt v) {
  final n = v.toInt();
  return '${n ~/ unit}.${(n % unit).toString().padLeft(12, '0')} QUAN';
}

String officialUtxoPath() {
  final env = Platform.environment['F3041_OFFICIAL'];
  if (env != null && File(env).existsSync()) return File(env).resolveSymbolicLinksSync();
  final linked = File('${Directory.current.path}/lib/src/services/wormhole_utxo_service.dart');
  if (linked.existsSync()) return linked.resolveSymbolicLinksSync();
  die('official wormhole_utxo_service.dart is not linked');
}

void assertOfficialFile(String path) {
  final text = File(path).readAsStringSync();
  if (!text.contains('static const int _transferPageSize = 300')) {
    die('official _transferPageSize is no longer 300');
  }
  if (!text.contains('static const int _reorgDepth = 180')) {
    die('official _reorgDepth is no longer 180');
  }
  if (!text.contains('order_by: {block: {height: asc}}')) {
    die('official query is no longer height-only');
  }
  if (text.contains('order_by: [{block: {height: asc}}, {id:')) {
    die('official query already has a unique id tie-break');
  }
  final send = path.replaceAll('wormhole_utxo_service.dart', 'wormhole_send_service.dart');
  final claim = File(send).readAsStringSync();
  final start = claim.indexOf('Future<ClaimResult> _runClaimFlow');
  if (start < 0) die('cannot find _runClaimFlow');
  final chunk = claim.substring(start, start + 1800);
  if (!chunk.contains('getUnspentTransfers')) {
    die('_runClaimFlow no longer calls getUnspentTransfers');
  }
  if (chunk.contains('forceReload') || chunk.contains('clearCachesForAddresses')) {
    die('_runClaimFlow now clears the UTXO cache');
  }
}

/// Ground-truth walk. Not used by the wallet. Same SQL engine, unique order.
Future<List<Map<String, dynamic>>> uniqueWalk() async {
  const q = r'''
query TransfersToAddresses($tos: [String!]!, $limit: Int!, $offset: Int!, $afterBlock: Int) {
  transfers: transfer(
    where: { to: { id: {_in: $tos } }, block: { height: {_gt: $afterBlock } } }
    order_by: [{block: {height: asc}}, {id: asc}]
    limit: $limit
    offset: $offset
  ) { id amount leafIndex transferCount block { height } from { id } to { id } }
}''';
  final all = <Map<String, dynamic>>[];
  var offset = 0;
  while (true) {
    final res = await http.post(
      Uri.parse(localGraphql),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        'query': q,
        'variables': {'tos': [dest], 'limit': 300, 'offset': offset, 'afterBlock': 0},
      }),
    );
    if (res.statusCode != 200) die('unique graphql HTTP ${res.statusCode}');
    final rows = (((jsonDecode(res.body) as Map)['data'] as Map)['transfers'] as List).cast<Map<String, dynamic>>();
    all.addAll(rows);
    if (rows.length < 300) break;
    offset += 300;
  }
  return all;
}

void main() async {
  print('F-3041  official WormholeUtxoService + local Hasura-SQL Postgres');
  print('        getTransfersToMany / getUnspentTransfers / clearCaches');
  print('        no public pin · no Python sort · no planted skip\n');

  final official = officialUtxoPath();
  print('========== STEP 0  official service file ==========');
  print('  $official');
  assertOfficialFile(official);
  print('  _transferPageSize = 300');
  print('  _reorgDepth       = 180');
  print('  order_by          = {block: {height: asc}}');
  print('  claimRewards      = getUnspentTransfers, no cache clear');

  final cacheDir = Directory('${Directory.systemTemp.path}/f3041_wormhole_cache');
  if (cacheDir.existsSync()) cacheDir.deleteSync(recursive: true);

  final svc = WormholeUtxoService();

  print('\n========== STEP 1  official getTransfersToMany (empty cache) ==========');
  await WormholeUtxoService.clearCachesForAddresses([dest]);
  final first = await svc.getTransfersToMany([dest]);
  final firstRows = first.byAddress[dest]!;
  final firstIds = [for (final t in firstRows) t.id];
  final firstSet = firstIds.toSet();
  final dups = <String>[];
  final seen = <String>{};
  for (final id in firstIds) {
    if (!seen.add(id)) dups.add(id);
  }
  print('  fetched           ${firstIds.length}');
  print('  unique            ${firstSet.length}');
  print('  dups              $dups');
  print('  miss $missId  ${firstSet.contains(missId)}');
  print('  safeCutoff        ${first.safeCutoff}');
  if (firstIds.length != 2439) die('expected 2439 rows, got ${firstIds.length}');
  if (firstSet.length != 2438) die('expected 2438 unique, got ${firstSet.length}');
  if (!dups.contains(dupId)) die('expected DUP $dupId, got $dups');
  if (firstSet.contains(missId)) die('official walk included the miss');
  if (first.safeCutoff <= 620674) die('miss height is not finalized vs safeCutoff ${first.safeCutoff}');

  print('\n========== STEP 2  unique order_by on the same Postgres ==========');
  final unique = await uniqueWalk();
  final uIds = [for (final r in unique) r['id'] as String];
  print('  fetched           ${uIds.length}');
  print('  unique            ${uIds.toSet().length}');
  print('  miss present      ${uIds.contains(missId)}');
  if (uIds.length != 2439 || uIds.toSet().length != 2439) die('unique walk is not complete');
  if (!uIds.contains(missId)) die('unique walk lost the miss');
  final missRow = unique.firstWhere((r) => r['id'] == missId);
  final missAmt = BigInt.parse(missRow['amount'] as String);
  print('  missed row        height=${(missRow['block'] as Map)['height']} amount=$missAmt');
  print('  missed value      ${unitStr(missAmt)}');
  if (missAmt != BigInt.from(missAmount)) die('miss amount $missAmt');

  print('\n========== STEP 3  official getTransfersToMany again (cache + height_gt) ==========');
  final cached = await svc.getTransfersToMany([dest]);
  final cachedIds = {for (final t in cached.byAddress[dest]!) t.id};
  print('  rows              ${cached.byAddress[dest]!.length}');
  print('  miss recovered?   ${cachedIds.contains(missId)}');
  if (cachedIds.contains(missId)) die('cached incremental walk recovered the miss');

  print('\n========== STEP 4  official clearCachesForAddresses + walk (forceReload) ==========');
  await WormholeUtxoService.clearCachesForAddresses([dest]);
  final reloaded = await svc.getTransfersToMany([dest]);
  final reloadIds = {for (final t in reloaded.byAddress[dest]!) t.id};
  print('  unique            ${reloadIds.length}');
  print('  miss after reload ${reloadIds.contains(missId)}');
  if (reloadIds.contains(missId)) die('forceReload healed the miss');

  print('\n========== STEP 5  official getUnspentTransfers (claimRewards entry) ==========');
  await WormholeUtxoService.clearCachesForAddresses([dest]);
  final unspent = await svc.getUnspentTransfers(
    wormholeAddress: dest,
    secretHex: '00' * 32,
  );
  final spendIds = {for (final t in unspent) t.id};
  final spendable = unspent.fold<BigInt>(BigInt.zero, (s, t) => s + t.amount);
  final truth = unique.fold<BigInt>(BigInt.zero, (s, r) => s + BigInt.parse(r['amount'] as String));
  print('  official UTXOs    ${unspent.length}');
  print('  unique-order rows ${unique.length}');
  print('  official spendable ${unitStr(spendable)}');
  print('  unique-order total ${unitStr(truth)}');
  print('  frozen            ${unitStr(truth - spendable)}');
  if (spendIds.contains(missId)) die('miss leaked into getUnspentTransfers');
  if (truth - spendable != BigInt.from(missAmount)) {
    die('frozen ${truth - spendable} != $missAmount');
  }

  print('\n========== RESULT ==========');
  print('  Official WormholeUtxoService.getTransfersToMany ran the shipped');
  print('  height-only OFFSET 300 query against local Postgres LATERAL SQL.');
  print('  Same class as the public pin: DUP $dupId, MISS $missId (2.59 QUAN).');
  print('  Unique {height,id} has the row. Cache, forceReload, and');
  print('  getUnspentTransfers (claimRewards) do not present it.');
  print('  Funds at risk this run: 2.59 QUAN. No token price in pre-deployment.');
  print('F-3041 E2E PASS');
}
```

Local transport / platform only (not the walk):

`poc/F-3041/sdk/lib/src/services/network/redundant_endpoint.dart`

```dart
import 'package:http/http.dart' as http;

/// Local-fork transport. Official WormholeUtxoService constructs these
/// and calls `post`. Endpoints are 127.0.0.1 only.
class GraphQlEndpointService {
  static final GraphQlEndpointService _instance = GraphQlEndpointService._();
  factory GraphQlEndpointService() => _instance;
  GraphQlEndpointService._();

  Future<http.Response> post({String? path, Map<String, String>? headers, String? body}) {
    return http.post(
      Uri.parse('http://127.0.0.1:4350/v1/graphql'),
      headers: {'Content-Type': 'application/json', ...?headers},
      body: body,
    );
  }
}

class RpcEndpointService {
  static final RpcEndpointService _instance = RpcEndpointService._();
  factory RpcEndpointService() => _instance;
  RpcEndpointService._();

  Future<http.Response> post({String? path, Map<String, String>? headers, String? body}) {
    return http.post(
      Uri.parse('http://127.0.0.1:19944/'),
      headers: {'Content-Type': 'application/json', ...?headers},
      body: body,
    );
  }
}
```

`poc/F-3041/sdk/lib/src/services/substrate_service.dart`

```dart
import 'dart:typed_data';

import 'package:ss58/ss58.dart';

Uint8List getAccountId32(String ss58Address) {
  return Uint8List.fromList(Address.decode(ss58Address).pubkey);
}
```

`poc/F-3041/sdk/lib/src/services/hd_wallet_service.dart`

```dart
import 'package:convert/convert.dart';
import 'package:quantus_sdk/src/rust/api/wormhole.dart' as wormhole_ffi;

class HdWalletService {
  String computeNullifier({required String secretHex, required BigInt transferCount}) {
    final secretBytes = hex.decode(secretHex.replaceFirst('0x', ''));
    final nullifierBytes = wormhole_ffi.computeNullifier(secret: secretBytes, transferCount: transferCount);
    return '0x${hex.encode(nullifierBytes)}';
  }
}
```

`poc/F-3041/sdk/lib/src/rust/api/wormhole.dart`

```dart
import 'dart:typed_data';

import 'package:crypto/crypto.dart';

/// Cache-key / nullifier preimage for the official UTXO service.
/// Pagination does not depend on these. Spent-check is empty on the local indexer.
String computeAddressHashHex({required List<int> rawAddress}) => sha256.convert(rawAddress).toString();

Uint8List computeNullifier({required List<int> secret, required BigInt transferCount}) {
  return Uint8List.fromList(sha256.convert([...secret, ...transferCount.toRadixString(16).codeUnits]).bytes);
}
```

`poc/F-3041/sdk/lib/src/utils/print.dart`

```dart
void quantusPrint(String message) {
  print(message);
}
```

`poc/F-3041/stubs/path_provider/lib/path_provider.dart`

```dart
import 'dart:io';

Future<Directory> getApplicationSupportDirectory() async {
  final dir = Directory('${Directory.systemTemp.path}/f3041_wormhole_cache');
  await dir.create(recursive: true);
  return dir;
}
```

`poc/F-3041/stubs/flutter/lib/foundation.dart`

```dart
const visibleForTesting = Object();
const kDebugMode = true;
```
