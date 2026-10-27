# 91450 w a low unbounded links rendering blocks manager navigation for 14 seconds on a gifted name

**Submitted on Sep 1st 2026 at 14:48:21 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91450
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
    * Locking up the victim from login
    * Cookie bombing, etc.

## Description

## Summary

Manager accepts the resolver-controlled `links` text record as an unbounded JSON array. It parses and validates every element, eagerly mounts one `LinkPreview` for every accepted item, and synchronously creates and percent-encodes a 64-rectangle SVG pattern for every card.

An ordinary attacker can register an ENSv2 name, configure its real `PermissionedResolver`, advertise the normal `links` key to Panoptes, make that resolver return 60,000 schema-valid links, and transfer the name to a victim. The transfer requires no action by the code-empty, zero-nonce victim. The gifted name appears through Manager's normal address dashboard. One ordinary click on that name makes the unmodified Manager retrieve a 1,620,001-byte record and eagerly mount all 60,000 cards.

In the final clean-clone run on native CPU, the attack route produced a 13.457-second main-thread task and a 13.848-second heartbeat stall. A renderer evaluation issued only after the complete RPC response had arrived took 14.546 seconds, and Manager's real global **Open navigation** button took 14.659 seconds to respond. An equal-size rejected-value control completed the same actions in 79 milliseconds, a canonical three-link control completed them in 81 milliseconds, and a sibling Manager page remained responsive in 257 milliseconds. The victim's active Manager tab is therefore temporarily inaccessible because of Manager's unbounded synchronous work, not RPC latency, the indexer, or general service failure.

## Finding Description

### Intended behavior

Profile records are untrusted resolver-controlled input. Reading one profile must impose aggregate byte, item, and rendering-work bounds so that a crafted record cannot prevent the visitor from using Manager's global navigation.

### Actual behavior

`getProfileRecords()` merges every text key advertised by the indexer into the ENSJS request, including `links`:

```ts
const texts = unique([
  ...staticTextRecords,
  ...textRecords.map((record) => record.key),
  ...forceFetchRecords.always,
  ...forceFetchRecords.whenNotIndexed,
  ...indexerRecords.texts,
])

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

The `links` transformer parses the complete value and validates every array element without a raw-byte or item-count limit:

```ts
const parseLinksJson = (rawValue: string): LinkItem[] => {
  let raw: unknown
  try {
    raw = JSON.parse(rawValue)
  } catch {
    return []
  }
  if (!Array.isArray(raw)) return []
  const out: LinkItem[] = []
  for (const item of raw) {
    const parsed = v.safeParse(LinkItemSchema, item)
    if (parsed.success) out.push(parsed.output)
  }
  return out
}
```

The public profile normalizes the whole accepted array and eagerly mounts every card:

```tsx
export const getSafeProfileLinks = (
  records: ProfileRecords,
): SafeProfileLink[] =>
  records.links.flatMap((link) => {
    const href = toSafeHttpHref(link.url)
    return href ? [{ ...link, href, displayHost: getDisplayHost(href) }] : []
  })

{links.map((link) => (
  <LinkPreview key={`${link.name}-${link.href}`} link={link} />
))}
```

Each `LinkPreview` calls `getGeneratedLinkPattern()`. That helper constructs an 8-by-8 rectangle array, joins the complete SVG, and percent-encodes it into a data URI:

```ts
const createLinkPatternRects = (...) =>
  Array.from(
    { length: LINK_PATTERN_COLUMNS * LINK_PATTERN_COLUMNS },
    (_, i) => {
      // construct one rectangle
    },
  )

