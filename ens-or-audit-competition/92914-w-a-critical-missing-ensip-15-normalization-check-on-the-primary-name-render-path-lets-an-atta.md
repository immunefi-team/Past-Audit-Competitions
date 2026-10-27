# 92914 w a critical missing ensip 15 normalization check on the primary name render path lets an attacker s address be displayed as another party s name

**Submitted on Sep 13th 2026 at 07:44:19 UTC by @Magnum for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92914
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Explorer renders the ENS name of an arbitrary address without ever checking that the name is ENSIP-15 normalized. That name is read from a reverse registrar, where its value is written by the owner of the address being displayed, so it is attacker-authored input. An attacker can therefore register a visual twin of a well-known name, point it at their own address, and have the application present their address to every other user as that name. In three of the nine render sites the name replaces the address entirely, so the viewer has nothing left to compare against. In production this turns any Explorer surface that lists addresses — transaction history, the profile owner field, the events table, the transfer recipient preview, the registry role sheet — into an identity-spoofing surface, where a user approves a name transfer, grants a role, or trusts a counterparty on the strength of a name the attacker authored.

## Vulnerability Details

### The write side accepts an un-normalized name

`packages/l2-primary/src/utils/setReverseName.ts::createSetReverseNameRequest` passes the caller's string directly into the contract call:

```ts
if (targetAddress) {
  return {
    address: registrarAddress,
    abi: l2ReverseRegistrarSetNameForAddrSnippet,
    functionName: 'setNameForAddr',
    args: [targetAddress, name] as const,
  }
}
return {
  address: registrarAddress,
  abi: l2ReverseRegistrarSetNameSnippet,
  functionName: 'setName',
  args: [name] as const,
}
```

There is no normalization and no validation. The same is true of writing an L1 reverse record directly on-chain, which requires no application involvement at all.

### The read side verifies ownership but not normalization

`apps/portal/src/features/profile/hooks/usePrimaryName.ts`:

* line 40 — ensjs `getName` performs the L1 default reverse (coin 60); the result is accepted when `match === true` (line 43)
* line 76 — the ENSIP-19 fallback reads `nameForAddr(address)` from each L2 reverse registrar in `L2_FALLBACK_COIN_TYPES = [10, 42161, 8453, 59144, 534352]`, then forward-verifies against `addr(name, coinType)` and returns on a match (line 95)

Both branches return the registrar string exactly as read. `isNormalized` is never called.

Forward verification is not a defence here and this report does not claim it is bypassed. The attacker genuinely owns the homograph name and sets its address record to their own address, so ENSIP-19 verification passes correctly. The missing step is the normalization check after verification.

The same applies to wagmi's `useEnsName`: viem's `getEnsName` calls `reverse(bytes)` on the UniversalResolver and returns the name verbatim. The module contains no reference to normalization.

### The render side prints it, and often instead of the address

`apps/portal/src/components/table/EventsDataTable/AddressDisplay.tsx:29-30`:

```ts
const displayName = ensName || (short ? truncateAddress(address) : address)
const variant = ensName ? 'name' : variantProp
```

When a name exists the address is not rendered at all. The second line is worth attention on its own: the badge relabels the entity from `Address` to `Name`, so the application is not merely printing a string, it is asserting that the value is a name. `EntityBadge` applies no normalization either.

### The check exists in this application and is not applied here

`apps/portal/src/utils/token/isNormalized.ts`:

```ts
export const isNormalized = (name: string) =>
  name.split('.').every((label) =>
    isEncodedLabelhash(label) || isNormalizedLabel(label))
```

Its doc comment is explicitly about display contexts: encoded labelhashes are accepted because such names "are real, registered, resolvable on-chain names and must not be rejected as invalid". It has a dedicated test file, `isNormalized.test.ts`.

A sweep over `apps/portal/src`, excluding tests and the helper's own module, gives:

* `isNormalized` call sites: **1** — `routes/$name/token.tsx:290`
* chain-sourced name render sites: **9**

routes/addr/$addr/index.tsx:39 useEnsName components/WalletMenu.tsx:23 getPrimaryNameQueryOptions components/table/EventsDataTable/AddressDisplay.tsx:17 useEnsName features/transfer/hooks/useTransferName.ts:95 getPrimaryNameQueryOptions features/transfer/components/SendNameForm.tsx:255 getPrimaryNameQueryOptions features/history/components/AccountBadge.tsx:52 useEnsName features/dashboard/components/DashboardProfilePreview\.tsx:18 useEnsName features/profile/components/Owner.tsx:26 useEnsName features/registry/components/v2/RegistryEditUserSheet.tsx:69 useEnsName

The helper's siblings `isValidEnsName` and `getLabelRegistrationError` are used in `buildSearchSuggestions.ts`, `isNameOrAddress.ts`, `nameValidation.ts`, `routes/$name.tsx`, `routes/$name/index.tsx` and `routes/$name/create-subname.tsx` — every one of them a user-input path. The application validates what the user types and never validates what it renders from chain data.

### Relationship to EXP-INPUT-005

