# 91444 w a low an unsolicited ensv2 name with an oversized coin 121 result blocks the victim s default manager dashboard

**Submitted on Sep 1st 2026 at 14:35:10 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91444
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
    * Locking up the victim from login
    * Cookie bombing, etc.

## Description

## Summary

An attacker can transfer one ENSv2 name with an attacker-controlled resolver to an arbitrary victim. When that victim connects normally and opens Manager's root route, Manager automatically redirects to `/dashboard`, places the unsolicited name on the first default owner-query page, and loads its complete profile without any name click or primary-name selection.

Manager adds its fixed coin inventory to that profile request even though Panoptes reports no address records. The attacker's resolver returns one ABI-valid 131,072-byte value for ZEN coin type `121`, beginning with the accepted `0x2089` prefix. Manager passes the complete value into synchronous quadratic Base58Check conversion without a raw-byte or work bound.

The production-browser PoC reproduced the result twice in fresh Chromium processes. After the exact gifted-name/coin-121 RPC response completed, both a renderer evaluation and Manager's real `Open navigation` action remained unavailable for 45.37 seconds in the first run and 43.89 seconds in the second. A canonical 22-byte control and an equal-size 131,072-byte invalid-prefix control completed the same dashboard actions in at most 71.36 milliseconds.

## Finding Description

### Intended behavior

Resolver results are untrusted. Manager should reject oversized or noncanonical coin-address values with bounded work before format-specific conversion. Merely receiving a name must not allow its resolver to make the victim's ordinary default dashboard unresponsive.

### Actual behavior

The default dashboard combines three operations:

1. `MyNamesList` starts `profileRecordsQuery(name)` for every name on its current page.
2. `profileRecords` merges Manager's fixed address list into the request even when the indexer advertises no address records.
3. The ZEN decoder checks only the first two bytes and synchronously Base58Check-encodes the entire resolver-returned byte array.

The relevant dashboard query is automatic:

```tsx
const pageProfileRecords = useQueries({
  queries: pageRows.map(({ item, metadata, name }) => ({
    ...profileRecordsQuery(name),
    enabled: item.kind === 'v2' && !metadata.isInGrace,
  })),
})
```

The profile service expands the request:

```ts
const coins = unique([
  ...alwaysProbeAddressRecords.map(Number),
  ...addressRecords.map((record) => record.coinType),
  ...indexerRecords.coins,
]).filter(isSupportedCoinType)
```

Manager's 149-entry source map produces 148 supported coin requests after coin `784` is filtered. Coin `121` remains included when the indexed coin collection is empty.

The pinned ZEN coder checks the prefix but not the length:

```ts
const prefix = source.slice(0, 2)

if (!validPrefixes.some((x) => equalBytes(x, prefix))) {
  throw new Error('Invalid prefix')
}

return base58CheckEncode(source)
```

The bounded resolver returns the bytes during the ordinary read:

```solidity
function addr(bytes32, uint256 coinType)
    external
    view
    returns (bytes memory result)
{
    if (coinType != 121) return abi.encodePacked(ETH_VALUE);

    uint256 length = payloadLength;
    result = new bytes(length);
    assembly {
        let cursor := add(result, 0x20)
        let end := add(cursor, length)
        for { } lt(cursor, end) { cursor := add(cursor, 0x20) } {
            mstore(cursor, not(0))
        }
    }
    result[0] = payloadPrefix[0];
    result[1] = payloadPrefix[1];
}
```

The 131,072-byte value is produced by a bounded `eth_call`; it is not stored on chain as a large transaction payload. In the fresh run, the direct resolver read estimated `461,217` gas and the exact Manager Multicall3/Universal Resolver request estimated `549,260` gas against the pinned block's `60,000,000` gas limit. Its complete JSON-RPC response was `316,966` bytes. The accepted-prefix value decodes into a 179,005-character ZEN string.

## Root Cause

Manager automatically performs a full fixed-inventory, multi-coin profile read for each visible dashboard name and sends arbitrary resolver bytes to synchronous format-specific decoding without a raw-byte, canonical-length, per-record-work, or aggregate-work limit.

For ZEN, a two-byte prefix is treated as sufficient validation before the entire value enters the quadratic radix conversion. The default dashboard therefore exposes the renderer to attacker-selected work from a name the victim did not request.

The encoder and radix helper correctly process the bytes they are given; the vulnerability is the in-scope Manager composition that automatically selects this unused record, accepts an unbounded noncanonical result, and performs the conversion on the renderer thread before the default dashboard can be used.