const svg = createLinkPatternSvg(patternId, palette, variant)
return {
  backgroundImage: `url("${toSvgDataUri(svg)}")`,
  // ...
}
```

For 60,000 valid links this performs approximately 3.84 million rectangle constructions in addition to JSON parsing, schema validation, URL parsing, React reconciliation, CSS processing, DOM creation, and layout.

The complete production path exercised by the PoC is:

```
attacker registers ENSv2 name and configures PermissionedResolver
  -> attacker emits the ordinary `links` key-discovery event
  -> attacker transfers the ERC-1155 name to the victim
  -> Manager address dashboard discovers the victim's gifted name
  -> victim clicks the ordinary name row
  -> profileRecordsQuery(name)
  -> getProfileRecords(name)
  -> Panoptes supplies the real resolver and indexed `links` key
  -> ENSJS and Universal Resolver read text(node, "links")
  -> resolver returns 60,000 valid link objects
  -> transformProfileRecords() parses every object
  -> ProfileLinksSection mounts 60,000 LinkPreview components
  -> each component generates and encodes a 64-rectangle SVG
  -> the active Manager renderer cannot process global navigation
```

## Root Cause

The in-scope Manager places no aggregate byte, count, DOM, or synchronous-work bound on a resolver-controlled `links` record before parsing and eager rendering. `ProfileLinksSection` amplifies every accepted item through per-card SVG generation. The resolver and Panoptes deliver valid attacker input through the intended profile-read path; the security defect is Manager's unbounded first-party consumption of that input.

## Impact Explanation

The final clean-clone native-CPU run measured:

```
links JSON value:                         1,620,001 bytes
exact Manager RPC response:              3,266,918 bytes
accepted links/cards:                           60,000
gifted-name click handler:                    9.267 ms
gifted-route renderer evaluation:          13,991.623 ms
gifted-route global navigation action:     14,188.680 ms
repeat-route renderer evaluation:          14,545.801 ms
repeat-route global navigation action:     14,658.751 ms
repeat-route maximum heartbeat stall:      13,847.800 ms
repeat-route maximum main-thread task:     13,457.000 ms
repeat-route DOM nodes:                         708,174
repeat-route V8 used heap:                 656,174,668 B
repeat-route embedder heap:              1,895,216,512 B
```

The action probes begin after the decisive 3.27 MB RPC response has completed. Manager still cannot execute the probes or its own global navigation control for approximately 14 seconds. This is a measured temporary loss of access to the active target application tab.

The controls isolate the vulnerable first-party work:

```
equal-size malformed JSON response:        3,266,918 bytes
equal-size control accepted cards:                  0
equal-size control evaluation/menu:        79.218 / 79.449 ms
canonical control accepted cards:                   3
canonical control evaluation/menu:         80.510 / 80.691 ms
sibling Manager page load:                         257.487 ms
RPC and Panoptes health probes:                         PASS
```

The equal-size response travels through the same Manager, RPC, Universal Resolver, and Panoptes path but does not create cards. Its 79-millisecond result demonstrates that response size alone does not cause the denial. The canonical control proves normal profile-link rendering remains responsive.

## Likelihood Explanation

The attacker needs one ordinary ENSv2 name and control of the resolver assigned to that name. The PoC uses normal registration, resolver deployment and assignment, `setText`, authorized UUPS upgrade, and ERC-1155 transfer calls. Every state transaction consumes less than the pinned 60,000,000 block gas limit.

The victim does not connect a wallet, sign a transaction, approve the attacker, or select a special unsafe composition. The attacker transfers the name to the victim without participation, after which the victim performs a normal Manager workflow: opening a gifted name displayed on the address dashboard.

The frozen Manager sends the decisive read without a `gas` field. The exact full output succeeds at a 300,000,000 read ceiling. The PoC also includes a bounded read-only capability receipt from the exact dRPC route configured by the frozen Manager: an omitted-gas call returned a 4,185,702-byte HTTP body, 918,784 bytes larger than the exploit response. The attack therefore does not depend on increasing the chain's 60,000,000 transaction block limit.

## Attack Path / Reproduction Path

{% stepper %}
{% step %}

### Register an ordinary ENSv2 name

The attacker registers an ordinary ENSv2 name through the deployed registrar.
{% endstep %}

{% step %}

### Deploy and assign a resolver

The attacker deploys and assigns a dedicated real `PermissionedResolver` proxy.
{% endstep %}

{% step %}

### Advertise the `links` key

The attacker writes `links=[]`, emitting the ordinary event used by Panoptes to inventory that key.
{% endstep %}

{% step %}

### Install the response implementation

Using the resolver owner's normal upgrade authority, the attacker installs the response implementation that returns 60,000 valid `{"name":"a","url":"http:a"}` objects.
{% endstep %}

{% step %}

### Transfer the ENSv2 name

The attacker transfers the real ENSv2 ERC-1155 name to a separate victim EOA.
{% endstep %}

{% step %}

### Discover the gifted name

Manager's address dashboard discovers the unsolicited gifted name through its normal production query.
{% endstep %}

{% step %}

### Open the name

The victim clicks the normal name row.
{% endstep %}

{% step %}

### Request the advertised value

Manager requests the advertised `links` value through ENSJS, Multicall3, Universal Resolver, and the assigned resolver.
{% endstep %}

{% step %}

### Render all links

Manager receives the complete 1,620,001-byte value, parses 60,000 valid links, eagerly mounts 60,000 cards, and creates 60,000 SVG patterns.
{% endstep %}

{% step %}

### Observe blocked navigation

After the response has completed, the active Manager renderer cannot process evaluation or its global navigation action for approximately 14 seconds.
{% endstep %}
{% endstepper %}

## External Preconditions

* The victim visits the affected frozen Manager application.
* The attacker knows the victim's public address and controls one ENSv2 name and its assigned resolver.
* The resolver's ordinary `links` key has been indexed.
* Manager's configured read provider supports the response; the included receipt proves a larger omitted-gas response on that exact route.

## Internal Preconditions

* `getProfileRecords()` includes the indexed `links` key in the profile request.
* The resolver returns the 1,620,001-byte array through the normal Universal Resolver path.
* `transformProfileRecords()` accepts all 60,000 schema-valid objects.
* `ProfileLinksSection` eagerly renders the entire array.
* `LinkPreview` synchronously generates the SVG pattern for every card.

## Affected Code

All links use frozen commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

* [`profileRecords.ts#L82-L153`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileRecords.ts#L82-L153) accepts the indexer's text-key inventory and requests every value without a response bound.
* [`transformRecords.ts#L42-L65`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/utils/transformRecords.ts#L42-L65) parses the full `links` array and validates every element without a byte or count limit.
* [`transformRecords.ts#L156-L161`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/utils/transformRecords.ts#L156-L161) appends every accepted link to profile state.
* [`ProfileView.helpers.ts#L114-L140`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileView.helpers.ts#L114-L140) normalizes and reparses every accepted URL.
* [`ProfileLinksSection.tsx#L24-L83`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileLinksSection.tsx#L24-L83) eagerly maps the complete array and generates a pattern for every preview.
* [`ProfileLinks.helpers.ts#L208-L266`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileLinks.helpers.ts#L208-L266) constructs and percent-encodes 64 SVG rectangles for every link.

## Recommended Mitigation

Reject oversized `links` values before `JSON.parse`, enforce a small maximum number of accepted items before per-item schema and URL processing, and render only a bounded or virtualized subset. Avoid generating one complete SVG data URI per item during the initial synchronous render. Apply the byte and count bounds at the service/transform boundary so every current and future consumer receives a bounded structure.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/f8234f383386b20a75c811136ed31bbf/e992097e5367841bf8a3bca288535098dd549a5c>

## Proof of Concept

### Description

The PoC executes the complete attack and victim workflow against the frozen unmodified application. It uses the repository's real ENSv2 registration fixture and deployed Sepolia contracts to register three names: the attack, equal-size rejected-value control, and canonical three-link control. For every name it deploys a dedicated real PermissionedResolver, emits the `links` key-discovery event, performs the authorized upgrade, transfers the real ERC-1155 name to a code-empty zero-nonce victim, waits for the real Panoptes row, and verifies the response through Universal Resolver and the Manager-shaped ENSJS request.

The browser stage builds the unmodified optimized Manager and uses native-CPU Chromium. It starts from the victim address dashboard, clicks the real gifted name row, captures the exact payload-bearing `eth_call`, waits for the response, then measures a renderer command and the real global navigation button. It repeats the attack in a fresh browser and runs both controls. Assertions cover:

* frozen commit, fork block/hash, and deployed contract bindings;
* real registration, resolver assignment, key event, proxy implementation, and ERC-1155 transfer;
* attacker balance `1 -> 0`, victim balance `0 -> 1`, and unchanged victim nonce;
* 60,000 accepted attack cards, zero equal-size control cards, and three canonical cards;
* the browser request's exact name packet, Universal Resolver target, omitted `gas`, and complete response;
* successful full output at a 300,000,000 read ceiling;
* 60,000,000-gas local block headers and every state transaction below that limit;
* post-response native action delays of at least ten seconds for both attack runs;
* sub-two-second equal-size and canonical controls; and
* responsive sibling Manager, RPC, and Panoptes controls.

`rpc-ceiling-proxy.mjs` does not create or replace RPC output. It adds the configured provider's 600,000,000 read ceiling only to omitted-gas `eth_call` and `eth_estimateGas` requests before forwarding them to the real forked EVM. The browser receipt preserves the original omitted-gas request and separately proves the exact complete output at 300,000,000 gas. The included `provider-capability-receipt.json` records a one-time read-only observation that the exact frozen Manager route accepts a larger omitted-gas response; the normal PoC run does not contact that route for this probe.

There are no target source patches, target storage writes, fabricated indexer rows, mocked RPC results, browser fixture arrays, component-only substitutes, CPU throttling, or exploit transactions against public ENS contracts.

Tested tool versions:

```
Node.js:          22.23.2
pnpm:             10.27.0
Playwright:       1.58.2 with bundled Chromium
tsx:              4.21.0
Foundry:          1.7.1 (4072e48705af9d93e3c0f6e29e93b5e9a40caed8)
solc:             0.8.30
Docker:           29.6.1
Docker Compose:   5.1.4
```

### Where to place files and their names

Clone the frozen target, install its locked dependencies, clone the immutable PoC revision under `poc/`, and restore the Solidity path flattened by Gist:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git audit-comp-ens
cd audit-comp-ens
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
pnpm --dir e2e exec playwright install chromium

mkdir -p poc
git clone https://gist.github.com/f8234f383386b20a75c811136ed31bbf.git \
  poc/manager-profile-links-client-denial
git -C poc/manager-profile-links-client-denial checkout --detach \
  e992097e5367841bf8a3bca288535098dd549a5c
mkdir -p poc/manager-profile-links-client-denial/src
mv poc/manager-profile-links-client-denial/AmplifyingResolver.sol \
  poc/manager-profile-links-client-denial/src/AmplifyingResolver.sol
cd poc/manager-profile-links-client-denial
shasum -a 256 -c SOURCE_SHA256SUMS
cd ../..
```

The complete authored PoC tree is:

```
poc/manager-profile-links-client-denial/
├── README.md
├── SOURCE_SHA256SUMS
├── docker-compose.poc.yml
├── fixture-time-proxy.mjs
├── foundry.toml
├── measure-browser.mts
├── package.json
├── panoptes-contracts.json
├── provider-capability-receipt.json
├── rpc-ceiling-proxy.mjs
├── run.sh
├── setup.mts
└── src/
    └── AmplifyingResolver.sol
```

`AmplifyingResolver.sol` generates the attack and both controls. `setup.mts` performs and verifies every chain/indexer/Universal-Resolver setup step. `measure-browser.mts` runs the gifted-name browser path, repeat, controls, and all action/health assertions. `run.sh` pins the target, starts the digest-pinned local fork and Panoptes, builds Manager, runs the exploit, writes the evidence, and removes only the disposable PoC services.

### Command to run

From the frozen repository root:

```bash
POC_LOG_DIR="$(mktemp -d /tmp/ens16-evidence.XXXXXX)" \
  bash poc/manager-profile-links-client-denial/run.sh
```

The command creates all attack state on the disposable local fork, starts the unmodified Manager, executes the browser scenarios, verifies every assertion, prints the final marker, removes the disposable containers/chain volume, and preserves the selected evidence directory.

The generated evidence files are:

```
delivery-state.json
gifted-discovery-attack.json
native-attack-repeat.json
equal-size-rejected-control.json
canonical-control.json
final-verification.json
setup.log
browser-discovery.log
browser-attack-repeat.log
browser-rejected-control.log
browser-canonical-control.log
rpc-ceiling-proxy.log
fixture-time-proxy.log
forge-build.log
manager-build.log
manager.log
compose-anvil.log
compose-panoptes.log
```

### Logs proving the bug

Fresh clean-clone output from the immutable Gist revision:

```
ENS16_PROFILE_LINKS_CLIENT_DENIAL_E2E_PASS
name ens16-links-attack-1788255631.eth
victim 0x81dc9046b61da595e3ae1eb6285a8a8c07638e11
resolver 0x88DE4c7B53071D3253CB18847cDbc751F7f58EE8
linksBytes 1620001
cards 60000
giftedNameClickMs 9.267207980155945
discoveryEvaluationDelayMs 13991.622542023659
discoveryMenuDelayMs 14188.680083036423
repeatEvaluationDelayMs 14545.801250040531
repeatMenuDelayMs 14658.751083016396
repeatHeartbeatStallMs 13847.800000011921
repeatLongTaskMs 13457
equalSizeControlEvaluationMs 79.21758300065994
equalSizeControlMenuMs 79.44904202222824
canonicalControlEvaluationMs 80.51033300161362
canonicalControlMenuMs 80.69062501192093
fullResponseGasCeiling 300000000
attackResponseBytes 3266918
equalSizeControlResponseBytes 3266918
siblingManagerLoadMs 257.4869999885559
```

The decisive repeat request and timing receipt additionally records:

```
rpcMethod:                       eth_call
target:                          0xca11bde05977b3631167028862be2a173976ca11
requestedGas:                    null
containsUniversalResolver:       true
containsExactNamePacket:         true
requestBytes:                    61192
callDataBytes:                   30532
responseFinishedMs:              407.7283329963684
commandIssuedAfterResponseMs:    529.4960839748383
responseBodyBytes:               3266918
renderedPanels:                  60000
latestHeaderGasLimit:            60000000
```

Evidence SHA-256 values from that run:

```
0d1594472685869cf7d69670470de763b52bf93cba87108d6c4416ec545382be  delivery-state.json
04faf484102893d0c51a70c8774f09fdde30892b6d193f147325edf2936d26c7  gifted-discovery-attack.json
c7a8369c2e4ea64db576f18559960cc51cf5c5a08d1afcd5091a106df6428f47  native-attack-repeat.json
ae9e755fbeeb2eda5b97873d96f37e7910015e0c01c6d1852f7823644d8aeb78  equal-size-rejected-control.json
fe370e91214878a982be5bd1d1ef1ed384825d8b4f986676d68bffbc15aefef8  canonical-control.json
7e37f2e124c1b47ca6e910f906d1434711db4681331e38b6de643114316548c7  final-verification.json
```

**Full source:** [**immutable ENS Manager profile-links client-denial PoC Gist**](https://gist.github.com/aydarkhusaenov/f8234f383386b20a75c811136ed31bbf/e992097e5367841bf8a3bca288535098dd549a5c)