The published known-issues document lists EXP-INPUT-005 (Medium): name validators accept homoglyphs, bidirectional-control and other control characters, with the stated impact being "homograph-style display confusion in our UI rather than incorrect resolution". That entry concerns the permissiveness of input validators.

This report is the mirror defect on the other side of the boundary: the render path performs no normalization check at all on chain-sourced, third-party-authored names, while the helper written for exactly that purpose exists in the same application, is unit-tested, and is wired to one unrelated route. The root cause is not "the validator is too permissive" but "the render path has no validator", and the measured ratio is 1 call site against 9 render sites.

QA-06 (a deleted or non-resolving subname still shown as primary) is a stale-state defect with a different cause. WEB-1301 concerns which fields the address view sources from a primary name, not the trustworthiness of the name itself. No published entry addresses normalization on display.

## Impact Details

Selected impact: causing the application to display or pre-fill an incorrect name so that a user submits a transaction to an unintended recipient — specifically "rendering an ENS name to an attacker-controlled address".

Attack sequence:

1. The attacker registers a visual twin of a well-known name. On the local Sepolia fork the registrar reports `alice` as taken and the fullwidth variant as available, so the twin is obtainable.
2. The attacker sets the twin's address record to their own address, which makes forward verification succeed.
3. The attacker sets their reverse record to the twin, on L1 or via the Explorer's own L2-primary feature.
4. Every Explorer surface listing that address now renders the twin.

Concrete losses, by surface:

* **Name transfer** (`SendNameForm`): the recipient preview shows the attacker's address labelled with a name the victim reads as a party they trust. An ERC-1155 name transfer is irreversible; the loss is the full value of the transferred name.
* **Registry roles** (`RegistryEditUserSheet`): role grants are made against a list of accounts identified by name. Granting `ROLE_SET_RESOLVER` or `ROLE_SET_SUBREGISTRY` to a misidentified account hands control of resolution for that name to the attacker.
* **Events table, profile owner field, history badge, dashboard preview** (`AddressDisplay`, `Owner`, `AccountBadge`, `DashboardProfilePreview`): the address is not rendered at all, and the badge asserts the value is a `Name`. A user reading provenance or ownership is given a false identity with no fallback to verify against.

No privileged position is required. The attacker needs only to own a name and control their own address. The cost of the attack is one name registration.

## References

* `apps/portal/src/features/profile/hooks/usePrimaryName.ts` — lines 40, 43, 76, 95
* `apps/portal/src/components/table/EventsDataTable/AddressDisplay.tsx` — lines 17, 29, 30
* `apps/portal/src/utils/token/isNormalized.ts` and `isNormalized.test.ts`
* `apps/portal/src/routes/$name/token.tsx:290` — the sole call site of `isNormalized`
* `packages/l2-primary/src/utils/setReverseName.ts` — `createSetReverseNameRequest`
* `node_modules/.pnpm/viem@*/node_modules/viem/_esm/actions/ens/getEnsName.js` — no normalization
* ENSIP-15 (name normalization), ENSIP-19 (L2 primary names)
* Tested at `1c9b47f`, HEAD of `audit-comp-ready`

## Link to Proof of Concept

<https://gist.github.com/Magnat79A/3a92ff436ab911a8355ff790803c8c03>

## Proof of Concept

## Overview

Two runnable test files, both in the repository's own test harness, no network required. The first proves the check is absent; the second renders the real component and shows what a viewer sees. A third layer confirms on-chain that the visual twin is obtainable.

All results below are from a clean run at revision 1c9b47f.

## Step 1 — Setup

1. Clone the scope repository and check out `audit-comp-ready`
2. `pnpm install --frozen-lockfile`
3. Copy the two test files (from the linked gist) to:
   * `apps/portal/src/features/profile/hooks/poc-c05-primary-name.test.ts`
   * `apps/portal/src/components/table/EventsDataTable/poc-c05-render.test.tsx`

## Step 2 — The check is absent

Run: `pnpm --filter portal test poc-c05-primary-name`

Result: **5 passed (5)**

Assertions:

1. The application's own display-context helper rejects the twin. `isNormalized("alice.eth")` -> true, `isNormalized("ａlice.eth")` -> false
2. The write side forwards an un-normalized name to the registrar. `createSetReverseNameRequest({ name: "ａlice.eth", reverseRegistrarChainId: 10, targetAddress })` -> the returned `args` contain `"ａlice.eth"` unchanged
3. The read side returns it unflagged. ensjs `getName` is mocked to `{ name: "ａlice.eth", match: true }`, i.e. a reverse record that forward-verifies because the attacker owns the name. -> `getPrimaryNameQueryOptions(attackerAddress).queryFn()` returns `"ａlice.eth"`
4. What the UI receives is not a normalized name. `isNormalized(displayed)` -> false; `displayed.normalize("NFKC")` -> `"alice.eth"`

Printed output:

\[ { "displayedToVictim": "ａlice.eth", "looksLike": "alice.eth", "isNormalized": false, "nfkcFoldsTo": "alice.eth", "rendersIdentically": true } ]

