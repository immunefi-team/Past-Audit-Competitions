# 91997 w a low manager re authors a prior owner s social records into the recipient s replacement resolver despite a clear old details promise

**Submitted on Sep 6th 2026 at 05:09:45 UTC by @Jornason for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91997
* **Report Type:** Websites & Apps
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (persistent), such as:
* HTML injection without JavaScript
* Replacing existing text with arbitrary text
* Arbitrary file uploads, etc.

## Description

## Brief/Intro

ENS Manager's Edit Profile dialog normalizes every social text record in the form before dirty-tracking, then diffs those normalized values against the raw on-chain snapshot. After an ordinary `.eth` name transfer, a prior owner's non-canonical social record (for example `com.twitter` stored as `https://twitter.com/attacker`, which the app's own display normalizer rewrites to `attacker`) therefore appears "modified" the moment the new owner edits only an unrelated field such as `description` and saves. When the recipient cannot write the inherited resolver, Manager's resolver-recovery flow shows a confirmation that promises "This name still has profile details from a previous owner. Continuing clears them and keeps only the changes you save." — but the transaction the application actually builds is a resolver `multicall(bytes[])` containing nine attacker-selected social `setText` writes plus the one intended description write, all targeted at the recipient's fresh replacement resolver. Executed proof on a local chain running the canonical ENS `ENSRegistry` + `PublicResolver` contracts: the mined transaction input is byte-identical to the app-built calldata (3,268 bytes, 10 `setText` calls), the recipient's new resolver settles with all nine attacker socials persisted, Portal renders seven of them as direct fixed-origin links to the attacker's profiles, and Manager promotes three into prominent fallback Contact cards. The production consequence is that any transferred or gifted `.eth` name can carry a persistent, attacker-authored content payload that the new owner's own ordinary save unknowingly re-authors on-chain under the new owner's authority and payment — at the exact moment the application affirmatively claims those old details are being cleared.

## Vulnerability Details

At audited public head `1c9b47f18fcddd2e864dfe385c4171061c9811ae` (tree `fd2fa034e987dd81c4a170b3e8370e6edf2a3376`, branch `audit-comp-ready`), the required invariant is: when the resolver-replacement flow tells the user that previous-owner details will be cleared and only their saved changes kept, fields the user did not touch must not be re-authored into the new resolver. Manager violates that invariant through a representation mismatch in its dirty-tracking, and the violation is triggered entirely by ordinary product flows.

#### Attack setup — the attacker is a third party, not the victim

The attacker is the ordinary previous owner of a `.eth` second-level name. Before transferring (or selling, or gifting) the name, they set one or more of the nine supported social text records in a non-canonical representation, using any ordinary record-write path (for example Portal's own Edit Records flow, which stores the entered value as-is). A non-canonical value is simply one that the Manager display layer rewrites: `https://twitter.com/attacker`, `x.com/attacker`, `@attacker` all normalize to `attacker` for `com.twitter`, and eight other social keys (Telegram, Farcaster, Instagram, GitHub, LinkedIn, Reddit, TikTok, Twitch) have equivalent normalizers. The attacker then transfers the name normally. Nothing about this preparation is anomalous: storing a full URL in a social record is a plausible, common user behavior, and the records are publicly visible before the transfer. No stolen credential, no local-state edit, no browser bug, no attacker-supplied calldata, and no pasted attacker address is required at any point.

The recipient later opens Manager's Edit Profile dialog for the acquired name. The dialog defaults to the General tab (`apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialogBody.tsx:128-132`, `defaultValue="general"`); the Contact tab that would display social records is a separate tab the recipient never needs to open. The recipient edits only `description`, clicks Save Profile, sees the clearing promise, and confirms `Replace & Save`.

#### Root cause: whole-form normalization before dirty-tracking, diffed against the raw on-chain snapshot

Stage 1 — the whole form is normalized before any dirty-tracking, irrespective of which fields the user touched (`apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialogBody.tsx:73-74`):

```tsx
{({ canSubmit, values }) => {
  const submittedValues = normalizeProfileRecords(values)
  const diff = createDiff(savedRecords, submittedValues)
```

`normalizeProfileRecords` rewrites every social record through its per-key display normalizer (`apps/manager/src/features/profile/utils/transformRecords.ts:78-93`):

```ts
const normalizeProfileTextRecords = (
  records: readonly TextRecordValue[],
): TextRecordValue[] =>
  records.map(({ key, value }) => ({
    key,
    value: getRecordDisplayValue(getRecordDef(key), value),
  }))

export const normalizeProfileRecords = (
  records: ProfileRecords,
): ProfileRecords => ({
  ...records,
  addresses: normalizeProfileAddresses(records.addresses),
  links: normalizeProfileLinks(records.links),
  social: normalizeProfileTextRecords(records.social),
})
```

Nine social record definitions carry a `normalize` function (`apps/manager/src/features/profile/data/records/text.tsx:45-171`, e.g. `com.twitter` at :45-56 uses `createSocialProfileValueNormalizer({ hosts: ['x.com', 'twitter.com'] })`); three further keys (`com.discord`, `com.youtube`, `com.mastodon`) have no normalizer and act as built-in controls in the PoC. The normalization is display-grade string surgery, not a canonicalization guarantee — the PoC demonstrates the Twitter normalizer is not even idempotent: `https://x.com/x.com%2Fx.com%252Fattacker` → `x.com/x.com%2Fattacker` → `x.com/attacker` → `attacker` over repeated passes.

Stage 2 — the save snapshot compares raw against normalized. The dialog state machine serializes `before` from the raw saved (on-chain) state and `after` from the submitted, already-normalized state (`apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialog.machine.ts:127-134`):

```ts
const getPendingSave = (
  savedRecords: ProfileRecords,
  currentRecords: ProfileRecords,
  deps: SaveDeps,
): PendingSave => {
  const before = transformToServiceFormat(savedRecords)
  const after = transformToServiceFormat(currentRecords)
```

On-chain text values enter `ProfileRecords` unchanged at ingestion (`apps/manager/src/features/profile/utils/transformRecords.ts:123-179`), so for the attacker's `com.twitter` record `before` holds the raw `https://twitter.com/attacker` while `after` holds the normalized `attacker` — even though the recipient never touched that field.

Stage 3 — every raw/normalized mismatch becomes a text change. `computeRecordChanges` (`apps/manager/src/features/profile/service/profileRecordTransactions.ts:158-181`) treats any `prev !== next` as a write:

```ts
for (const key of textKeys) {
  const prev = (beforeTexts.get(key) ?? '').trim()
  const next = (afterTexts.get(key) ?? '').trim()

  if (prev !== next) {
    textChanges.push({
      key,
      value: next === '' ? null : next,
    })
  }
}
```

The changes are passed to the ENSJS record-call builder and ABI-encoded into a resolver call (`apps/manager/src/features/profile/service/profileRecordTransactions.ts:372-457`). The executed harness shows this produces `setText(node, "com.twitter", "attacker")` and eight sibling writes the recipient never requested.

Stage 4 — the replacement-resolver flow batches those writes atomically with the resolver swap. A present-but-non-writable inherited resolver steers the dialog into resolver-setup mode (`apps/manager/src/features/profile/components/dialogs/edit-profile/useEditProfileDialogSave.ts:224-226`: `needsResolverSetup = !hasResolver || (Boolean(ownerAddress) && resolverWriteAccess.data === false)`). The setup path builds one Rhinestone intent whose call array ends with the record multicall aimed at the fresh resolver (`apps/manager/src/features/profile/service/setupControlledResolver.ts:96-134`):

```ts
const recordCalls = hasRecordsToWrite
  ? await buildRecordsUpdateCalls({ name, ... })
  : []
...
const calls = [
  <deploy owned resolver call>,
  buildSetResolverCall({ name, newResolver: resolver }),
  ...recordCalls,
]

const request: TransactionRequest = {
  type: 'rhinestone-intent',
  from: smartAccount,
  chainId,
  rhinestoneParams: { calls, feeAsset: 'USDC' },
}
```

Stage 5 — the confirmation affirmatively misrepresents the transaction. For the edit-profile case the dialog says (`apps/manager/src/features/profile/components/dialogs/ResolverSetupConfirmDialog.tsx:38-46`):

```tsx
.with('edit-profile', () => ({
  title: <Trans>Start a new profile for this name?</Trans>,
  description: (
    <Trans>
      This name still has profile details from a previous owner. Continuing
      clears them and keeps only the changes you save.
    </Trans>
  ),
  confirmLabel: <Trans>Replace & Save</Trans>,
}))
```

No field list is shown anywhere in the confirmation; the PoC harness asserts the rendered confirmation contains the clearing promise and does not contain `attacker` or `com.twitter`. The user is told old details will be cleared; the application's own transaction re-authors nine of them.

#### Display consequences after settlement

Once the writes settle, both in-scope applications present the attacker content as the recipient's own. Manager's profile view promotes copied socials into linkable fallback Contact cards — `getPrimaryContactItems` (`apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:170-194`) selects the first three non-empty contact/social records, which the executed run shows are Twitter, Telegram, and Farcaster with `href` values `https://x.com/attacker`, `https://t.me/attacker`, `https://farcaster.xyz/attacker`. Portal's `SocialRecord` component (`apps/portal/src/features/profile/components/SocialRecord.tsx:128-137`) displays the stored text and appends it to a fixed per-platform base URL (`baseUrls` at :19-29):

```tsx
<ExternalLink
  href={`https://${baseUrls[record.key]}/${record.value}`}
  ...
>
  <Icon record={record} /> {record.value}
</ExternalLink>
```

The executed reload stage, driven purely by values read back from chain, renders seven direct attacker-profile links from the recipient's name profile: `https://x.com/attacker`, `https://t.me/attacker`, `https://github.com/attacker`, `https://instagram.com/attacker`, `https://linkedin.com/in/attacker`, `https://warpcast.com/attacker`, `https://reddit.com/user/attacker`.

#### Runtime proof summary

The PoC (full steps in the Proof of Concept section) executes the real frozen Manager dialog, form body, state machine, confirmation dialog, resolver-setup service, and ENSJS record-call builder; captures the exact app-built record multicall; sends those exact bytes on a local Anvil chain (chain ID 11155111) running the canonical `ENSRegistry` + `PublicResolver` contracts; asserts the mined transaction input is byte-identical to the app-built calldata; reads the settled state back from chain; and pushes the chain-read values through the real Manager `transformProfileRecords` / `getPrimaryContactItems` code and the real Portal `SocialRecord` component. Three independent cold-start runs reproduce: run 1 and run 3 produce byte-identical on-chain end-state JSON (SHA-256 `5bb7b2d74eb7e43ff572b335a98da15db91e54063d46f640020cb4ab1becb43a`), and the app-built multicall artifact is byte-identical across all three runs (SHA-256 `7456b11f5ad44520e5716f4a0f69067607e8d2dd9c6881c7a7198dbdd2805db1`).

#### Evidence boundaries (full honesty list in the PoC section)

The Rhinestone HCA intent broadcast is not executed — the app-built call payloads are delivered as plain EOA transactions carrying the identical bytes the app hands to its transaction manager. The replacement resolver on-chain is the canonical v1 `PublicResolver` (ens-contracts @ `5141a2acdf77cf49f5134d674c2792b30f5ee0db`), substituted for the app's v2 owned permissioned resolver; both expose the `setText`/`multicall` interface the app bytes target. The inherited-resolver non-writability precondition is represented at the UI-trigger layer by the component harness (which drives the dialog down the real `needsResolverSetup` → Replace & Save path), not reproduced with a v2 permissioned resolver on-chain. Portal rendering is verified at component level (happy-dom), not against a live Portal deployment. None of these boundaries alter the demonstrated facts: the app builds the unintended writes, the confirmation conceals them while promising clearing, and the bytes settle exactly as built.

#### Recommended remediation

1. Canonicalize both sides of the comparison — apply the same normalization to the saved baseline and the submitted state before diffing — or, preferably, construct resolver-replacement record writes from explicitly user-touched fields rather than an object-wide derived diff.
2. Enumerate the exact record writes (key and value) in the `Replace & Save` confirmation so the clearing promise is verifiable against the transaction.
3. Make display/href normalization single-pass and idempotent (the demonstrated nested Twitter rewrite shows it is currently neither).
4. Regression coverage: a transferred name whose inherited resolver holds non-canonical socials must produce exactly one `setText` (the description edit) through the full dialog → machine → service → builder path; cover all nine normalizers and the three non-normalizing controls.

## Impact Details

Selected impact row (Websites & Apps, High): **"Injecting/modifying the static content on the target application without JavaScript (persistent), such as: - HTML injection without JavaScript - Replacing existing text with arbitrary text - Arbitrary file uploads, etc."**

Why this is the literal row, and why it is not self-inflicted stale-state replay: the attacker is a third party — the prior owner of the name — who plants the payload before the transfer using ordinary record writes, and the victim is the new owner who never sees it. The injected text is attacker-selected (any handle string the attacker chooses within the nine supported social keys), persistent (written on-chain to the recipient's own resolver by a mined transaction), involves no JavaScript (React-escaped plain text rendered as static profile content and static links), and replaces existing text with arbitrary attacker text in the recipient's profile (the recipient's profile ends up displaying the attacker's handles where the recipient's own or no handles should be). Both in-scope applications then render it: Portal turns seven of the carried keys into direct, fixed-origin links to the attacker's social profiles from the recipient's name page, and Manager promotes three into fallback Contact cards. The program's reflected-plain-text exclusion explicitly does not apply — it carves out reflected injection while preserving persistent plain-text injection, and this write is persistent on-chain state.

The deceit multiplier matters for impact assessment: the victim's save is an ordinary, prompted, in-app workflow (Edit Profile → edit description → Save Profile → Replace & Save), and the application's own confirmation affirmatively misrepresents the transaction at the decision moment — "Continuing clears them and keeps only the changes you save" — while the transaction the app builds does the opposite for nine fields. The carried records are invisible to the recipient at decision time (the dialog defaults to the General tab; the Contact tab showing socials is never opened; the confirmation enumerates nothing), so repair requires first discovering a hidden write the app claimed not to make. The recipient also pays for the poisoned transaction (the flow charges a USDC fee through the recipient's HCA intent), meaning the victim funds the publication of the attacker's content under their own name ownership.

Honest limits, stated plainly: the injected content is React-escaped plain text — no HTML injection, no script execution, no XSS is claimed. Rendered links stay on the configured fixed social-platform origins (`x.com`, `t.me`, `github.com`, `instagram.com`, `linkedin.com/in`, `warpcast.com`, `reddit.com/user`), so this is not an open redirect to an arbitrary domain. The recipient can delete the records after discovering them. No funds, name ownership, NFT, address record, or authority changes hands — the name, its ownership, and its resolver pointer all settle exactly as the product intends; only the textual profile content is attacker-authored. The attack requires a transferred or gifted name whose inherited resolver the recipient cannot write (the Replace & Save path), plus one ordinary profile save by the recipient. Within those limits, the demonstrated behavior is precisely the High row: persistent, JavaScript-free replacement of static profile text with attacker-arbitrary text on the target applications, authored by a third party and published by the victim's own routine action.

## References

* In-scope asset: `apps/manager` (Manager app, Websites & Apps): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* Whole-form normalization before dirty-tracking: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialogBody.tsx>
* Per-key display normalizers applied to every social record: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/utils/transformRecords.ts>
* Nine social records with `normalize` (three controls without): <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/data/records/text.tsx>
* Raw-before / normalized-after save snapshot: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialog.machine.ts>
* Non-writable inherited resolver steering into replacement mode: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/dialogs/edit-profile/useEditProfileDialogSave.ts>
* Clearing-promise confirmation that enumerates no carried fields: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/dialogs/ResolverSetupConfirmDialog.tsx>
* `prev !== next` text-change sink and ENSJS call construction: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileRecordTransactions.ts>
* Batched replacement-resolver intent (deploy + setResolver + record multicall, USDC fee): <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/setupControlledResolver.ts>
* Manager fallback Contact cards: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileView.helpers.ts>
* Portal fixed-origin social links from stored text: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/components/SocialRecord.tsx>
* Official competition scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* ENS text-record standard (social keys such as `com.twitter` are ENSIP-5 global keys): <https://docs.ens.domains/ensip/5>

Public-state refresh on 2026-09-06: `git ls-remote https://github.com/immunefi-team/audit-comp-ens` shows `audit-comp-ready` still at `1c9b47f18fcddd2e864dfe385c4171061c9811ae` (the exact revision executed by the PoC) and a single pull ref — PR #1 (`d35fbdf827f37d7b22ad7cd347342ad425250163`, a Hono dependency bump touching only `apps/manager/package.json`, `packages/dqa-server/package.json`, `pnpm-lock.yaml`, and `workers/api-worker/package.json`), unrelated to the profile normalization, state machine, resolver setup, record transaction, confirmation, or display paths. Private-report overlap remains unknowable and is not claimed either way.

## Link to Proof of Concept

<https://gist.github.com/Jornason/3814160116a18d15ac83e65320b9c0c0>

## Proof of Concept

## Proof of Concept

### What this PoC proves

The linked secret Gist bundle (<https://gist.github.com/Jornason/3814160116a18d15ac83e65320b9c0c0>) preserves an executed end-to-end reproduction (run tag r8h3v5) against the frozen public competition head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. It drives the real Manager Edit Profile dialog, form body, XState machine, `ResolverSetupConfirmDialog`, `setupControlledResolver` service, and ENSJS record-call builder (all unmodified frozen application code); captures the exact record multicall the app constructs when the recipient edits only `description` and confirms `Replace & Save`; executes those exact bytes on a fresh local Anvil chain (chain ID 11155111) running canonical ENS `ENSRegistry` + `PublicResolver` contracts compiled from ens-contracts @ `5141a2acdf77cf49f5134d674c2792b30f5ee0db`; asserts the mined transaction input is byte-identical to the app-built calldata; reads every text key back from both resolvers; and feeds the chain-read values through the real Manager `transformProfileRecords` / `getPrimaryContactItems` and the real Portal `SocialRecord` component. It proves the recipient's ordinary save re-authors nine attacker-selected social records into the recipient's replacement resolver while the app's confirmation promises old details are cleared, and that both applications then render the attacker content as the recipient's profile.

All writes are local to a disposable Anvil chain. No public Sepolia transaction is broadcast, no production user, site, or deployed contract is contacted or modified, and every key used is a public Anvil deterministic test account.

### Prerequisites

* A clean checkout of the official repository at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` with locked dependencies installed (`corepack enable && pnpm install --frozen-lockfile`; the completed runs used pnpm with Node.js v22.17.1 on macOS arm64).
* Foundry (`anvil`, `cast`, `forge`; the completed runs used 1.5.x-stable), `jq`, and `shasum`.
* Free loopback port for Anvil (the examples use 8546).

### Step-by-step reproduction

1. Download the secret Gist files into one directory and verify the manifest in the Gist root:

```bash
shasum -a 256 -c SHA256SUMS.txt
```

2. Prepare the frozen checkout and stage the two harness test files (they are plain vitest tests added under a new directory; no tracked file is modified):

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git ens-audit-r8h3v5
git -C ens-audit-r8h3v5 checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
cd ens-audit-r8h3v5 && corepack enable && pnpm install --frozen-lockfile && cd ..
mkdir -p ens-audit-r8h3v5/apps/manager/src/__poc_r8h3v5__
cp manager-profile-social-carryover-r8h3v5.test.tsx \
   manager-profile-social-carryover-onchain-r8h3v5.test.tsx \
   ens-audit-r8h3v5/apps/manager/src/__poc_r8h3v5__/
```

3. Build the canonical-contract environment (one time). The script clones ens-contracts @ `5141a2acdf77cf49f5134d674c2792b30f5ee0db`, copies the needed source subset into `contracts/poc-env/src/`, vendors `openzeppelin-contracts` v4.9.3, `@ensdomains/buffer` 0.1.1, and `@ensdomains/solsha1` 0.0.3 into `contracts/poc-env/lib/`, writes `foundry.toml` (solc 0.8.26, cancun), and runs `forge build`:

```bash
bash setup-poc-env-r8h3v5.sh
```

4. Run the full seamless lifecycle (stages: real-dialog harness → contract build check → fresh Anvil → on-chain lifecycle with byte-identity assertion → reload render stage):

```bash
FROZEN_DIR=/absolute/path/to/ens-audit-r8h3v5 bash run-hardened-poc-r8h3v5.sh run1 8546
```

Expected terminal output includes: the app-harness stage `Tests 3 passed (3)`; `[*] app-built multicall mined (tx 0x525192aec4e9a5b7c10ae1ee733e73b16f756c537c58d72c8465d3f9047dec33 status=0x1)`; `[*] mined tx input is byte-identical to the app-built multicall`; the reload stage `Tests 3 passed (3)`; and `=== DONE run1`.

5. Independently reproduce from a cold start (fresh Anvil process, new port), and confirm the end state is byte-identical to the reference artifacts:

```bash
pkill -f "anvil --port" 2>/dev/null || true
FROZEN_DIR=/absolute/path/to/ens-audit-r8h3v5 bash run-hardened-poc-r8h3v5.sh run2 8547
shasum -a 256 artifacts/app-built-multicall.run1.json artifacts/app-built-multicall.run2.json
# both: 7456b11f5ad44520e5716f4a0f69067607e8d2dd9c6881c7a7198dbdd2805db1
```

### Executed on-chain lifecycle (what the run does)

1. Deploy canonical `ENSRegistry`, canonical `ReverseRegistrar` (required by `PublicResolver`'s `ReverseClaimer` constructor), and two canonical `PublicResolver` instances on fresh Anvil (chain ID 11155111).
2. The attacker (prior owner, Anvil account 0) registers `recipient.eth` (node `0x7a37a174771552ce8189e64ba2e97cde57062f1577fa0a103dacd3e3ca4e1913` — enforced identical to the node embedded in the app-built calldata), seeds 13 raw baseline records on the old resolver (9 non-canonical socials such as `com.twitter = "https://twitter.com/attacker"`, 3 non-normalizing controls `com.discord`/`com.youtube`/`com.mastodon = "attacker-control"`, and `description = "previous-owner description"`), then transfers ownership to the recipient.
3. The recipient points the name at the fresh replacement resolver (`setResolver`, tx `0xb5b3af124bbcd7432f556164d30695aff6edba77fd792613133fe48e8638b16e`) and sends the exact app-built multicall bytes (tx `0x525192aec4e9a5b7c10ae1ee733e73b16f756c537c58d72c8465d3f9047dec33`, status success). The script asserts the mined transaction input equals the app calldata byte-for-byte and aborts otherwise.
4. End state is read back from chain into `onchain-end-state.<run>.json`.

### On-chain end state (run 1 == run 3, byte-identical)

Reference artifact `onchain-end-state-run1-r8h3v5.json` (SHA-256 `5bb7b2d74eb7e43ff572b335a98da15db91e54063d46f640020cb4ab1becb43a`; run 3 artifact `onchain-end-state-run3-r8h3v5.json` has the identical hash):

* New (replacement) resolver `0xf6168876932289d073567f347121a267095f3dd6`, settled owner = recipient `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`.
* `new_resolver_texts_after`: all nine attacker socials persisted in normalized form — `com.twitter`, `org.telegram`, `xyz.farcaster`, `com.instagram`, `com.github`, `com.linkedin`, `com.reddit`, `com.tiktok`, `com.twitch` = `"attacker"` — plus the recipient's intended `description = "recipient-authored description"`.
* The three non-normalizing control keys are empty on the new resolver (`""`) — proving the carryover is specifically the normalization-mismatch mechanism, not a blanket copy.
* `old_resolver_texts_after`: the inherited resolver still holds the raw baseline (`com.twitter = "https://twitter.com/attacker"`, `description = "previous-owner description"`, controls = `"attacker-control"`) — the "clearing" the confirmation promised does not delete anything anywhere; it re-authors selected attacker content into the new resolver.
* `mined_input_byte_identical_to_app_calldata: true`; the 3,268-byte `multicall(bytes[])` decodes to exactly ten `setText` calls (nine socials + description), visible in `decoded-calldata-r8h3v5.json` and `app-built-multicall-run1-r8h3v5.json`.

### Render consequences from chain state

The reload stage (3/3 tests pass) consumes only the chain-read JSON. Manager's real `transformProfileRecords` + `getPrimaryContactItems` promote `com.twitter`, `org.telegram`, `xyz.farcaster` to the three fallback Contact cards with hrefs `https://x.com/attacker`, `https://t.me/attacker`, `https://farcaster.xyz/attacker`. Portal's real `SocialRecord` component renders the seven Portal-supported carried keys as direct links, captured in `portal-links-run1-r8h3v5.json`: `https://x.com/attacker`, `https://t.me/attacker`, `https://github.com/attacker`, `https://instagram.com/attacker`, `https://linkedin.com/in/attacker`, `https://warpcast.com/attacker`, `https://reddit.com/user/attacker`.

### App-layer harness results (component level, real frozen code)

`manager-profile-social-carryover-r8h3v5.test.tsx` (3/3 pass, log in `manager-profile-social-carryover-r8h3v5.vitest.out`) proves with the real dialog and services: only `base.description` was changed; the rendered confirmation contains the clearing promise and does not contain `attacker` or `com.twitter`; the app-built call set contains exactly the deploy + setResolver + record multicall; the multicall decodes to the ten writes above; the three control keys are absent; and the canonical-baseline control case (records already stored as `attacker`) emits only the single description write — confirming the raw/normalized mismatch is the trigger. A third test documents that the Twitter normalizer is not idempotent (`https://x.com/x.com%2Fx.com%252Fattacker` → `x.com/x.com%2Fattacker` → `x.com/attacker` → `attacker`).

### Honest simplifications (nothing implied beyond this list)

1. The Rhinestone HCA intent broadcast (smart-account execution + USDC fee) is not executed on-chain; the app-built call sequence is delivered as plain EOA transactions. The record-write bytes themselves are the app's exact output — the harness captures the literal `rhinestoneParams.calls` the app hands to its transaction manager, and the mined input is asserted byte-identical to those bytes.
2. The replacement resolver on-chain is the canonical v1 `PublicResolver`, not the app's v2 owned permissioned resolver (VerifiableFactory / DedicatedResolverImpl). Both expose the `setText`/`multicall` interface the app bytes target (ensjs `publicResolverSetTextSnippet` / `publicResolverMulticallSnippet`); authorization semantics differ (v1 follows registry node ownership, v2 is role-based).
3. The "inherited resolver not writable by the recipient" precondition is not reproduced on-chain (with canonical contracts, node-ownership transfer makes the recipient authorized on the inherited resolver). It is represented at the UI-trigger layer: the app-layer harness drives the real dialog down the real `needsResolverSetup` → Replace & Save path with a mocked non-writable resolver probe.
4. The resolver-pointer call executed on-chain is v1 `setResolver` on `ENSRegistry`, whereas the app builds `buildSetResolverCall` against the v2 registry stack — same semantic, different contract generation; the record-write evidence (the vulnerable payload) is unaffected.
5. Portal rendering is verified at component level in vitest (happy-dom) through the real `SocialRecord` source, not in a browser against a live Portal deployment; record reads were direct `eth_call`s, which is what the apps' record layer reduces to for resolver text reads.

### Bundle contents and integrity

| File                                                                       | Content                                                                                                                                   |
| -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `run-hardened-poc-r8h3v5.sh`                                               | Single-entry orchestrator (5 stages), prints artifact hashes                                                                              |
| `onchain-lifecycle-r8h3v5.sh`                                              | Deploy → attacker seed → transfer → recipient setResolver + app bytes → read-back, with byte-identity assertion                           |
| `setup-poc-env-r8h3v5.sh`                                                  | One-time canonical-contract environment builder (pinned commits/versions)                                                                 |
| `manager-profile-social-carryover-r8h3v5.test.tsx`                         | App-layer harness (real dialog/machine/confirmation/services), SHA-256 `2a3a36fbaf3885bdddcb8504bb73aa893f42c7db8e5fec72c7eff43ea8f90e28` |
| `manager-profile-social-carryover-onchain-r8h3v5.test.tsx`                 | Reload-stage harness (chain state → real Manager + Portal render code)                                                                    |
| `manager-profile-social-carryover-r8h3v5.vitest.out`                       | Captured app-layer harness output (3/3 pass, decoded writes)                                                                              |
| `decoded-calldata-r8h3v5.json`                                             | Full `multicall(bytes[])` calldata + 10 decoded `setText` writes + control case                                                           |
| `app-built-multicall-run1-r8h3v5.json`                                     | Captured app-built record call (SHA-256 `7456b11f…5db1`, identical across 3 runs)                                                         |
| `onchain-end-state-run1-r8h3v5.json`, `onchain-end-state-run3-r8h3v5.json` | Chain-read end state, byte-identical across cold starts (SHA-256 `5bb7b2d7…b43a`)                                                         |
| `portal-links-run1-r8h3v5.json`                                            | Seven direct attacker-profile links rendered by the real Portal component                                                                 |
| `README-r8h3v5.md`                                                         | Full reproduction guide and environment notes                                                                                             |
| `SHA256SUMS.txt`                                                           | Integrity manifest covering every bundle file (verifies in the Gist root)                                                                 |

### Safety limits

* The production applications, deployed ENS contracts, and public Sepolia are never contacted for writes; everything executes on disposable local Anvil instances.
* All accounts are public Anvil deterministic test accounts; no real wallet key, user credential, or production data is required or included.
* The frozen competition checkout is never modified: the harness adds two new untracked test files under `apps/manager/src/__poc_r8h3v5__/` and changes no tracked file (the original runs verified `git status --porcelain` shows zero tracked modifications).
* No phishing, no social engineering, no third-party service interaction; the demonstration uses only local processes.
