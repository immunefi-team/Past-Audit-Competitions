# 92300 w a critical portal lists an attacker owned name under names you own in place of the user s own and transferring it transfers the user s real name instead

**Submitted on Sep 8th 2026 at 19:59:53 UTC by @SaiyanVegeta for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92300
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Bug Description

**Brief.** Portal identifies a name by the raw path string in five places (search listing, routing, page chrome, transfer form, transaction label) and by its **ENSIP-15 normalised** form in the three places that matter (authorisation, detach planning, token id). An attacker who registers the non-normalised twin of a user's name — `Nick896607.eth` against `nick896607.eth` — and gifts it to that user achieves two things with one registration. First, Portal's own search **replaces** the user's name with the twin under *"Names you own"*: the two collide on a case-insensitive dedup key and only the first survives. Second, every write reachable from that listing is computed on the **normalised** name. The user opens what the interface calls their name, asks to transfer it, and Portal builds `safeTransferFrom` for the token id of the *canonical* name. The name the user believes they are giving away is worthless; the one that actually leaves is theirs.

**Details (verified at HEAD).**

*Step 1 — the display defect, complete before any transaction exists.*

The indexer returns both names for the owner, twin first:

```
domains(where:{owner:"0xf39f…2266"}) → [ {"name":"Nick896607.eth"}, {"name":"nick896607.eth"}, … ]
```

* `apps/portal/src/features/dashboard/hooks/useV2NamesForAddress.ts:29-37` issues exactly that query and returns `name` verbatim.
* `apps/portal/src/features/dashboard/utils/ownedNamesUtils.ts` (`mergeOwnedNames`) then deduplicates:

  ```ts
  const key = d.name.trim().toLowerCase()
  if (!byName.has(key)) byName.set(key, d)
  ```

  `Nick896607.eth` and `nick896607.eth` share the key `nick896607.eth`. **The first wins and the second is discarded.** Order comes from the indexer, and the gift is the most recent acquisition.
* `filterAndSortOwnedNames` in the same file matches case-insensitively (`d.name.trim().toLowerCase().includes(matchQuery)`), so the user typing their **own** name reaches the surviving row.

Measured in the running Explorer: the user types `nick896607`, and *"Names you own"* renders **one row — `Nick896607.eth`** (screenshot `3-search-for-own-name.png`). Their real name is not in that section at all.

*Step 2 — routing accepts the non-normalised name.*

* `apps/portal/src/routes/$name.tsx:29`:

  ```ts
  if (isValidEnsName(name) && isTLD(name)) {
  ```

  `isValidEnsName` is imported in this file and used **only in conjunction with `isTLD`**. A non-normalised, non-TLD name is therefore never checked. `apps/portal/src/routes/$name/ownership/transfer.tsx` contains no occurrence of `isValidEnsName`, `getLabel` or any normalisation call.

*Step 3 — authorisation is answered about a different name.*

* `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:55-63`:

  ```ts
  // getLabel normalises and can throw on a malformed name. A name we can't parse
  // is one we can't check, so treat it as "not transferable" rather than crash.
  const label = useMemo(() => { try { return getLabel(name) } catch { return null } }, [name])
  ```

  and the query below it calls `hasRoles(registryAddress, label, TRANSFER_ROLE, account)`. The comment anticipates `getLabel` **failing** on a malformed name. It does not anticipate `getLabel` **succeeding on a different name**. `getLabel('Nick896607.eth')` returns `nick896607` — so the permission question Portal asks the chain is *"may this account transfer `nick896607.eth`?"*, and the answer is yes. The form enables.

*Step 4 — the write leg carries the canonical token id.*

* `apps/portal/src/features/transfer/hooks/useTransferName.ts:112-115`:

  ```ts
  getEnsTokenId({ label: getLabel(name), registryAddress })
    .map((tokenId) => ({ recipient, tokenId, options }))
  ```

  and `:132` (`const label = getLabel(name)`) for the detach steps. Both twins live in the same registry (`sameRegistry: true` in the PoC), so the canonical token id is a live, transferable asset at that address.

*Step 5 — nothing in the confirmation names the asset that moves.*

The transaction label keeps the **raw** string:

* `apps/portal/src/features/transfer/hooks/useTransferName.ts:174-177` — `` transactionName: `${STEP_LABELS[step]} - ${name}` ``

Captured verbatim from the modal during the run, in order:

```
Transaction overview / Transfer name / Not Started / Est. Cost / Not yet / Start / Close
Transfer name / Transfer name - Nick896607.eth / Open wallet / Close
Transfer name / Transfer name - Nick896607.eth / Done / Close
```

