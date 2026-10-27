# 91446 w a low quadratic primary contact deduplication temporarily locks the victim s manager tab after an unsolicited name transfer

**Submitted on Sep 1st 2026 at 14:40:44 UTC by @legat for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91446
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
    * Locking up the victim from login
    * Cookie bombing, etc.

## Description

## Summary

Manager always requests the fixed `domains.ens.primary-contacts` text record. Its parser retains every supported key, deduplicates the complete array with `validKeys.indexOf(key)` once per element, and applies the three-contact limit only after the scan.

An attacker can register and configure an ordinary ENS V2 name, then transfer the name token to an arbitrary victim. The transfer is unsolicited: the victim does not connect a wallet, approve the transfer, or send a transaction. The real Panoptes inventory makes Manager display the newly owned name on the victim's ordinary public address page. One normal click on that first-party name link invokes the vulnerable in-scope Manager profile parser.

The attack resolver returns a valid 2,070,001-byte JSON array containing 270,000 supported contact keys grouped as 90,000 `mail`, 90,000 `email`, then 90,000 `phone`. This forces exactly 24,300,270,000 equality comparisons before Manager renders only three cards.

In the final immutable-Gist clean run, the equal-size interleaved control finished 1.401 seconds after its complete response. The grouped attack blocked the renderer for 23.910 seconds after the same-size complete response, caused a 23.477-second heartbeat stall, and delayed Back for 23.381 seconds. A sibling Manager route loaded in 20 milliseconds while the attacked tab remained blocked.

## Finding Description

### Intended behavior

Manager defines a strict product limit of three primary contact methods:

```ts
export const maxPrimaryContactMethods = 3
```

Resolver text is attacker-controlled input. Manager should reject excessive raw input and stop normalization after collecting three distinct supported keys. Input ordering should not be able to turn three-card composition into billions of synchronous comparisons on the renderer thread.

### Actual behavior

The primary-contact key is part of Manager's static inventory, so Manager asks for it even when the indexer advertises no text keys:

```ts
export const staticTextRecords = [
  'name',
  'avatar',
  'header',
  'description',
  'url',
  'theme',
  'language',
  'primary-contact',
  'domains.ens.primary-contacts',
] as const
```

The profile transformation stores the complete returned value without a byte or element limit:

```ts
if (staticTextRecords.includes(key as StaticRecordKey)) {
  return {
    ...acc,
    base: { ...acc.base, [key]: value },
  }
}
```

The parser materializes the complete JSON array and normalizes it as follows:

```ts
const normalizePrimaryContactKeys = (
  keys: readonly string[],
): ContactMethodKey[] => {
  const validKeys = keys.filter(isContactMethodKey)

  return validKeys
    .filter((key, index) => validKeys.indexOf(key) === index)
    .slice(0, maxPrimaryContactMethods)
}
```

For the grouped attack value, the first `mail` is at index 0, the first `email` is at index 90,000, and the first `phone` is at index 180,000. The exact work is:

```
90,000 * (1 + 90,001 + 180,001) = 24,300,270,000 comparisons
```

The control contains the identical 270,000-string multiset and identical 2,070,001-byte value but interleaves `mail,email,phone`. Every `indexOf` then finds its first match in the first three positions. Both paths return the same 4,185,702-byte production JSON-RPC response and render these exact cards in the same order:

```
Mailing Address / 1 Main Street
Email Address   / victim@example.test
Phone Number    / +1 555 0100
```

The final run captured the exact unmodified browser `eth_call`. It omitted the gas field, targeted Multicall3, contained the exact Universal Resolver call, name packet, and primary-contact key, and returned the complete nested value. The PoC decoded the exact outer row and proved `allowFailure == true` and `success == true`; it did not infer success from Multicall3's permissive flag. Replaying the exact transaction with an explicit 600,000,000 read ceiling produced the same complete value hash, byte count, element count, and multiset.

Every locally mined block kept the real 60,000,000 Sepolia header. The harness audited all 19 state transactions; the largest declared and consumed gas was 604,774. The attack's direct full Universal Resolver read estimated 329,860,138 gas, below the configured provider's documented 600,000,000 `eth_call` ceiling. The local provider-compatible proxy applies that ceiling only to omitted-gas read calls and never changes a transaction or chain state.