## Impact Explanation

The clean production-browser results were:

| Scenario                          | Coin-121 bytes |   Prefix | RPC response body | Evaluation unavailable after response | `Open navigation` unavailable after response | Heartbeat stall |
| --------------------------------- | -------------: | -------: | ----------------: | ------------------------------------: | -------------------------------------------: | --------------: |
| Attack, fresh process 1           |        131,072 | `0x2089` |         316,966 B |                         45,372.933 ms |                                45,370.055 ms |   45,298.600 ms |
| Attack, fresh process 2           |        131,072 | `0x2089` |         316,966 B |                         43,881.848 ms |                                43,887.118 ms |   43,809.700 ms |
| Canonical control                 |             22 | `0x2089` |          54,886 B |                             60.571 ms |                                    46.756 ms |       51.200 ms |
| Equal-size invalid-prefix control |        131,072 | `0x0000` |         316,966 B |                             71.350 ms |                                    55.476 ms |       47.500 ms |

The equal-size control has exactly the same RPC response-body size as the attack. It follows the same registration, transfer, indexing, wallet connection, root-route redirect, dashboard query, Multicall3, Universal Resolver, and browser flow. It differs only in the two-byte prefix and therefore does not enter the expensive accepted-ZEN conversion.

During both attack stalls, a sibling Manager page loaded in 27 milliseconds or less. Local RPC and Panoptes health requests completed in 14 milliseconds or less. The denial was confined to the victim's affected Manager renderer, and the real Manager navigation action became available only after the decoder released it.

The gifted-name ownership and resolver assignment remain unchanged, so opening the same ordinary default dashboard starts the same record read and temporary denial again.

## Likelihood Explanation

The attacker needs one ordinary ENSv2 name, one bounded custom resolver, and one ERC-1155 transfer. The victim does not sign a transaction, grant approval, select the name as primary, click the gifted name, configure the resolver, or choose the queried coin inventory.

The PoC proves each victim is a fresh deterministic EOA with zero code, zero nonce, zero ETH, and no registry approval before delivery. Those properties remain unchanged after transfer and after all browser runs. Panoptes places the single gifted name at index zero of the victim's exact default first dashboard page.

The trigger is the ordinary connected flow: connect on `/?landing=true`, visit `/`, and allow Manager to redirect automatically to `/dashboard`.

## Attack Path / Reproduction Path

{% stepper %}
{% step %}

## The attacker registers one ENSv2 name.

{% endstep %}

{% step %}

## The attacker deploys a bounded resolver returning 131,072 bytes beginning `0x2089` for `addr(node, 121)`.

{% endstep %}

{% step %}

## The attacker assigns that resolver through the real ENSv2 Registry.

{% endstep %}

{% step %}

## The attacker transfers the ENSv2 ERC-1155 name to the victim.

{% endstep %}

{% step %}

## The victim sends no transaction and grants no approval; the configured resolver remains assigned.

{% endstep %}

{% step %}

## Panoptes truthfully indexes the victim as owner and the attacker resolver as current, with no indexed address records.

{% endstep %}

{% step %}

## The victim connects normally on Manager's landing route.

{% endstep %}

{% step %}

## The victim visits `/`; Manager automatically redirects to `/dashboard`.

{% endstep %}

{% step %}

## The gifted name is on the first default owner-query page. The victim does not click it and has no primary name.

{% endstep %}

{% step %}

## `MyNamesList` automatically starts the complete profile request for the gifted name.

{% endstep %}

{% step %}

## Manager's fixed inventory includes coin type `121` in the 148-coin request.

{% endstep %}

{% step %}

## The real Multicall3 and Universal Resolver path returns the 131,072-byte accepted-prefix result.

{% endstep %}

{% step %}

## Manager synchronously converts it through the pinned ZEN/Base58 path.

{% endstep %}

{% step %}

## Renderer evaluation and the real `Open navigation` button remain unavailable for more than 43 seconds after the response is complete.

{% endstep %}

{% step %}

## After conversion finishes, the navigation menu opens and the ordinary `Privacy Policy` link successfully reaches `/legal/privacy-policy`.

{% endstep %}
{% endstepper %}

## External Preconditions

