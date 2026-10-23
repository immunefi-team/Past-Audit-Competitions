# 89270 w a critical manager profile page resolves records from the raw url name but owner expiry from the normalized name letting an attacker display their own address under a victim name s identity

**Submitted on Aug 18th 2026 at 11:03:14 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89270
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Summary

The Manager's profile route (`/$name`) resolves a single page from two different ENS nodes.

* Owner, registration date, and expiry are resolved from `ens_normalize(urlParam)`.
* The address records, avatar, and every text record are resolved from the raw, unnormalized `urlParam`.

ENS normalization is many to one: many distinct byte strings normalize to the same name, and several of them are invisible when rendered. Those distinct strings are distinct onchain nodes, and the ENS registrar registers label strings verbatim without normalizing, so an attacker can own the unnormalized node.

The result is a page served from the real ENS Manager, over TLS, whose heading reads `vitalik.eth`, whose Owner field shows the genuine owner's reverse name (`vitalik.eth`) and whose Registered / Expires dates are the genuine ones, while the "Main receiving address" card, behind a copy to clipboard button, holds the attacker's address.

This is not homograph display confusion. It is incorrect resolution, corroborated by the victim name's authentic ownership metadata.

***

## Root cause

Three of the five profile data sources normalize; two do not.

### Normalize (→ the real name)

| File:line                                                             | Code                                               |
| --------------------------------------------------------------------- | -------------------------------------------------- |
| `apps/manager/src/features/profile/service/profileOwner.ts:25`        | `const ethName = normalizeEthName(params.name)`    |
| `apps/manager/src/features/profile/service/profileOwner.ts:56`        | `ensjsv2_getOwner(client, { name: ethName.name })` |
| `apps/manager/src/features/profile/service/profileExpiry.ts:77`       | `normalizeEth2LdName(name)`                        |
| `apps/manager/src/features/profile/service/profileRegistration.ts:66` | `normalizeEth2LdName(name)`                        |

`normalizeEthName` → `apps/manager/src/features/profile/service/profileName.ts:14-18` → `normalize()` from `viem/ens` (ENSIP-15 / `@adraffy/ens-normalize`).

### Do not normalize (→ the attacker's name)

| File:line                                                              | Code                                                                                                      |
| ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `apps/manager/src/features/profile/service/profileRecords.ts:82`       | `getProfileRecords = ResultFn(async function* (name: string) {`, no `normalize` call anywhere in the file |
| `apps/manager/src/features/profile/service/profileRecords.ts:122-132`  | `getRecords(client, { name, texts, coins, contentHash: true, abi: true, ... })`                           |
| `apps/manager/src/features/profile/service/profileRecords.ts:165-169`  | query keyed on the raw string                                                                             |
| `apps/manager/src/features/profile/service/profileEthAddress.ts:29-49` | same omission                                                                                             |
| `apps/manager/src/features/profile/service/profileAvatar.ts:11-12`     | raw name → metadata service                                                                               |

Both are called with the same raw router param from the loader:

```ts
// apps/manager/src/routes/$name/index.tsx:51-56
const [profileRecords, ownerData] = await Promise.all([
  queryClient.ensureQueryData(profileRecordsQuery(name)),   // RAW
  queryClient.fetchQuery(profileOwnerQuery(name)),          // NORMALIZED
])
```

Route param validation is `length >= 3 && includes('.')` only (`apps/manager/src/routes/$name/route.tsx:5-13`). No normalization, no canonical redirect.

### ensjs does not normalize (confirmed against the pinned build)

`@ensdomains/ensjs` is pinned in `pnpm-workspace.yaml:10` to `pkg.pr.new/ensdomains/ensjs/@ensdomains/ensjs@e96662c` (`5.0.0-alpha.1`). Installed and inspected:

* `src/actions/public/getRecords.ts`: zero occurrences of `normalize`.
* Across all of `src/actions/`, the only file mentioning `normalize` is `getName.ts` (reverse resolution, normalizing the *result*).
* `getRecords` delegates to `src/actions/public/resolveNameData.ts`, which does:
  * `:19` `import { packetToBytes } from 'viem/ens'`
  * `:70` `const nameWithSizedLabels = getNameWithSizedLabels(name)`
  * `:93` / `:109` `args: [toHex(packetToBytes(nameWithSizedLabels)), ...]`

`packetToBytes` performs no normalization. The name string reaches the UniversalResolver byte for byte.

The project already knows this, from `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:191`:

> `// ensjs splits the label without normalizing, so pass a normalized 2LD name.`