### Exact first-party flow

```
Attacker registers an ordinary ENS V2 name
  -> assigns bounded immutable resolver
  -> transfers real Registry ERC-1155 token to victim
  -> victim balance becomes 1; victim nonce remains 0; resolver persists
       |
       v
Real Panoptes indexes victim owner and current resolver
       |
       v
Manager public address route displays the newly owned name
       |
       v
Victim clicks the displayed first-party name link
       |
       v
Manager getProfileRecords() requests the fixed primary-contact key
  -> complete 2,070,001-byte value returns successfully
  -> transformProfileRecords() retains it
  -> parsePrimaryContactKeys() parses 270,000 strings
  -> indexOf/filter performs 24,300,270,000 comparisons
  -> slice(0, 3) runs only after the complete scan
       |
       v
Active Manager renderer, heartbeat, and Back navigation remain blocked
```

This denial occurs entirely in the listed Manager asset before any separate contract action. It does not depend on a later transaction, user-chosen unsafe recipient, special timing window, loss of focus, absence of refresh, or failed reconnect.

## Root Cause

Manager does not enforce its three-contact policy, a raw-value limit, or a bounded work limit before normalizing untrusted primary-contact JSON. It uses an `Array.indexOf` search inside an `Array.filter` pass, making the work ordering-dependent and quadratic, then applies `.slice(0, 3)` only after every attacker-supplied element has been scanned.

## Impact Explanation

The exact selected impact is reproduced in the unmodified production Manager: the active target tab cannot process ordinary navigation while the synchronous primary-contact scan runs.

The final immutable-Gist clean run measured:

```
payload bytes:                             2,070,001
valid array elements:                         270,000
entries per supported key:                     90,000
exact grouped comparisons:             24,300,270,000
complete browser RPC response bytes:          4,185,702
equal-size control after response:             1,400.536 ms
grouped attack after response:                23,910.494 ms
attack heartbeat stall:                       23,477.400 ms
gifted-name click to rendered Contact:        35,052.858 ms
Back delay:                                   23,381.007 ms
sibling Manager route load:                       20.397 ms
```

The complete attack and control browser responses have identical byte size; the two rendered card arrays are exactly equal. RPC and Panoptes health checks also completed while the attacked renderer was unavailable. The equal-size control isolates the material delay to the grouped input's quadratic deduplication rather than resolver execution, network response size, JSON volume, route selection, or final DOM output.

## Likelihood Explanation

The attacker needs one ordinary ENS V2 name, one attacker-controlled resolver, and one normal ERC-1155 transfer. The large value is constructed by the resolver during `eth_call`; it is not stored in transaction calldata.

The attacker alone chooses the resolver and transfer recipient. The victim takes no action to receive the name. Manager truthfully discovers it as owned and presents it as a normal first-party link. The victim's only triggering action is one ordinary click on that displayed name. The persisted token ownership and resolver binding make the trigger repeatable without a narrow timing condition.

No wallet connection, signature, administrator, governance role, compromised indexer, compromised RPC, browser defect, target source patch, or mocked target response is required.

## Attack Path / Reproduction Path

{% stepper %}
{% step %}

### The attacker registers an ordinary ENS V2 name.

{% endstep %}

{% step %}

### The attacker assigns a resolver returning 270,000 supported primary-contact keys grouped as 90,000 `mail`, 90,000 `email`, and 90,000 `phone`.

{% endstep %}

{% step %}

### The attacker transfers the real Registry ERC-1155 name token to the victim.

{% endstep %}

{% step %}

### The victim sends no transaction.

Its name balance becomes one, its nonce remains zero, and the resolver remains assigned.
{% endstep %}

{% step %}

### Real Panoptes indexes the victim as owner and the attacker resolver as current.

{% endstep %}

{% step %}

### The victim opens the ordinary public Manager address page.

The page remains responsive and displays the unsolicited name as a normal link.
{% endstep %}

{% step %}

### The victim clicks that displayed name once.

{% endstep %}

{% step %}

### Manager's fixed record inventory retrieves the complete valid JSON through Multicall3 and the Universal Resolver.

{% endstep %}

{% step %}