The screen the user reads **at the moment of approval names no name at all**. The only name shown anywhere in the flow — sidebar, page heading, breadcrumb, step label, and the application's own transaction id `transfer-Nick896607.eth-transfer-token` — is the twin. The canonical name appears nowhere: asserted, not observed loosely (`pageNamesTheCanonical: false`, `anyFrameNamesTheCanonical: false`).

*The precondition is already established on the in-scope deployment.* The v2 ETHRegistrar performs no ENSIP-15 normalisation; `valid()` / `isAvailable` are length-only. Uppercase, fullwidth, soft-hyphen, zero-width-space and emoji-variant twins all register successfully against the real deployed bytecode (`PROOF-v2-registrar-accepts-nonnormalized-labels.md`). Registering the twin is an ordinary registration at ordinary cost.

***

## Impact

**The application displays a name that is not the user's, in the one place that exists to list the user's names, and every action from there acts on the user's real name.**

The user's interaction is the most ordinary one the Explorer supports: search for your own name, open it, transfer it. They never leave the genuine origin, never follow a link from anyone, never see a malformed string presented as anything unusual, and never approve anything but the operation they asked for. The transaction they approve is described to them as being about `Nick896607.eth`, which is exactly what Portal told them they own.

What is actually broadcast is `safeTransferFrom(user, attacker, 9747831…673664, 1, 0x)` — the token id of `nick896607.eth`. Measured immediately after: the user holds **0** of the canonical name and still holds **1** of the twin; the attacker holds **1** of the canonical name.

Two things make this different from a mislabelled screen:

1. **The user's own name is removed from the listing.** This is not a case of two similar rows where the user picks the wrong one. `mergeOwnedNames` drops the canonical entry entirely, so *"Names you own"* offers a single row and it is the attacker's. There is no correct row to choose.
2. **What leaves is the valuable name, not the decoy.** The transfer is applied to the normalised name — the one that resolves, that wallets and clients use, and that the user actually paid for. The attacker keeps nothing of their own: they gave the twin away deliberately, and it stays with the victim.

Cost to the attacker is one registration of a name nobody wants, plus the gas to gift it. There is no wallet compromise, no phishing page, no malicious indexer (Panoptes returns both names correctly — the discard happens in the app), and no contract defect: the registry executes precisely the token id Portal asked it to move.

***

## Risk Breakdown

**Which clause I rely on, and what I deliberately do not lean on.** I rely on **"Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient."** The incorrect display is measured *before any transaction exists*: `mergeOwnedNames` discards the user's own name and Portal renders the attacker's in its place under a heading that asserts ownership. The transfer is offered as the realised consequence, to show the display error is load-bearing rather than cosmetic — not to carry the eligibility argument.

I do **not** rest this on the wallet-interaction clause. Portal composes the calldata faithfully for the name it has decided the page is about; the defect is that it decided wrongly, and it decided wrongly on screen first.

**No contract or indexer behaviour is implicated.** The registrar accepts non-normalised labels by design — that is settled and not claimed as a defect here. Panoptes returns **both** names, correctly, for the owner; I verified this directly against the indexer's own GraphQL endpoint. The registry moves exactly the token id it is given. Every step of the failure occurs inside `apps/portal`, and the fix is four lines in `apps/portal`.

**How this differs from my earlier report P2, which was closed.** I flag the overlap rather than let triage find it. P2 concerned the *extend* flow and was closed on the ground that *the name lost is the non-canonical twin, which no client resolves, so the loss is a bounded fee*. That reasoning does not reach this finding, and the direction is reversed: here the twin is what the user **keeps**, and the canonical name — the one that resolves — is what leaves. The loss is the asset itself, not a fee. The mechanism is also different: P2 was a URL-driven label divergence in a renewal path; this begins in the owned-names listing, which no earlier report touched.

**Anticipated objection: the user approved a transfer.** They did, and that is the substance of the finding rather than an answer to it. Consent attaches to the asset the interface named. Every surface in the flow named `Nick896607.eth`: the section heading that asserts ownership, the sidebar, the page heading, the step label, and the app's internal transaction id. The approval screen itself named nothing. A user cannot withhold consent from an asset they are never shown.

**Anticipated objection: they should have noticed the capital N.** The comparison is not available to them. The listing shows one row, not two — their own name has been discarded by the dedup — so there is nothing on screen to compare against. The same substitution is reachable with a fullwidth character, a soft hyphen or a zero-width space, all proven to register against the real deployed registrar, and none of which renders as a visible difference at all.

