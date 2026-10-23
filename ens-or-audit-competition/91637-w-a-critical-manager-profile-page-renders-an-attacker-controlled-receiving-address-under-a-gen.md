# 91637 w a critical manager profile page renders an attacker controlled receiving address under a genuine name s verified owner identity

**Submitted on Sep 3rd 2026 at 08:42:11 UTC by @risinghorse for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91637
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves
  * Displayed-address substitution leading to loss of user funds. The address a user copies from a genuine-looking profile belongs to the attacker.
  * Persistent and remotely triggered. The trigger is on-chain state the victim's browser is required to read; it survives reloads and cache clears, and needs no further attacker action once set.
  * Low cost, broad target set. One registration per impersonated name. Every name with a mixed-case variant, and every name with a mapped-codepoint variant, is targetable — the variant space per name is large.

## Description

## Brief/Intro

The Manager's profile page resolves a name's **ownership, expiry and registration date** from the ENSIP-15-**normalized** name, but resolves **every resolver record — including the ETH receiving address — from the raw, un-normalized URL parameter**. Because `namehash` hashes label bytes verbatim, those are two different on-chain nodes, so a single profile page can display one name's genuine, forward-verified owner and registration history alongside a *different* name's receiving address. An attacker who registers a non-normalized label that normalizes to a high-value name (for example the literal `VICTIM`, which normalizes to `victim`) and points its `addr(60)` at themselves obtains a profile page that presents their own address under the real owner's identity and the real registration dates. In production this converts the app's own provenance display into corroboration for a fraudulent payment address: a user who performs exactly the verification the UI invites will copy the attacker's address and send funds to it.

***

**Severity:** Critical **Asset:** `apps/manager` (ENS Manager app) — in scope per the program's Assets in Scope **Target:** ENS Manager & Explorer — Audit Competition **Audited revision:** `63772fd872af472ced58b009499355f3430c2a86`

**Selected impact:** *Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient* — sub-bullet *"resolving or rendering an ENS name to an attacker-controlled address."*