Mocking note: three of the five assertions are pure — no mocks at all. The one mocked assertion stubs only the module boundary (`safeGetClient` and ensjs `getName`); `getPrimaryName` itself executes unmodified.

## Step 3 — The consequence, in the real component's DOM

Run: `pnpm --filter portal test poc-c05-render`

Result: **4 passed (4)**

`AddressDisplay` is rendered unmodified. `wagmi` is mocked partially via `importOriginal`, so `src/lib/wagmi.ts` still builds the real config and only `useEnsName` is swapped — it stands in for the on-chain reverse read. Router, react-query and `NameAvatar` are stubbed exactly as the repository's own `src/components/EntityBadge.test.tsx` does, so the tree mounts without providers or network.

Assertions:

1. The attacker-authored string is present in the DOM.
2. The address is absent — neither the full address nor its truncated form appears.
3. The rendered text NFKC-folds to the real, well-known name.
4. Control: with no reverse record the address IS shown. The fallback a viewer would otherwise rely on is suppressed precisely when an attacker acts.

Printed output:

{ "attackerAddress": "0x00000000000000000000000000000000000000a1", "reverseRecordWrittenByAttacker": "ａlice.eth", "renderedWithoutReverseRecord": "WalletAddressCopyCopy0x0000…00a1", "renderedWithReverseRecord": "NameNameCopyCopyａlice.eth", "nfkcOfRendered": "NameNameCopyCopyalice.eth", "appsOwnCheckWouldReject": true, "addressVisibleToViewer": false }

Note the badge label across the two rows. Without a reverse record the component renders as `Wallet` / `Address` and shows the truncated address. With one it renders as `Name` and shows only the attacker-authored string. This is `AddressDisplay.tsx:30`:

const variant = ensName ? 'name' : variantProp

The application relabels the entity as a name, which is an explicit identity claim about a value the attacker wrote.

## Step 4 — The visual twin is obtainable on-chain

Bring up the repository's own local stack: `pnpm e2e:infra:up` This is an Anvil fork of Sepolia at chain id 11155111, so contract addresses match the deployed ones.

cast call 0xa88553F454b77203B0D036A05c894d555EAAa2Cc\
"isAvailable(string)(bool)" "alice" --rpc-url <http://127.0.0.1:8545> -> false

cast call 0xa88553F454b77203B0D036A05c894d555EAAa2Cc\
"isAvailable(string)(bool)" $'\uff41lice' --rpc-url <http://127.0.0.1:8545> -> true

The real name is registered; its visual twin is free. The twin can equally be registered by calling the registrar directly, so no application involvement is required for this step.

## Step 5 — Measure the radius

From the repository root:

grep -rn "\bisNormalized(" --include=*.ts --include=*.tsx apps/portal/src | grep -v "isNormalized.ts|.test." -> 1 result: apps/portal/src/routes/$name/token.tsx:290

grep -rnE "useEnsName(|getPrimaryNameQueryOptions(" --include=*.tsx --include=*.ts apps/portal/src | grep -v ".test.|usePrimaryName.ts" -> 9 results, listed in the Description

## Step 6 — Confirm the dependency does not normalize

grep -rn "normalize" node\_modules/.pnpm/viem@\*/node\_modules/viem/\_esm/actions/ens/getEnsName.js -> no results

viem calls `reverse(bytes)` on the UniversalResolver and returns the result verbatim, which is why the `useEnsName` consumers are affected and not only the `usePrimaryName` path.

## Full-chain reproduction

An optional shell script running all of the above in sequence is in the linked gist (`c05-full-chain.sh`), producing the log `08-c05-chain.log`. It walks: entry (twin obtainable) -> path (write unnormalized) -> exit (read unchecked) -> radius (1 vs 9) -> dependency -> both test suites.

## Artifacts (secret gist, linked above)

* poc-c05-primary-name.test.ts
* poc-c05-render.test.tsx
* 07-c05.txt — output of step 2, md5 14b128000a518557996ed1bd7f0770bb
* 09-c05-render.txt — output of step 3, md5 748fd148a2e826331e6ea21ee7b155cd
* 08-c05-chain.log — full chain run, md5 edfa9b188e0287e705864e0009a752e7

## Counter-arguments, stated up front

* "Duplicate of EXP-INPUT-005." Different side of the boundary: input validation versus render validation. The distinguishing evidence is the 1-versus-9 ratio and the existence of an unused, unit-tested display-context helper.
* "The viewer can see the address." True only for `SendNameForm`. Step 3 shows that in `AddressDisplay` the address is not rendered at all once a reverse record exists, and the badge relabels itself `Name`.
* "Forward verification protects this." It does not, and this report does not claim it is bypassed. The attacker owns the name; ENSIP-19 passes correctly. The missing step is the normalization check after it.
* "The contracts should reject such names." ENS registrars are byte-transparent by design; normalization is a client responsibility, which is why this repository ships `@adraffy/ens-normalize` and applies it on input paths.