***

## Render path (source → sink)

| Rendered element                     | File:line                                                                                                | Name source | Shows                        |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------- | ----------- | ---------------------------- |
| `<h1>` name badge                    | `ProfileDetails.tsx:22-28`                                                                               | raw         | reads as the victim name     |
| Owner                                | `ProfileDetails.tsx:113-122`, `value={ownerReverseName \|\| truncateAddress(owner)}`                     | normalized  | genuine owner's reverse name |
| Registered / Expires                 | `ProfileView.tsx:102-112`                                                                                | normalized  | genuine dates                |
| Main receiving address (copy button) | `ProfileAddressesSection.tsx:90-95`, selected at `:159` → `ProfileView.helpers.ts:229-244` (coinType 60) | raw         | attacker's address           |
| Avatar / header                      | `ProfileView.tsx:139-143`                                                                                | raw         | attacker's                   |
| Description, socials, links, agents  | `ProfileCards.tsx:19-29`                                                                                 | raw         | attacker's                   |

Nothing aborts: `ownerMissing` (`ProfileView.tsx:120`) is false because the owner lookup succeeded. There is no cross check between `records.resolverAddress` and the owner's expected resolver; `normalizeResolverAddress` (`profileRecords.ts:66-67`) is a zero address check only.

The `<h1>` uses Tailwind `truncate`, which sets `white-space: nowrap`. A soft hyphen (U+00AD) only paints at a line break opportunity, so with `nowrap` it never renders at all.

***

## Impact

An attacker registers an unnormalized variant of any target name, for example Vitalik.eth, or vitalik.eth carrying an invisible soft hyphen. The Manager then serves a profile page showing the target's genuine owner, registration date and expiry next to the attacker's own address in the "Main receiving address" card, under a heading that reads as the target name.

Any user who copies that address sends funds to the attacker. The false address comes from the real application, on its own domain, corroborated by authentic onchain ownership data, so there is nothing a user can inspect to catch it. Clicking Favorite persists the poisoned name into the victim's account, after which their own dashboard keeps serving it.

Cost to the attacker is one ordinary registration per name squatted. Every .eth name is exposed.

***

## Recommendation

{% stepper %}
{% step %}

### Normalize in the record path

Apply the existing `normalizeEthName` / `normalizeDnsName` helpers in `getProfileRecords` (`profileRecords.ts:82`), `getProfileEthAddressSnapshot` (`profileEthAddress.ts:29`), and `buildNameImageUrl` (`profileAvatar.ts:11`).
{% endstep %}

{% step %}

### Canonicalize at the boundary

In the `/$name` loader, compute `normalize(name)` once and `redirect()` to the canonical URL when it differs. This removes the whole class, prevents the raw form from being stored in favorites or search history, and makes any future unnormalized call site unreachable.
{% endstep %}

{% step %}

### Fail closed

Fail closed on names that do not normalize, rather than falling through to a raw lookup.
{% endstep %}

{% step %}

### Add an invariant test

Consider an invariant test asserting that every profile data source is called with an identical, normalized name.
{% endstep %}
{% endstepper %}

## Proof of Concept

### Part A: offline, deterministic (run as is, \~30 seconds)

Proves that the two strings the page resolves are two different onchain nodes.

```bash
mkdir -p /tmp/ens-poc && cd /tmp/ens-poc
npm init -y >/dev/null
npm install viem @adraffy/ens-normalize >/dev/null

cat > poc.mjs <<'EOF'
import { namehash, normalize, packetToBytes } from 'viem/ens'
import { toHex } from 'viem'

const cases = [
  ['soft hyphen U+00AD (invisible)', 'vita­lik.eth'],
  ['ASCII capital',                  'Vitalik.eth'],
  ['variation selector U+FE0F',      'vitalik️.eth'],
]

for (const [label, raw] of cases) {
  const norm = normalize(raw)
  console.log(`\n--- ${label} ---`)
  console.log(`raw              : ${JSON.stringify(raw)}`)
  console.log(`normalize(raw)   : ${JSON.stringify(norm)}   <- getOwner / getExpiry resolve THIS`)
  console.log(`node(raw)        : ${namehash(raw)}   <- getRecords resolves THIS`)
  console.log(`node(normalized) : ${namehash(norm)}`)
  console.log(`SAME NODE?       : ${namehash(raw) === namehash(norm)}`)
  console.log(`UR bytes(raw)    : ${toHex(packetToBytes(raw)).slice(0, 42)}...`)
  console.log(`UR bytes(norm)   : ${toHex(packetToBytes(norm)).slice(0, 42)}...`)
}
EOF
node poc.mjs
```