* Manager runs the affected source at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` or equivalent code.
* The attacker can register, configure, and transfer one ENSv2 name.
* The victim connects to Manager and opens the ordinary root route.

## Internal Preconditions

* The unsolicited name is on the first default dashboard owner-query page.
* Its custom resolver remains assigned after transfer.
* `MyNamesList` automatically starts `profileRecordsQuery` for the name.
* Manager's fixed address inventory includes coin type `121` despite an empty indexed address collection.
* The returned value begins with the accepted ZEN prefix and reaches the synchronous decoder.

## Affected Code

All repository links use frozen commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

* [`MyNamesList.tsx#L255-L263`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/dashboard/components/MyNamesList.tsx#L255-L263) automatically starts a complete profile request for every name on the dashboard page.
* [`nameRowProfileRecords.ts#L18-L32`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/dashboard/components/nameRowProfileRecords.ts#L18-L32) reduces the completed profile to the lightweight row preview.
* [`address.tsx#L1-L49`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/data/records/address.tsx#L1-L49) defines the fixed address-record inventory.
* [`profileRecords.ts#L93-L139`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileRecords.ts#L93-L139) merges the fixed coin inventory and accepts decoded results without a work bound.
* [`profileRecords.ts#L165-L176`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileRecords.ts#L165-L176) exposes the automatic query used by the dashboard.
* [`DesktopNavigation.tsx#L45-L58`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/navigation/Header/navigation/DesktopNavigation.tsx#L45-L58) implements the real `Open navigation` action measured by the PoC.
* [`address-encoder v1.1.1 zen.ts#L16-L21`](https://github.com/ensdomains/address-encoder/blob/v1.1.1/src/coin/zen.ts#L16-L21) performs the prefix-only check before Base58Check conversion.
* [`ENSJS getAddress.ts#L169-L186`](https://github.com/ensdomains/ensjs/blob/e96662c22a73e6bc2e4eb23b66785defab4feb8c/packages/ensjs/src/utils/coders/getAddress.ts#L169-L186) converts returned hex into bytes and invokes the selected coder.
* [`@scure/base 1.2.6 index.ts#L178-L181`](https://github.com/paulmillr/scure-base/blob/1.2.6/index.ts#L178-L181) contains the quadratic radix conversion used by the pinned dependency graph.

## Recommended Mitigation

Before invoking a coin-specific decoder, enforce a strict raw-byte maximum and the canonical byte lengths accepted by that coin. Reject oversized values before allocating strings or entering radix conversion.

Also avoid loading the full fixed coin inventory for a dashboard row that consumes only preview fields. Request only the bounded fields required by the row, and move any remaining untrusted format conversion off the renderer thread or enforce a per-record and per-page work budget.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/030846e15ce9a1e857f21383065e9d00/b55632d151d9827e8e9e7d5e9b88ec6ea446522f>

## Proof of Concept

### Description

This full E2E PoC runs the frozen production Manager, deployed ENSv2 contracts on a fixed Sepolia fork, digest-pinned Panoptes, and real Chromium. It registers one attack name and two controls, assigns each a bounded resolver through the real Registry, and transfers each ERC-1155 name to a distinct fresh victim EOA. The victims send no transaction, grant no approval, select no primary name, and never click the gifted name.

Manager's ordinary connected root route redirects to `/dashboard`, where the unmodified first-page name-row flow automatically performs the exact 26-text/148-coin profile request. The attack resolver returns 131,072 bytes beginning with accepted ZEN prefix `0x2089`; the canonical control returns 22 bytes with the same prefix; and the equal-size control returns 131,072 bytes beginning `0x0000`. The browser binds the completed RPC response to the exact gifted name, resolver, and coin-121 result, then measures a renderer evaluation and Manager's real `Open navigation` action. The attack is repeated in two fresh browsers; both controls use the same complete dashboard flow.

The command fails unless both attack runs block both actions and the heartbeat for at least 30 seconds after the response completes, both controls remain below two seconds, the attack and equal-size control response bodies match in size, a sibling Manager page plus RPC and Panoptes stay responsive, and final chain/indexer reads prove unchanged victim nonce, balance, approval, ownership, resolver, and dashboard position. It preserves `delivery-state.json`, four browser receipts, `final-chain-indexer-invariants.json`, the corresponding logs, and build/service diagnostics in `POC_LOG_DIR`. Cleanup removes only the named disposable local services and chain volume. No Manager code, component, RPC response, GraphQL row, wallet state, or victim transaction is mocked. All state-changing actions and exploit-bearing reads remain on the disposable local fork. The only public-service request is Manager's normal hard-coded, read-only V1 `getNamesForAddress` inventory query; the receipts prove that it contains no exploit payload and returns an empty domain list for each fresh victim.

### Where to place files and their names

The Gist clones these files under the frozen repository root:

```
poc/manager-oversized-zen-address-client-denial/
├── README.md
├── SOURCE_SHA256SUMS
├── docker-compose.poc.yml
├── foundry.toml
├── measure-browser.mts
├── package.json
├── panoptes-contracts.json
├── run.sh
├── setup.mts
├── verify-final.mts
└── src/
    └── ZenAmplifyingResolver.sol
```

### Command to run

Run with Node.js 22.23.2, pnpm 10.27.0, Docker Compose v5.1.4, Foundry Forge 1.7.1, and read-only access to the pinned Sepolia archive block and ENS V1 subgraph:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git audit-comp-ens
cd audit-comp-ens
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/030846e15ce9a1e857f21383065e9d00.git \
  poc/manager-oversized-zen-address-client-denial
git -C poc/manager-oversized-zen-address-client-denial checkout --detach \
  b55632d151d9827e8e9e7d5e9b88ec6ea446522f

mkdir -p poc/manager-oversized-zen-address-client-denial/src
mv poc/manager-oversized-zen-address-client-denial/src__ZenAmplifyingResolver.sol \
  poc/manager-oversized-zen-address-client-denial/src/ZenAmplifyingResolver.sol

test "$(git -C poc/manager-oversized-zen-address-client-denial rev-parse HEAD)" = \
  "b55632d151d9827e8e9e7d5e9b88ec6ea446522f"
cd poc/manager-oversized-zen-address-client-denial
shasum -a 256 -c SOURCE_SHA256SUMS
cd ../..

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
pnpm --filter e2e exec playwright install chromium

ens10_evidence_dir="$(mktemp -d /tmp/ens10-evidence.XXXXXX)"
POC_LOG_DIR="$ens10_evidence_dir" \
bash poc/manager-oversized-zen-address-client-denial/run.sh
```

### Logs proving the bug

```
ENS18_REAL_TRANSFER_DELIVERY_READY
ENS18_PRODUCTION_ATTACK_RENDER_DENIAL_PROVED
ENS18_PRODUCTION_ATTACK_RENDER_DENIAL_PROVED
ENS18_PRODUCTION_CANONICAL_CONTROL_PASS
ENS18_PRODUCTION_PREFIX_CONTROL_PASS
ENS18_FINAL_CHAIN_INDEXER_INVARIANTS_PASS
{
  "result": "ENS18_OVERSIZED_ZEN_CLIENT_DENIAL_E2E_PASS",
  "attacker": "0xD47815115B7b84ff6af26702E9D947bC040171Ce",
  "attackName": "ens18-zen-attack-1788242801.eth",
  "victimAddress": "0xD5A53300Ca4E89C81071C3030eC64FC5E8E6936A",
  "requestedTexts": 26,
  "requestedFixedCoins": 148,
  "excludedUnsupportedCoins": [784],
  "indexedAttackCoins": 0,
  "payloadBytes": 131072,
  "encodedZenCharacters": 179005,
  "managerShapeDecodeMs": 48295.892792,
  "attackRpcBytes": 316966,
  "invalidPrefixRpcBytes": 316966,
  "evaluationBlockedAfterResponseMs": 45372.93341600895,
  "menuOpenBlockedAfterResponseMs": 45370.05495798588,
  "repeatedEvaluationBlockedAfterResponseMs": 43881.847959041595,
  "repeatedMenuOpenBlockedAfterResponseMs": 43887.11829203367,
  "canonicalEvaluationBlockedAfterResponseMs": 60.57091701030731,
  "canonicalMenuOpenBlockedAfterResponseMs": 46.7559170126915,
  "invalidPrefixEvaluationBlockedAfterResponseMs": 71.35029101371765,
  "invalidPrefixMenuOpenBlockedAfterResponseMs": 55.476000010967255,
  "maxHeartbeatStallMs": 45298.59999996424,
  "siblingManagerLoadMs": 26.42895895242691,
  "upstreamHealth": [
    {"service": "rpc", "elapsedMs": 8.233167052268982},
    {"service": "indexer", "elapsedMs": 8.529749989509583}
  ]
}
ENS18_OVERSIZED_ZEN_CLIENT_DENIAL_E2E_PASS
```

**Full source:** [**immutable eleven-file PoC Gist**](https://gist.github.com/aydarkhusaenov/030846e15ce9a1e857f21383065e9d00/b55632d151d9827e8e9e7d5e9b88ec6ea446522f)