**Not a Known Issue.** Checked against *ENS Apps: Known Security Issues* (2026-08-14): all 25 Part 1 findings, all 10 QA items, both tracker lists. No entry concerns the owned-names listing, `mergeOwnedNames`, case-insensitive dedup of name lists, or the absence of a normalisation gate on `routes/$name.tsx`. **QA-01 / WEB-1257** concerns transfers that error afterwards; nothing errors here. **WEB-716** is a missing action, not a misdirected one.

**Honest bounds.** Measured on an isolated local fork of the real deployed Sepolia v2 contracts, in the unmodified Explorer UI, against the repository's own Panoptes API — no shim, stub or substituted endpoint, and nothing sent to Sepolia. The dedup outcome depends on the order in which the indexer returns the two names; in every run the most recently acquired name came first, which is the gift. If that order were reversed the user's own name would survive the dedup and the twin would vanish from the listing — the transfer defect (steps 2–5) is unaffected either way, and remains reachable by a direct link. I did not audit whether the same raw/normalised split affects other flows reachable from `routes/$name`; I measured the transfer path only. No affected production source file was modified; the audited commit is `1c9b47f`, and `git status` is clean at that commit after every run.

***

## Recommendation

In order of importance:

1. **Gate the route.** `apps/portal/src/routes/$name.tsx` already imports `isValidEnsName`. Use it:

   ```ts
   if (!isValidEnsName(name)) throw notFound()
   ```

   This is the negative control in the PoC below: with it applied the page 404s and no transaction is built.
2. **Never key a name list on a lossy transform.** `mergeOwnedNames` must deduplicate on the identity the chain uses — the token id, or the registry-plus-label pair — not on `name.toLowerCase()`. Two distinct on-chain assets must never collapse into one row.
3. **Do not let a write and its label disagree about their subject.** `useTransferName` should carry one name value end to end. If `getLabel(name) !== name`, that is a mismatch the flow must refuse, not silently resolve in favour of the normalised form.
4. **Say what is being transferred, in the confirmation.** `Transfer name` with no subject is not a confirmation. The modal should name the asset — and a test should assert that the name shown equals the name whose token id is in the calldata.

***

## References

* `apps/portal/src/features/dashboard/utils/ownedNamesUtils.ts` — `mergeOwnedNames` (dedup on `name.trim().toLowerCase()`), `filterAndSortOwnedNames` (case-insensitive match)
* `apps/portal/src/features/dashboard/hooks/useV2NamesForAddress.ts:29-37` — `domains(where:{owner}) { name }`
* `apps/portal/src/features/dashboard/hooks/useSearchResults.ts` → `HomeSearchInput.tsx`, `SearchModalContent.tsx` — where the listing renders
* `apps/portal/src/routes/$name.tsx:29` — `isValidEnsName(name) && isTLD(name)`, the only use of the guard
* `apps/portal/src/routes/$name/ownership/transfer.tsx` — no normalisation of any kind
* `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:55-73` — authorisation asked about `getLabel(name)`
* `apps/portal/src/features/transfer/hooks/useTransferName.ts:112-115`, `:132`, `:174-177` — canonical token id, canonical detach label, raw transaction label
* `apps/portal/src/utils/token/getLabel.ts` — `ens_split(ens_normalize(name))[0]`
* `PROOF-v2-registrar-accepts-nonnormalized-labels.md` — the v2 ETHRegistrar registers non-normalised labels against real deployed bytecode

***

## Proof of Concept

## Proof of Concept

All testing is local (competition rule): an isolated local Anvil fork of the **real deployed Sepolia v2 contracts** (`:8546`, keyless public endpoint), the repository's **own Panoptes indexer and API** (`:5658`), and the unmodified Explorer served by its own dev server, driven through the project's `headless-web3-provider` fixture. **No indexer shim, stub or substituted response was used at any point.** Nothing was sent to Sepolia; no production system, third-party account or wallet was touched.

```
victim  (the Explorer's own mock wallet) : 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
attacker                                 : 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
registry (both names)                    : 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
victim's real name                       : nick896607.eth   token 9747831014882167876719679852534767402565462812997305512221194764843602673664
attacker's twin, gifted to the victim    : Nick896607.eth   token 66501685653505095991655359344416086218726060760385220263857699115559280443392
```

### Result