Actual output:

```
--- soft hyphen U+00AD (invisible) ---
raw              : "vita­lik.eth"
normalize(raw)   : "vitalik.eth"   <- getOwner / getExpiry resolve THIS
node(raw)        : 0x3c095ce5f7be325c15df51977928155a92bf4457e193134fb5731c4bb03338d2   <- getRecords resolves THIS
node(normalized) : 0xee6c4522aab0003e8d14cd40a6af439055fd2577951148c14b6cea9a53475835
SAME NODE?       : false
UR bytes(raw)    : 0x0976697461c2ad6c696b0365746800...
UR bytes(norm)   : 0x07766974616c696b0365746800...

--- ASCII capital ---
raw              : "Vitalik.eth"
normalize(raw)   : "vitalik.eth"
node(raw)        : 0xfc1a1eb20849620be486e201e6aa4ab264c7a01f1e48d8a9ae3aacead41c405e
node(normalized) : 0xee6c4522aab0003e8d14cd40a6af439055fd2577951148c14b6cea9a53475835
SAME NODE?       : false

--- variation selector U+FE0F ---
raw              : "vitalik️.eth"
normalize(raw)   : "vitalik.eth"
node(raw)        : 0x2faafc764230cf5eb0dea1fea0e1eba7c680d6190aec0b041b81ce25316b9700
node(normalized) : 0xee6c4522aab0003e8d14cd40a6af439055fd2577951148c14b6cea9a53475835
SAME NODE?       : false
```

Note the UniversalResolver wire bytes: `0x09 76697461 c2ad 6c696b`, a length 9 label `vita<C2 AD>lik`, versus `0x07 766974616c696b`. Two different labels sent to the resolver.

### Label candidates (empirically verified against `@adraffy/ens-normalize` 1.10.1)

| Candidate                                  | → normalizes to `vitalik.eth` | Visible difference |
| ------------------------------------------ | ----------------------------- | ------------------ |
| `vita<U+00AD>lik.eth` (soft hyphen)        | ✅                             | none               |
| `vitalik<U+FE0F>.eth` (variation selector) | ✅                             | none               |
| `Vitalik.eth`                              | ✅                             | one capital letter |
| `VITALIK.ETH`                              | ✅                             | all caps           |
| `vitalik.ETH`                              | ✅                             | caps TLD           |
| `Ｖitalik.eth` (fullwidth)                  | ✅                             | wide first letter  |
| `𝐯𝐢𝐭𝐚𝐥𝐢𝐤.eth` (math bold)           | ✅                             | bold styling       |
| `Ⅴitalik.eth` (roman numeral V, U+2164)    | ✅                             | negligible         |
| `vitaℓik.eth` (script ℓ, U+2113)           | ✅                             | slight             |
| `vitaliK.eth` (Kelvin sign, U+212A)        | ✅                             | negligible         |
| `vita<ZWJ>lik.eth`                         | ❌ throws                      | n/a                |

### Part B: reproduce end to end on a local Anvil fork

Run this against a fork, never against a live network. The repo ships a complete local stack for exactly this (Anvil fork of Sepolia + bundler + paymaster + orchestrator + indexer):

```bash
pnpm --filter @ens-apps/e2e infra:up      # e2e/infra/scripts/start-local-env.sh
# Anvil fork listens on <http://127.0.0.1:8545>
```

Point the Manager at it. `apps/manager/.env.ci` already carries the local fork overrides, and `apps/manager/vite.config.ts:20-23` proxies `/rpc` to the fork:

```
VITE_SEPOLIA_RPC_URL=/rpc
VITE_SEPOLIA_RPC_URL_SERVER=http://127.0.0.1:8545
```

```bash
pnpm --filter @ens-apps/manager dev        # <http://localhost:3000>
```

Register both sides on the fork, from two Anvil funded accounts. Do not target a name anyone owns.

{% stepper %}
{% step %}

### Victim side

Register `pocvictim.eth` normally, set `addr(60)`, an avatar, a description, and a primary name on wallet A so the Owner row renders a reverse name.
{% endstep %}

{% step %}

### Attacker side

Register the raw label from wallet B:

```bash
LABEL=$(printf 'pocvict\xc2\xadim')
```

Confirm with Part A's script that `ens_normalize(LABEL + '.eth') === 'pocvictim.eth'` while the two namehashes differ.
{% endstep %}

{% step %}