> **Testing & rules compliance.** No testing was performed against mainnet or any public testnet. This is a client-side name-resolution defect; it is demonstrated below as a static data-flow trace over the in-scope source at the audited revision, plus a reproduction on a **local chain only** (the repository's own Dockerised Anvil fork), with the Manager dev server pointed at that local chain. No public-testnet or mainnet contract was registered, called, or written to.

***

## Vulnerability Details

The profile route fans five queries out from one route parameter (`ProfileView.tsx:93-112`), but two different strings reach the chain.

### The normalized path — owner, expiry, registration

`apps/manager/src/features/profile/service/profileName.ts:14-21`

```ts
export const normalizeEthName = (name: string): EthName | null => {
  let normalized: string

  try {
    normalized = normalize(name)          // ENSIP-15, via viem/ens
  } catch {
    return null
  }
  // ...
}
```

`apps/manager/src/features/profile/service/profileOwner.ts:24-25, 55-58`

```ts
export const getOwner = ResultFn(async function* (params: { name: string }) {
  const ethName = normalizeEthName(params.name)                 // NORMALIZED
  // ...
  const v2Owner = yield* fromPromise(
    ensjsv2_getOwner(client, { name: ethName.name }),           // uses ethName.name
    (e) => new GetOwnerError({ cause: e }),
  )
```

The same normalized value is used for the indexer lookup (`profileOwner.ts:70`, `namehash(ethName.name)`) and the v1 fallback (`profileOwner.ts:87-88`). Expiry and registration do likewise:

```ts
// profileExpiry.ts:77
const ethName = normalizeEth2LdName(name)      // NORMALIZED
// profileRegistration.ts:66
const ethName = normalizeEth2LdName(name)      // NORMALIZED
```

### The raw path — every resolver record, including `addr(60)`

`apps/manager/src/features/profile/service/profileRecords.ts:82, 122-131`

```ts
export const getProfileRecords = ResultFn(async function* (name: string) {
  // ...
  const records = yield* fromPromise(
    getRecords(client, {
      name,                          // <-- RAW route param, never normalized
      texts,
      coins,
      contentHash: true,
      abi: true,
      ignoreInvalidCoinTypes: true,
    }),
    (error) => new GetProfileRecordsError({ cause: error }),
  )
```

Coin type 60 is always requested (`apps/manager/src/features/profile/data/records/utils.ts:42`):

```ts
export const alwaysProbeAddressRecords = ['60']
```

The indexer lookup in the same file also takes the raw string (`profileRecords.ts:95`, `fetchIndexerDomain(name)`), and the avatar/header images are built from it too (`profileAvatar.ts:11-13`):

```ts
const url = new URL(
  `${ENS_METADATA_V2_SEPOLIA_URL}/${kind}/${encodeURIComponent(name)}`,   // RAW
)
```

### Both are rendered on one screen

`apps/manager/src/features/profile/components/view/ProfileView.tsx:93-94, 99-100, 108, 111`

```tsx
const { data: profileRecords, refetch: refetchRecords } = useSuspenseQuery({
  ...profileRecordsQuery(name),                            // RAW  -> address, avatar, links
})
// ...
const { data: ownerData, isPending: isOwnerPending } = useQuery({
  ...profileOwnerQuery(name),                              // NORMALIZED -> owner
})
// ...
  ...profileExpiryQuery(name, ownerData?.protocol),        // NORMALIZED
// ...
  ...profileRegistrationQuery(name, ownerData?.protocol),  // NORMALIZED
```

The heading prints the raw parameter (`ProfileDetails.tsx:22-27`):

```tsx
export const ProfileNameBadge = ({ name }: { readonly name: string }) => (
  <div className="inline-flex max-w-full items-center rounded-[3px] ...">
    <h1 className="truncate font-semi-mono text-[32px] leading-[1.12]">
      {name}
    </h1>
  </div>
)
```

The Owner row prints the value derived from the **normalized** lookup (`ProfileDetails.tsx:113-122`):

```tsx
{owner ? (
  <ProfileDetail
    copyValue={owner}
    // ...
    label={<Trans>Owner</Trans>}
    value={ownerReverseName || truncateAddress(owner)}      // line 121
  />
) : null}
```

And the receiving-address card prints coin type 60 from the **raw**-name records, inside a one-click copy control:

`apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:229-237`

```ts
const getMainAddressCandidate = (
  records: ProfileRecords,
): IndexedProfileAddressItem | undefined => {
  const addresses = getAddressItems(records)
  return (
    addresses.find((address) => address.coinType === ETH_COIN_TYPE) ??  // 60
    addresses[0]
  )
}
```

`apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:76, 90-95`

```tsx
const MainAddressCard = ({ address, avatarUrl, chains, name }) => {
  // ...
  return (
    <CopyableButton
      // ...
      value={address.value}          // one-click copy of addr(60) from the RAW node
    >
```

### The owner identity is genuine, not forged — which is the point

`profileReverseName.ts:13-31` documents and uses viem's `getEnsName`, whose ENSIP-19 reverse resolution performs the bidirectional forward check on-chain:

```ts
/**
 * ENSIP-19 reverse resolution via the UniversalResolver's
 * `reverseWithGateways(address, coinType)`: the forward-verification
 * (bidirectional check) and the chain-specific → `default.reverse` fallback
 * both happen on-chain in the reverse resolvers ...
 *
 * Returns `null` when the address has no verified primary name (including
 * when the reverse claim does not forward-resolve back to the address).
 */
```

The attacker therefore **cannot** make the Owner row assert something false — and does not need to. They rely on it being genuinely **true** for `victim.eth` while the receiving address beside it is theirs.

### There is no normalization gate in the Manager

The only route guard is a length/dot check (`apps/manager/src/routes/$name/route.tsx:5-13`):

```ts
params: {
  parse: ({ name }) => {
    if (name.length < 3 || !name.includes('.')) {
      throw new Error('Invalid ENS name')
    }
    return { name }
  },
},
```

A full inventory of ENSIP-15 usage across `apps/manager/src` (non-test) shows `normalize()` is called **only** inside the owner/expiry/registration services and `setPrimaryName.ts` — never as a route- or record-level gate:

```
$ grep -rn "ens_normalize\|from 'viem/ens'" apps/manager/src
  features/profile/service/setPrimaryName.ts:28:      import { normalize } from 'viem/ens'
  features/profile/service/profileName.ts:1:         import { normalize } from 'viem/ens'
  features/profile/service/profileImageRecord.ts:7:  import { parseAvatarRecord } from 'viem/ens'
  features/migration/utils/dnsEncodeName.ts:2:       import { packetToBytes } from 'viem/ens'
  features/migration/service/migrationApprovalNeeds.ts:1: import { labelhash } from 'viem/ens'
```

The share route forwards the raw parameter unchanged (`apps/manager/src/routes/p/$name/index.tsx:9`):

```ts
throw redirect({ to: '/$name', params: { name } })
```

By contrast the **Explorer** does gate on normalization (`apps/portal/src/routes/$name/index.tsx:442`, `const isValidName = isValidEnsName(name)` → `<InvalidNameMessage />`), which establishes the intended invariant and shows the Manager is the outlier.

### The codebase already states the rule this file breaks

`apps/manager/src/features/dashboard/components/NamesTable.tsx:84-86`

```ts
// Use the canonical normalized name for the on-chain label — the display name
// may be unnormalized (e.g. mixed case) and would hash to the wrong label.
const name = domain.normalizedName ?? displayName
```

`profileRecords.ts` is precisely the place that does not apply it.

### The attacker's label is registrable on-chain

The v2 registrar validates byte length, not charset, so a non-normalized label registers cleanly as a distinct node:

`contracts-v2-main/contracts/src/registrar/StandardRentPriceOracle.sol:365-368`

```solidity
function getBasePrice(string calldata label, uint64 duration) public view returns (uint256) {
    uint256 n = bytes(label).length;
    if (n == 0 || n > 255)
        return 0; // too long or too short
```

`register()` itself only rejects the zero owner and never inspects the label's charset (`contracts-v2-main/contracts/src/registrar/ETHRegistrar.sol:123-142`).

### Recommended fix

1. Normalize once at the route boundary and use that single canonical string for **every** downstream query. Concretely, in `getProfileRecords` (`profileRecords.ts:82`) normalize `name` before `fetchIndexerDomain` and `getRecords`, and bail (or redirect to the normalized URL) when `normalize()` throws or changes the string.
2. Add a normalization gate to `routes/$name/route.tsx` and the `/p/$name` share route, mirroring the Explorer's `isValidEnsName` check.
3. Render the normalized name in the `<h1>` rather than the raw parameter, so display, hashing and every query agree.

***

## Impact Details

### What an exploit costs the victim

The loss is the full value the victim transfers. There is no cap imposed by the app: the profile page is the canonical place a user looks up "where do I send funds for this name," and the address it presents is copied with a single click (`ProfileAddressesSection.tsx:90`). A single successful use is an unbounded, irreversible on-chain transfer to the attacker.

The exploit economics strongly favour the attacker:

* **Cost per target: one name registration.** Nothing else is required — no compromise, no phishing infrastructure beyond a link, no interaction with the victim's wallet.
* **Target set is every valuable name.** Every name has at least one mixed-case variant, and many have mapped-codepoint variants (see the seamless variant in the PoC), so the reachable target set is effectively the whole registry.
* **Persistent and unattended.** The trigger is on-chain state the victim's browser is required to read. Once written, the attacker takes no further action; the page misrenders for every visitor, indefinitely, surviving reloads and cache clears.
* **Defeats the user's own diligence.** A user who does *not* check the owner is phished by ordinary means. This bug specifically defeats the user who *does* check, because the owner and registration data they verify are genuine — they simply belong to the other node.

### Mapping to the program's in-scope impacts

Claimed:

* ✅ **Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient**, specifically *"resolving or rendering an ENS name to an attacker-controlled address."* The profile of a name the user believes is genuine renders an `addr(60)` read from a node the attacker owns.

Explicitly **not** claimed, to avoid overstating scope:

* ❌ *Direct theft of user funds* as a standalone impact — the application does not itself move funds. The loss is realised by the user acting on the displayed address, which is why the display bullet above is the accurate mapping.
* ❌ *Injection of malicious HTML or XSS through metadata* — nothing executes; records render as escaped text.
* ❌ *Hijacking or bricking an ENS name* — the victim's name, ownership and registry are untouched.
* ❌ *Taking state-modifying authenticated actions on behalf of other users* — no action is performed as the victim.

### Prerequisites, stated plainly

The victim must open an attacker-supplied profile URL, and the attacker must pre-register the non-normalized label. There is **no** requirement that the victim connect a wallet, sign anything, or have any prior relationship with the attacker, and the page requires no interaction beyond viewing it and copying the address it presents.

### Duplicate analysis

This is **not** a duplicate of **EXP-INPUT-005** ("ENS name validators accept homoglyphs, bidi and control characters"). That entry is scored Medium and the Known Issues document explicitly bounds its impact to *"homograph-style display confusion in our UI rather than incorrect resolution."*

The defect reported here **is** the incorrect resolution that entry disclaims: one page resolving two different on-chain nodes and presenting them as a single identity, so the ETH receiving address shown belongs to a different node than the owner and registration metadata rendered beside it. It therefore falls under the program's own carve-out — *"new consequences of a listed root cause that materially change its severity."* Had the Manager normalized consistently, accepting `VICTIM.eth` would render `victim.eth`'s real data and the impact really would be display confusion only.

It is likewise distinct from **QA-03** (per-year vs total price display mismatch) and the Round-3 transaction-manager findings, which concern amounts and archiving rather than name resolution.

***

## References

In-scope code (audited revision `63772fd872af472ced58b009499355f3430c2a86`):

* `apps/manager/src/features/profile/service/profileRecords.ts:82, 95, 122-131` — raw name passed to `getRecords`
* `apps/manager/src/features/profile/service/profileName.ts:14-21` — `normalizeEthName`, ENSIP-15
* `apps/manager/src/features/profile/service/profileOwner.ts:24-25, 55-58, 70, 87-88` — normalized owner lookup
* `apps/manager/src/features/profile/service/profileExpiry.ts:77` / `profileRegistration.ts:66` — normalized
* `apps/manager/src/features/profile/service/profileAvatar.ts:11-13` — raw name in image URL
* `apps/manager/src/features/profile/service/profileReverseName.ts:13-31` — ENSIP-19 forward-verified owner name
* `apps/manager/src/features/profile/components/view/ProfileView.tsx:93-112` — both paths on one screen
* `apps/manager/src/features/profile/components/view/ProfileDetails.tsx:22-27, 113-122` — raw `<h1>`, normalized Owner row
* `apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:229-237` — coin-type-60 selection
* `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:76, 90-95` — copyable address card
* `apps/manager/src/features/profile/data/records/utils.ts:42` — `alwaysProbeAddressRecords = ['60']`
* `apps/manager/src/routes/$name/route.tsx:5-13` — the only route guard
* `apps/manager/src/routes/p/$name/index.tsx:9` — share route forwards the raw param
* `apps/manager/src/features/dashboard/components/NamesTable.tsx:84-86` — the project's own statement of the rule
* `apps/portal/src/routes/$name/index.tsx:442` — the Explorer's normalization gate (the intended invariant)
* `contracts-v2/contracts/src/registrar/StandardRentPriceOracle.sol:365-368` — label length is the only validation
* `contracts-v2/contracts/src/registrar/ETHRegistrar.sol:123-142` — `register()` does not inspect the charset

Specifications and libraries:

* ENSIP-15, ENS Name Normalization — <https://docs.ens.domains/ensip/15>
* ENSIP-19, Reverse Resolution — <https://docs.ens.domains/ensip/19>
* `@adraffy/ens-normalize` (the normalization implementation behind viem's `normalize`) — <https://github.com/adraffy/ens-normalize.js>
* viem `normalize` — <https://viem.sh/docs/ens/utilities/normalize>
* viem `namehash` (hashes labels as given; callers must normalize first) — <https://viem.sh/docs/ens/utilities/namehash>

## Proof of Concept

## Proof of Concept

**All steps run offline or against a local chain.** The repository ships a Dockerised Anvil fork for exactly this purpose and the Manager has a first-class environment override to point at it, so nothing below touches a public network.

### Part 1 — Static proof of the split (no chain required)

For any input string `N` where `normalize(N) !== N`:

| Query                         | Name actually sent on-chain | Source                      |
| ----------------------------- | --------------------------- | --------------------------- |
| `profileOwnerQuery(N)`        | `normalize(N)`              | `profileOwner.ts:24, 55`    |
| `profileExpiryQuery(N)`       | `normalize(N)`              | `profileExpiry.ts:77`       |
| `profileRegistrationQuery(N)` | `normalize(N)`              | `profileRegistration.ts:66` |
| `profileRecordsQuery(N)`      | **`N` (raw)**               | `profileRecords.ts:82, 124` |
| avatar / header image         | **`N` (raw)**               | `profileAvatar.ts:11-13`    |

`namehash` hashes label bytes as given, so `namehash(N) !== namehash(normalize(N))`: the first three rows and the last two address **different on-chain nodes**, and `ProfileView.tsx:93-112` renders all of them on one screen.

Confirming the premise (uppercase is mapped by ENSIP-15; `namehash` does not normalize):

```js
import { normalize, namehash } from 'viem/ens'

normalize('VICTIM.eth')                            // 'victim.eth'
namehash('VICTIM.eth') === namehash('victim.eth')  // false  <-- two distinct nodes
```

### Part 2 — Local end-to-end reproduction

Environment: local Anvil fork on `127.0.0.1:8545`, started by the repository's own tooling. No public testnet or mainnet is touched.

```bash
# 1. Start the repository's local chain (Anvil, forked state, local only)
pnpm e2e:infra:up          # -> anvil listening on 127.0.0.1:8545

# 2. Point the Manager at the LOCAL chain, per e2e/infra/docker-compose.yml:8
#    apps/manager/.env.local
echo 'VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8545' >> apps/manager/.env.local

# 3. Run the Manager against it
pnpm dev:manager           # -> http://localhost:3000
```

The override is read at `apps/manager/src/lib/wagmi.ts:25`:

```ts
return import.meta.env?.VITE_SEPOLIA_RPC_URL || MANAGER_SEPOLIA_RPC_URL
```

On that **local** chain:

1. Pick a label already registered on the forked state and controlled by a third party; this stands in for the impersonation target. Call it `victim.eth`.
2. From an attacker account, register the literal label **`VICTIM`** (uppercase) through the v2 `ETHRegistrar` commit/reveal flow (`commit` → wait `MIN_COMMITMENT_AGE` → `register`). The registrar validates byte length only, so the non-normalized label registers cleanly and yields a node distinct from `victim`.
3. Set a resolver on the attacker's new name and write `addr(60) = 0xATTACKER`, plus an `avatar` and a plausible `description` / `url`.
4. Open `http://localhost:3000/VICTIM.eth`.

### Observed result

`routes/$name/route.tsx:6` accepts the parameter (length ≥ 3, contains a dot). The page then renders, on one screen:

| Field                                    | Value shown                       | Node it came from                          |
| ---------------------------------------- | --------------------------------- | ------------------------------------------ |
| `<h1>` heading                           | `VICTIM.eth`                      | raw parameter (`ProfileDetails.tsx:22-27`) |
| **Owner**                                | the genuine owner of `victim.eth` | **normalized** node                        |
| **Registered / Expires**                 | the genuine dates of `victim.eth` | **normalized** node                        |
| **Main receiving address** (copy button) | **`0xATTACKER`**                  | **raw** node                               |
| Avatar / links                           | attacker's                        | **raw** node                               |

### Outcome

A user performing exactly the verification the UI invites — confirming the Owner row matches the expected owner and that the registration dates match an explorer — concludes the profile is authentic, clicks the copy control on the main receiving address, and sends funds to `0xATTACKER`.

### Delivery vector in production

The attacker shares `https://app.ens.domains/VICTIM.eth` in a DM, a post, a QR code or a marketplace listing. A capitalized ENS name looks ordinary and contains no confusable characters, so no homograph heuristic fires. The `/p/$name` share route (`routes/p/$name/index.tsx:9`) redirects to `/$name` with the raw parameter intact, so shared profile links carry the payload unchanged.

### Visually seamless variant (stronger)

Instead of uppercase, use a codepoint that `String.prototype.toLowerCase()` leaves unchanged but ENSIP-15 **maps** to ASCII — for example **U+2170 ROMAN NUMERAL SMALL ONE**, which renders as an ordinary `i`. Then `vⅰctim.eth` normalizes to `victim.eth` yet hashes to the attacker's node, so the `<h1>` *and* the Owner row both visually read `victim.eth` and nothing on the page appears wrong. The uppercase case is given first because it requires no confusable at all.