### Manager parses all 270,000 strings and performs 24,300,270,000 synchronous equality comparisons.

{% endstep %}

{% step %}

### The target tab cannot service heartbeat or Back navigation until the scan ends.

Only then does Manager render the three permitted cards.
{% endstep %}
{% endstepper %}

## External Preconditions

* The affected frozen Manager code is served.
* The attacker can register, configure, and transfer one ordinary ENS V2 name.
* The victim address can receive the Registry ERC-1155 token.
* The victim opens its ordinary Manager address view and clicks the displayed owned name.
* Manager's configured read transport accepts the demonstrated 4,185,702-byte omitted-gas response within its documented 600,000,000 read ceiling.

## Internal Preconditions

* The configured resolver remains bound after transfer.
* Manager's fixed text inventory includes `domains.ens.primary-contacts`.
* Manager stores and parses the complete returned value.
* The supported keys are grouped so their first occurrences are far apart.
* The three-item cap remains downstream of the complete `indexOf`/`filter` scan.

## Affected Code

All links use frozen competition commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.

* [`text.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/data/records/text.tsx#L31-L41)
* [`profileRecords.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileRecords.ts#L93-L169)
* [`transformRecords.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/utils/transformRecords.ts#L116-L179)
* [`constants.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/dialogs/edit-profile/tabs/contact/constants.ts#L3-L7)
* [`records.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/dialogs/edit-profile/tabs/contact/records.ts#L136-L170)
* [`ProfileView.helpers.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileView.helpers.ts#L170-L194)
* [`ProfileContactSection.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileContactSection.tsx#L82-L99)
* [`profileAddressNames.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileAddressNames.ts#L29-L49)
* [`NameRow.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/dashboard/components/NameRow.tsx#L87-L115)
* [`wagmi.ts`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/lib/wagmi.ts#L13-L55)

## Recommended Mitigation

Reject an oversized or non-array `domains.ens.primary-contacts` value before normalization. Enforce a small element limit before iterating. Deduplicate with a `Set` in one pass and stop immediately when `maxPrimaryContactMethods` distinct supported keys have been collected. This makes the work proportional to a tightly bounded prefix rather than the full attacker-controlled array.

## Link to Proof of Concept

<https://gist.github.com/aydarkhusaenov/893e1d4d1ef3a18194834eee8eb2afbf/a345832c14c487187436d1d1813617645e8b0010>

## Proof of Concept

### Description

This full no-mock E2E PoC runs the frozen production Manager, deployed ENS V2 contracts on a fixed Sepolia fork, real digest-pinned Panoptes, the real Universal Resolver and Multicall3, and Chromium at CPU factor 1.

The attacker registers and configures a real attack name, then transfers the Registry ERC-1155 token to a fresh victim EOA. Panoptes indexes that owner/resolver state, and the public Manager address route displays the name without victim acceptance or a victim transaction. One ordinary click opens the vulnerable profile.

The grouped attack and interleaved control return the same 2,070,001-byte JSON multiset and render the same three exact cards. The harness captures and decodes the exact unmodified omitted-gas Manager browser request, proves the complete outer call succeeded, replays the exact transaction under an explicit 600,000,000 read ceiling, and asserts full-output equality. It also audits every state transaction under the real 60,000,000 block limit and proves the configured provider's matching response-size capability. No target source, RPC result, GraphQL row, browser response, or UI component is mocked or replaced.

### Where to place files and their names

The immutable Gist clones these twelve authored files under the frozen repository root. Restore the flattened Solidity filename to the shown `src/` path:

```
poc/manager-primary-contacts-quadratic-client-denial/
├── README.md
├── SOURCE_SHA256SUMS
├── docker-compose.poc.yml
├── foundry.toml
├── measure-browser.mts
├── package.json
├── panoptes-contracts.json
├── provider-capability-receipt.json
├── rpc-ceiling-proxy.mjs
├── run.sh
├── setup.mts
└── src/
    └── PrimaryContactsAmplifyingResolver.sol
```

### Command to run

Run with Node.js 22.23.2, pnpm 10.27.0, Foundry 1.7.1, Docker/Compose, and Playwright Chromium installed:

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-primary-contacts-poc
cd ens-primary-contacts-poc
git checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae

corepack enable
corepack prepare pnpm@10.27.0 --activate
pnpm install --frozen-lockfile
pnpm --dir e2e exec playwright install chromium

mkdir -p poc
git clone https://gist.github.com/aydarkhusaenov/893e1d4d1ef3a18194834eee8eb2afbf.git \
  poc/manager-primary-contacts-quadratic-client-denial
git -C poc/manager-primary-contacts-quadratic-client-denial checkout --detach \
  a345832c14c487187436d1d1813617645e8b0010

mkdir -p poc/manager-primary-contacts-quadratic-client-denial/src
mv poc/manager-primary-contacts-quadratic-client-denial/src__PrimaryContactsAmplifyingResolver.sol \
  poc/manager-primary-contacts-quadratic-client-denial/src/PrimaryContactsAmplifyingResolver.sol

(cd poc/manager-primary-contacts-quadratic-client-denial && \
  shasum -a 256 -c SOURCE_SHA256SUMS)

POC_LOG_DIR=/tmp/ens-primary-contacts-evidence \
bash poc/manager-primary-contacts-quadratic-client-denial/run.sh
```

The source-manifest SHA-256 is `9de05fd6adbf6fc00d08954bc1bf7037f1718dfc864f816ab0dc247353e6373e`. The clean reproduction used detached target and Gist revisions, installed the frozen lockfile, returned `OK` for every manifest entry, and exited successfully after removing its disposable containers and fork state.

### Logs proving the bug

```
ENS21_REAL_TRANSFER_DELIVERY_READY
ENS21_GIFTED_NAME_DISCOVERY_AND_CLICK_DENIAL_PROVED
ENS21_PRODUCTION_EQUAL_SIZE_CONTROL_PASS
ENS21_PRODUCTION_ATTACK_RENDER_DENIAL_PROVED
ENS21_PRODUCTION_BACK_NAVIGATION_DENIAL_PROVED
{
  "result": "ENS21_PRIMARY_CONTACTS_QUADRATIC_CLIENT_DENIAL_E2E_PASS",
  "name": "ens21-primary-contacts-attack-1788257592.eth",
  "victim": "0x81dc9046b61da595e3ae1eb6285a8a8c07638e11",
  "resolver": "0x52877414910c15981fea68af1b5af64c6de32b8a",
  "payloadBytes": 2070001,
  "elements": 270000,
  "exactIndexOfComparisons": 24300270000,
  "headerGasLimit": "60000000",
  "localStateTransactions": 19,
  "maximumStateTransactionGasLimit": "604774",
  "maximumStateTransactionGasUsed": "604774",
  "fullUniversalReadEstimatedGas": "329860138",
  "exactManagerRequestSha256": "a68f476a3ec3c948cd941be9f1126a9ae2d49dbe91d82e703022e945c60a4372",
  "browserRequestGasField": null,
  "explicitReplayGasCeiling": "600000000",
  "exactOuterCallSuccess": true,
  "exactOutputEqual": true,
  "renderedContacts": {
    "cardCount": 3,
    "contacts": [
      {"key":"mail","label":"Mailing Address","value":"1 Main Street"},
      {"key":"email","label":"Email Address","value":"victim@example.test"},
      {"key":"phone","label":"Phone Number","value":"+1 555 0100"}
    ]
  },
  "productionRpcResponseBytes": 4185702,
  "discoveryAddressDelayMs": 127.09308397769928,
  "giftedNameClickDelayMs": 35052.857582986355,
  "equalSizeControlDelayMs": 1400.5359590053558,
  "attackPostResponseDelayMs": 23910.494249999523,
  "attackHeartbeatStallMs": 23477.400000035763,
  "backDelayMs": 23381.006666958332,
  "siblingManagerLoadMs": 20.396750032901764
}
ENS21_PRIMARY_CONTACTS_QUADRATIC_CLIENT_DENIAL_E2E_PASS
Disposable containers and chain state removed.
```

**Full source:** [**immutable twelve-file PoC Gist**](https://gist.github.com/aydarkhusaenov/893e1d4d1ef3a18194834eee8eb2afbf/a345832c14c487187436d1d1813617645e8b0010)