### Register directly on the fork

Registration must be a direct contract call on the fork, because the Manager's own registration path lowercases and would strip the trick (`name-parser.ts:46`). The e2e helpers pass `label` straight through to `register(string label, ...)` (`e2e/fixtures/makeV2Name.ts:72-75`), so they are a convenient starting point, but note `makeV2Name` appends a timestamp suffix to the label (`:199`), so for this PoC call the registrar directly rather than reusing that fixture verbatim.
{% endstep %}

{% step %}

### Set attacker-controlled records

On the resolver set in step 3, write `setAddr(node, 60, WALLET_B)` plus `avatar`, `description`, `com.twitter`, copying the values from `pocvictim.eth` so every visible field matches except the address.
{% endstep %}
{% endstepper %}

Two things to record while you are on the fork, since both were previously only reasoned from source:

* Whether `getRegisterPrice(label, ...)` prices the unnormalized label and `makeCommitment(label, ...)` returns a hash for it, i.e. that the registrar applies no normalization or character validation. The v2 ETHRegistrar ABI declares `InvalidOwner` and `NameNotAvailable` and no label validity error (`@ensdomains/ensjs-abi/v2/ethRegistrar`), so this is expected, but confirm it on the fork rather than asserting it.
* That `getRecords(rawName)` and `getOwner(normalize(rawName))` return data from the two different nodes, the data layer form of the divergence, using the app's own two calls.

### Part C: observe

Load the two URLs side by side against the local app:

```
<http://localhost:3000/pocvictim.eth>          # control
<http://localhost:3000/pocvict%C2%ADim.eth>     # attack
```

Expected on the attack page, all on one page:

| Field                          | Value                                                                         | Comes from |
| ------------------------------ | ----------------------------------------------------------------------------- | ---------- |
| `<h1>`                         | renders as `pocvictim.eth`, soft hyphen invisible under `white-space: nowrap` | raw        |
| Owner                          | wallet A's reverse name, i.e. `pocvictim.eth`                                 | normalized |
| Registered / Expires           | wallet A's dates                                                              | normalized |
| Main receiving address         | wallet B, with a working copy button                                          | raw        |
| Avatar / description / socials | wallet B's                                                                    | raw        |

A weaker but even cheaper variant, if you want a screenshot before doing any registration at all: register only `pocvictim.eth` on the fork, then load `/Pocvictim.eth`. Owner, Registered and Expires stay populated and correct (normalized lookups), while the Addresses card disappears entirely (`ProfileAddressesSection.tsx:163` returns `null` when the raw node has no records), along with Social/Links/Agents, and the avatar drops to the generated fallback. That already shows the two halves of the page resolving different nodes.

Evidence to capture:

* Screenshots of both pages, plus a diff/overlay showing the `<h1>` is pixel identical.
* A screen recording of clicking the copy button on the attack page and pasting, showing wallet B's address on the clipboard while the page says `pocvictim.eth` and names wallet A as owner.
* Both `namehash` values and the two fork registration tx hashes.
* Optional, and worth including because it proves persistence rather than reflection: sign in, click Favorite on the attack page, then navigate to `/dashboard` and screenshot the poisoned entry rendering from the victim's own account (`addFavorite.ts:18-21` stores the raw name; `NameRow.tsx:111` links back with it).

## Attack delivery

{% stepper %}
{% step %}

### Paste

ENS names travel as text: bios, Discord, invoices, marketplace listings. A pasted `vita<U+00AD>lik.eth` into the app's own search bar survives: `parseSearchInput` (`searchSuggestions.utils.ts:53-54`) and `normalizeQuery` (`nameUtils.ts:31-35`) apply only `trim().toLowerCase()`, which performs no UTS-46 mapping and strips no ignored characters. The suggestion links to `/$name` with the raw string (`SuggestionItem.tsx:25-30`).
{% endstep %}

{% step %}

### Link

`https://app.ens.domains/Vitalik.eth` needs no invisible characters and looks unremarkable, because capitalisation genuinely *is* meaningless in ENS, which is exactly why it works.
{% endstep %}

{% step %}

### Squat and wait

Register capitalisation variants of popular names; any link anyone ever writes with a capital resolves to the attacker.
{% endstep %}
{% endstepper %}

### Persistence and amplification

