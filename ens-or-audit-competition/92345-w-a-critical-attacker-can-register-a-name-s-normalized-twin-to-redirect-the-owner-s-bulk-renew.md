# 92345 w a critical attacker can register a name s normalized twin to redirect the owner s bulk renew payment and let their name expire

**Submitted on Sep 9th 2026 at 04:33:42 UTC by @nomorecaffeine for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92345
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

The Manager dashboard lets a user select several names and renew them in one transaction. Each selected row is turned into a renewal payload by `toBulkRenewName`:

[NamesTable.tsx#L83-L90](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/dashboard/components/NamesTable.tsx#L83-L90)

```ts
  const displayName = resolveDomainLabel(domain)
  // Use the canonical normalized name for the on-chain label — the display name
  // may be unnormalized (e.g. mixed case) and would hash to the wrong label.
  const name = domain.normalizedName ?? displayName
  return {
    displayName,
    label: name.replace(/\.eth$/i, ''),
    name,
```

As seen from above, the row renders `displayName`, the label as registered, while the `renew()` calldata is built from `label`, which comes from `normalizedName`.

However, this is incorrect, as an ENS registration is keyed by `keccak256(label_as_registered)`. `normalizedName` is not another spelling of the same registration; it is the ENSIP-15 normalization *of* that label, a different string hashing to a different key. Whenever a registered label is not already normalized, the two address two different registrations.

Nothing downstream corrects it. From `NamesTable.tsx:86` the label passes through `useBulkRenew.ts:181`, `useBulkRenewSubmit.ts:76` and `registration.actors.ts:1072` into `renew(string label, ...)`, crossing the ABI boundary as a string for the registrar to hash. No step re-normalizes it, re-derives it from `tokenId`, or checks who owns the name.

On its own this would be a display bug, because the same normalized label is also what gets priced:

[useBulkRenew.ts#L155](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/bulk-renew/hooks/useBulkRenew.ts#L155)

```ts
        ...getRenewPriceQueryOptions(n.label, Number(duration), selectedToken),
```

`getRenewPrice` reverts for a label that is not registered, so the query returns no data, `pricesReady` at `:178-180` is false, and `canConfirm` at `:235` keeps the button disabled. A user whose normalized twin does not exist cannot submit at all.

This is a vulnerability when combined with the fact that anyone may register that twin:

* Assume the following:
  * Alice owns `éhca-1a016f8610f.eth`, registered in NFD form (`e` + `U+0301`).
  * The NFC twin (`U+00E9`) is unregistered, so Alice's bulk renew is currently blocked for that row.
* Mallory registers the NFC twin at the 28-day minimum, for `0.613701` USDC.
* Alice opens her dashboard. Her row is unchanged: her own name, correctly spelled, with her own expiry. The two forms render identically in every browser.
* Alice selects the row, or uses "Select all names", and confirms:
  * Name validation does not filter it out. `isSelectableDomain` at `NamesTable.tsx:74-78` is handed `domain.normalizedName ?? domain.name`, the already-normalized form, which passes any character check by construction.
  * `getRenewPrice` now succeeds, because the normalized label exists, so `pricesReady` becomes true and the button unlocks.
  * The price quoted is the price of Mallory's registration.
* Alice signs one transaction:
  * Mallory's twin gains the full duration Alice paid for.
  * Alice's own name is untouched and continues to expire.

Renewal is permissionless, so no ownership check stops this: `AbstractETHRegistrar.renew` performs none, and the only validity gate, `StandardRentPriceOracle.isValid`, carries the NatSpec *"Does not check if normalized."*

Note that the population this applies to is not hypothetical. Against `https://staging-graphql.ens.dev/`, the endpoint the app defaults to, 22 registrations currently have `isNormalized: false`, and 8 of those already have their normalized twin held by a different address. For the rest, Mallory only has to register the twin herself.

The same defect exists in the Explorer by a different mechanism: [useRenewalTransactions.ts#L197](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/portal/src/features/renew/hooks/useRenewalTransactions.ts#L197) builds the calldata as `` `${getLabel(params.name)}.eth` ``, normalized, while `:457-460` and `:223` display the raw `selectedName.name`.

### Impact

For `0.613701` USDC, an attacker can make any holder of a non-normalized name pay for the attacker's registration instead of their own.

The victim loses twice: they are charged the full renewal price, `8.000021` USDC per year on the preset they choose, and the name they believed they had just renewed is untouched, so it runs down its expiry and is lost. `CRYPTO.eth`, one of the eight live cases, expires `2026-09-17T16:59:12Z`.

Nothing is required of the victim beyond using bulk renew as intended, and the attacker never contacts them, holds no privileged role, and never touches the victim's wallet or session.

### Recommendation

Send the label the row displayed. `resolveDomainLabel` already returns it, so this needs no indexer change:

[NamesTable.tsx#L86](https://github.com/immunefi-team/audit-comp-ens/blob/audit-comp-ready/apps/manager/src/features/dashboard/components/NamesTable.tsx#L86)

```diff
-  const name = domain.normalizedName ?? displayName
+  const name = displayName
```

This makes the label that is hashed the same expression as the label that was rendered. `NamesBreakdown.tsx:37` and `useBulkRenew.ts:189` need the same change: both currently take the normalized form, so on the confirm screen the avatar and the profile records already describe the attacker's registration.

## Link to Proof of Concept

<https://gist.github.com/nguyen-xuan-quoc/f9d6f5b4969eedae3e4154df229eac50>

## Proof of Concept

The following gist contains two vitest tests. The first renders the real, unmodified `NamesTable` and captures the diverging label leaving it. The second drives the shipped `submitRenewActor` on a fork and reads the result back off chain:

<https://gist.github.com/nguyen-xuan-quoc/f9d6f5b4969eedae3e4154df229eac50>

To run the tests:

1. Install dependencies at the repo root:

```sh
pnpm install --frozen-lockfile
```

2. Start a Sepolia fork. Do not pin a block number, as public RPCs prune the state the second test reads:

```sh
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8547 --chain-id 11155111 --silent &
```

3. Copy the test into `apps/manager/src/` and run it. `--disableConsoleIntercept` is required, or the suite passes and prints nothing:

```sh
pnpm --filter manager exec vitest run \
  src/bulkRenewLabelDivergence.poc.test.tsx --disableConsoleIntercept
```

Alice's own name is expected to be renewed. Instead:

```
{
  "rowShown_NFD": "65 301 70 6f 63 30 34 31 36 32 31",
  "calldataLabel_NFC": "e9 70 6f 63 30 34 31 36 32 31",
  "visuallyIdentical": true,
  "attackerOwns": "0x9fffa7c2a44a07ccde6dd5e4186a333273c9bd0d",
  "victimOwnsBait": "0xa7bd77ad389a03143c898564392b5768de6b1500",
  "attackerExpiry": "1820460553 -> 1851996553 (+31536000)",
  "victimBaitExpiry": "1820460624 -> 1820460624 (unchanged)",
  "victimUsdcSpent": "8000021"
}
```