```json
{"searchedFor":"nick896607",
 "namesYouOwn":["Nick896607.eth"],
 "urlOpened":"/Nick896607.eth/ownership/transfer",
 "routeRenderedTransferForm":true,
 "pageNamesTheTwin":true,
 "pageNamesTheCanonical":false,
 "approvalScreenAsShown":["Transaction overview","Transfer name","Not Started","Est. Cost","Not yet","Start","Close"],
 "anyFrameNamesTheTwin":true,
 "anyFrameNamesTheCanonical":false,
 "consoleTransactionIds":["transfer-Nick896607.eth-transfer-token"],
 "minedTransfersToRegistry":[{"id":"9747831014882167876719679852534767402565462812997305512221194764843602673664",
                              "to":"0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
                              "hash":"0x4787dc81555615b055e1dc904b72eb1818aac1f29a152c5cb82f0447d9fec265"}],
 "canonicalTokenId":"9747831014882167876719679852534767402565462812997305512221194764843602673664",
 "rawTwinTokenId":"66501685653505095991655359344416086218726060760385220263857699115559280443392",
 "victimCanonicalAfter":"0","victimRawTwinAfter":"1","attackerCanonicalAfter":"1"}
```

Screenshots, taken by the spec during the run: `3-search-for-own-name.png` (the single decoy row under *"Names you own"*), `1-transfer-page-for-raw-twin.png`, `2-modal-frame-2.png`.

### Assertions, in order

1. **Clean start, read from chain.** The victim holds `1` of the canonical token and `1` of the twin; the two token ids are asserted to differ.
2. **The listing substitutes the decoy.** With the victim's wallet connected, they type their own label into the Explorer's search. *"Names you own"* renders exactly one row: `Nick896607.eth`. That the discard happens in the app, not upstream, is established separately: the same query the hook issues, run directly against the indexer's GraphQL endpoint, returns **both** names for that owner (transcript in `T42-FINAL-RUN.md`).
3. **The route is ungated.** `/Nick896607.eth/ownership/transfer` renders the full *Transfer ownership* form for a name ENSIP-15 rejects.
4. **The page names only the twin.** `pageNamesTheTwin` true, `pageNamesTheCanonical` false, asserted over the whole document text.
5. **Authorisation passes.** The *Transfer name* button becomes enabled — `hasRoles` was answered about the canonical name.
6. **The approval screen names no asset.** Captured verbatim before any click: `Transaction overview / Transfer name / Not Started / Est. Cost / Not yet / Start / Close`.
7. **Every subsequent frame names the twin, never the canonical name.** Asserted across all modal frames, and on the app's own transaction id `transfer-Nick896607.eth-transfer-token`.
8. **Exactly one transfer is sent to the registry in the measured window, and it carries the canonical token id.** Every block from before the page load to after the mine is scanned and the calldata decoded; `toHaveLength(1)`, and the id equals the canonical token, not the twin.
9. **The loss is final.** Victim: canonical `0`, twin `1`. Attacker: canonical `1`.

### Negative control

The same state, the same URL, the same gestures, with one line added to `apps/portal/src/routes/$name.tsx`:

```ts
if (!isValidEnsName(name)) throw notFound()
```

| Guard    | Control spec                                                                | Positive spec                          |
| -------- | --------------------------------------------------------------------------- | -------------------------------------- |
| applied  | **passes** — page 404s, `0` transfers sent, victim keeps the canonical name | (not run)                              |
| reverted | fails, as expected                                                          | **passes** — the canonical name leaves |

The patch was reverted immediately after the run and `git status` verified clean at `1c9b47f`.

### Reproduction

1. Start the isolated stack — anvil forked on `:8546`, plus the repository's Panoptes indexer and API on `:5658`.
2. `node setup-t42-transfer-twin.mjs` — registers `nick<ts>.eth` to the victim with a resolver, registers the raw twin `Nick<ts>.eth` to the attacker with no resolver and no subregistry (so the transfer plan has no cleanup steps), and gifts the twin to the victim using the **raw** token id. Writes `t42-state.json`.
3. Start the Explorer with `VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:5658/graphql` and `VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8546`.
4. Run the two specs in order:

   ```
   PORTAL_APP_URL=http://localhost:<port> ANVIL_RPC_URL=http://127.0.0.1:8546 \
     npx playwright test --config=projects/portal/playwright.config.ts \
     --workers=1 -g "T42 delivery|T42 — Portal"
   ```

Result: **2 passed**, archived verbatim in `T42-FINAL-RUN.txt`. To see the substitution by hand: register the uppercase twin of any name you own, send it to yourself, and search your own label — the row under *"Names you own"* will be the twin, and transferring it will move the name you actually own.