* **Favorites.** `ProfileFavoriteAction` (`ProfileActions.tsx:96`) sends the raw name to the backend: `addFavorite.ts:18-21`, `$put({ param: { name } })`. It is then rendered from the victim's own dashboard and linked back with that raw name (`NameRow.tsx:111`). After one favorite, the victim's own dashboard serves the poisoned profile indefinitely, with no attacker link needed again.
* **Share.** `ProfileShareAction` receives `url = getProfileUrl(name)` (`ProfileView.tsx:60-65`) → `/p/${rawName}`; `/p/$name` redirects preserving the param (`routes/p/$name/index.tsx:9`). A victim who shares what they believe is the real profile propagates the poisoned link.
* Search history persists it (`useSearchHistory.ts`).
* The Renew action carries the raw name, so a victim can be induced to pay the squat's renewal.

### Why existing controls do not fire

* `validateENSName` / `parseName` never run on this route; the loader calls the queries directly (`routes/$name/index.tsx:51-56`).
* No CSP relevance and no injection: the app resolves the wrong node itself.
* `ssr: false` (`routes/$name/index.tsx:142`) means no OG preview renders, so link unfurls reveal nothing.

## Preempting the likely closure arguments

### Out of scope: "Reflected plain text injection (e.g. url parameters, path, etc.)"

Nothing is injected. The URL param is not reflected into the page as content; it is used as a *lookup key* against the ENS registry, and the attacker's content is read from the blockchain by the app's own resolver calls. The rendered bytes never come from the URL. Note also that the same out of scope line explicitly does not exclude *"persistent plain text injection"*, and the Favorites path (`addFavorite.ts:18-21` → `NameRow.tsx:111`) makes this persistent in the victim's own account after one click, which is why capturing that step is worth the extra two minutes.

### Out of scope: "Theoretical impacts without any proof or demonstration"

Addressed by executing Parts A and B/C. Part A is deterministic and already run, and proves the page resolves two different nodes; Parts B and C run on a local Anvil fork and demonstrate the full impact. Do not submit on Part A alone.

### Out of scope: "Impacts caused by vulnerabilities requiring un-prompted, in-app user actions that are not part of the normal app workflows"

The only user action required is *opening a profile page*, which is the app's primary workflow. No unusual settings, no debug route, no wallet interaction is needed to reach the spoofed page.

### Prohibited activity: "Attempting phishing or other social engineering attacks against our employees and/or customers"

That clause governs *conduct during testing* (do not phish real users); the PoC above uses two self owned names and touches nobody. It is not a scope limit on the finding: the program lists *"Causing the application to display or pre-fill an incorrect address... so that a user sends funds... to an unintended recipient"* as a Critical impact, which necessarily presumes a user who arrives at a page and trusts what it shows. Arriving at an ENS profile via a link or a pasted name is normal use, not social engineering, and the deception is performed by the application itself, which renders the genuine owner's identity and registration history next to the attacker's address.

### "It's just a homograph / user should check the URL."

The plain capital variant (`/Vitalik.eth`) involves no lookalike characters at all, and capitalisation is genuinely meaningless in ENS; that is the premise of normalization. There is no URL inspection a user could perform that would reveal anything wrong. For the soft hyphen variant there is no visual difference to inspect in the first place.

### Feasibility

Cost is one testnet/mainnet registration per squatted name. Per the program's own Feasibility Limitations, *Attack Investment Amount* may not be cited to downgrade.

## Not a duplicate of any listed known issue

### EXP-INPUT-005 (nearest)

*"ENS name validators accept inputs that ENS normalization later rejects or transforms — homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot — causing homograph-style display confusion in the UI rather than incorrect resolution."*

* Different root cause and different layer. EXP-INPUT-005 is validator leniency (`validateENSName`, `parseName`). This is a resolution inconsistency between sibling queries on one page; neither validator is on this code path.
* It reproduces with a single plain ASCII capital letter: no homoglyph, no control character, nothing any validator would or should reject. `Vitalik.eth` *is* `vitalik.eth` under ENS normalization.
* Its impact is the one EXP-INPUT-005 explicitly rules out: incorrect resolution, not display confusion.
* The EXP-INPUT-005 fix does not fix this. Tightening the validators changes nothing in `profileRecords.ts`, which never calls one. The fix here is a `normalize()` call in the record path.
* No listed entry covers displaying the genuine owner's identity and registration history alongside an attacker controlled receiving address. That combination is what turns confusion into a credible wrong recipient prompt.

### WEB-1301

`$address` view showing the wallet's primary name metadata. Different route; there the metadata belongs to one real name, here two names' data are interleaved on one page.

### QA-06

Stale primary subname. Unrelated.

### R2-02

No markup is injected; not an injection finding.
