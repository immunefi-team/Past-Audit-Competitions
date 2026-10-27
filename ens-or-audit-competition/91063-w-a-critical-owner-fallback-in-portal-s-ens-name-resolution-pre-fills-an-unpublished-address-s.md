# 91063 w a critical owner fallback in portal s ens name resolution pre fills an unpublished address sending role grants and name transfers to an unintended recipient

**Submitted on Aug 29th 2026 at 10:22:41 UTC by @Vastargazing for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91063
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

Portal resolves an ENS name to an address through one shared hook. When forward resolution yields no address, Portal does not report that absence. Instead it substitutes the name's current owner, and renders that address the same way it renders a published `addr` record.

The address a name resolves to and the address that name publishes are different values. Portal presents them as the same value.

No layer records or displays which of the two happened. The substituted address is carried into signed transaction calldata as a registry role grantee, a resolver role grantee, a new subname's owner, or a name transfer recipient. A user who types a counterparty's ENS name into an Add-user sheet can grant `ROLE_SET_ADDR` at root scope on their own resolver to an address that name never published, with the confirmation dialog showing no grantee at all.

The selected impact is the Critical covering an application that displays or pre-fills an incorrect address so that the user submits a transaction to an unintended recipient, whose first named example is an ENS name resolving to an attacker-controlled address. This report satisfies the main clause directly: the application displays an address the name does not publish, under a label reserved for a published record, and the user then submits a transaction whose subject is that address. The named example is reached as a consequence, not assumed as a premise.

## Vulnerability Details

### Root cause

`apps/portal/src/features/roles/helpers/addUser.handlers.ts:19-39`

```ts
try {
  const resolved = await getEnsAddress(client, {
    name: nameOrAddress,
    universalResolverAddress,
  })

  let resolvedAddress = resolved

  // Fallback for names that do not set an address record:
  // use current ENS owner address so the role can still be granted.
  if (!resolvedAddress) {
    const ownerResult = await getEnsOwner({ name: nameOrAddress })
    if (ownerResult.isOk()) {
      resolvedAddress = ownerResult.value?.owner ?? null
    }
  }

  return resolvedAddress
} catch {
  return null
}
```

Three properties make the fallback reachable, shared, and invisible.

#### 1. The fallback is reachable through four separate paths

It runs whenever `getEnsAddress` returns a falsy value instead of throwing. In the pinned `viem@2.52.2`, `actions/ens/getEnsAddress.ts` returns `null` at four distinct points:

| Line   | Condition                                                   |
| ------ | ----------------------------------------------------------- |
| `:175` | UniversalResolver response is empty, `res[0] === '0x'`      |
| `:179` | decoded address is `'0x'`                                   |
| `:180` | `trim(address) === '0x00'`, an ABI-encoded zero address     |
| `:184` | the revert is allowlisted by `isNullUniversalResolverError` |

The allowlist at `viem/utils/ens/errors.ts:16-21` covers six revert names: `HttpError`, `ResolverError`, `ResolverNotContract`, `ResolverNotFound`, `ReverseAddressMismatch`, `UnsupportedResolverProfile`.

The fourth path is live only because the caller leaves `strict` unset. `getEnsAddress.ts:183-184` reads:

```ts
if (strict) throw err
if (isNullUniversalResolverError(err)) return null
```

`resolveAddressOrName` passes `name` and `universalResolverAddress`, and no `strict`. Every allowlisted revert, including a failed CCIP-read gateway (`HttpError`) and a missing resolver (`ResolverNotFound`), becomes a silent `null` and enters the fallback.

Passing `strict: true` would route all six reverts into the outer `catch` at `:37-39`, which fails closed. It would not close the other three paths: an empty response, a `'0x'` address, and an ABI-encoded zero address all return `null` without ever throwing. A missing published address has to be handled on its own terms.

#### 2. The fallback was written for one caller and is shared by all of them

The comment says "so the role can still be granted", which scopes it in intent to the add-user path. It does not stay there.

`features/address/queries/getResolvedAddress.ts:26` calls `resolveAddressOrName`, and its own doc comment at `:14-16` already describes the behaviour as "falling back to the ENS owner when the name has no address record". `features/address/hooks/useAddressResolution.ts:43-45` documents itself as "Shared by every 'name or address' input in the app (transfer recipient, add-user sheets, …)".

`resolveAddressOrName` has exactly one implementation and one caller chain in the entire monorepo.

#### 3. Provenance is erased at the hook and never recovered

`useAddressResolution.ts:77` maps any non-null result to a single status:

```ts
.with({ resolved: P.nonNullable }, () => 'resolved' as const)
```

The returned `AddressResolution` object (`:24-36`) carries `address`, `isResolving`, `isValid`, `isRawAddress`, `isInvalid`, `status`. There is no field describing where the address came from. A published record and an owner fallback are byte-identical to every downstream consumer, so no consumer can gate on the difference even if it wanted to.

### The application already guards one narrow case of this fallback

`apps/portal/src/utils/token/isNameOrAddress.ts:18-25` requires an input name to have at least two labels, and states why:

> A bare single label like `fox` is a valid ENS name string but not a resolvable account — only TLDs are single-label — and would otherwise "resolve" via the owner fallback in `resolveAddressOrName`.

The validator rejects single-label input specifically to stop the owner fallback from turning it into an address. The fallback's output was therefore already judged unsafe to surface. That judgement was applied to one input class and not extended: a two-label name with no `addr` record still reaches the fallback, and is still presented as `Resolved:`.

### What "the owner" actually is

`getEnsOwner` delegates to `apps/portal/src/utils/ens/resolveEnsOwner.ts`, which tries the V2 registry for `.eth` names and falls back to V1. On the V1 path (`:95-101`) it takes `v1Owner.owner`.

In the pinned `@ensdomains/ensjs@5.0.0-alpha.1`, `src/actions/public/v1/getOwner.ts:220-226` returns, for an unwrapped `.eth` second-level domain (2LD):

```ts
return {
  registrant: registrarOwner,   // holder of the registrar NFT
  owner: registryOwner,         // controller of the records
  ownershipLevel: 'registrar',
}
```

Portal reads `owner` and discards `registrant`. The substituted address is the registry controller, not the NFT holder. These are two different values for any name whose registration is held separately from its day-to-day management.

For a 2LD whose registrar read yields no owner, which is the expired case, `getOwner.ts:230-236` still returns `{ registrant: null, owner: registryOwner }`, because the registry entry is not cleared on expiry. An expired name therefore still produces an address.

### What the user is shown

**In the form.** `features/address/components/AddressNameInput.tsx:60-66` renders one line for a resolved name:

```ts
{isRawAddress
  ? `Using address: ${truncateAddress(address, 6, 4)}`
  : `Resolved: ${truncateAddress(address, 6, 4)}`}
```

A published record and an owner fallback produce the identical string. The word is `Resolved:` in both cases.

**In the confirmation dialog.** All five sinks that consume `useAddressResolution`:

| # | Sink                                                                        | Confirmation text before "Open wallet"                                    | Grantee/recipient shown       |
| - | --------------------------------------------------------------------------- | ------------------------------------------------------------------------- | ----------------------------- |
| 1 | `features/registry/components/v2/RegistryAddUserSheet.tsx:66` → `:193-197`  | `Grant registry roles`                                                    | No                            |
| 2 | `features/resolver/components/ResolverAddUserSheet.tsx:141` → `:316-322`    | `Grant resolver roles for (root)`, which is the scope and not the grantee | No                            |
| 3 | `routes/$name/create-subname.tsx:109` → `:318-322`                          | `Create <label>.<parent>`                                                 | No                            |
| 4 | `features/roles/components/RolesAddUserSheet.tsx:73` → `:240-247`           | `Grant roles for 0x1234…5678`                                             | Truncated (6+4)               |
| 5 | `features/transfer/components/SendNameForm.tsx:81` → `:128-134`, `:253-285` | recipient preview                                                         | Full address and primary name |

No component in the modal decodes the transaction. `EstimatedGasCost.tsx:100` receives `preparedIntent?.request`, but only to obtain a gas estimate. Its render at `:125-152` resolves to a cost figure or to a status label ("Unavailable", "Not yet"). The request bytes are never decoded or displayed.

This is not a display limitation. Given a full 42-character address as `transactionName`, the same modal in the same state renders all 42 characters. The omission is in the call sites' descriptors, not in the component. This is demonstrated in the PoC.

### Sink 5, name transfer

`SendNameForm` is the one sink that overrides the default rendering. Its `RecipientPreview` (`:253-285`) shows the resolved address untruncated (`truncate={false}` at `:280`) together with that address's primary name.

At sinks 1 to 4 the substituted address receives the granted authority while `request.to` remains the correct registry or resolver contract. At sink 5 it receives the name itself, so it is the transfer recipient directly.

Sink 5 is also the one place where the substitution can be caught without outside knowledge. A victim who independently knows the counterparty's address, or who notices that the displayed primary name is not the name they typed, has a signal there. Sinks 1 to 4 provide no such signal.

## Threat model

**Actor.** Any unprivileged party. Registering a lapsed ENS name is permissionless. It requires no relationship with the victim, no privileged address, no compromised credential.

**Attacker-created state.** The attacker registers a lapsed name and sets no ETH `addr` record. Setting no record is a deliberate choice, and the objection it raises is answered below.

**Victim's ordinary action.** The victim types that name into an Add-user sheet to delegate record management. This is the action the sheet exists to perform, inside a normal application workflow, with no unprompted step and no interaction outside the documented flow.

**What the application contributes.** Forward resolution yields nothing. Rather than reporting the absence, Portal substitutes the name's current owner, labels it `Resolved:` exactly as it labels a published record, and carries it into calldata.

The resolver sheet's scope field already reads `Root (all nodes)`. `ResolverAddUserSheet.tsx:39` defines `const ROOT_NODE_VALUE = ''` and `:127` initialises `selectedNode` to it, so the widest available grant is the default and requires no choice by the victim. The confirmation dialog then reads `Grant resolver roles for (root)`, with no grantee address anywhere.

**Resulting transaction.** `grantRootRoles(roles, attacker)` against the victim's resolver, with `ROLE_SET_ADDR` at root scope. Decoded calldata is in the PoC.

## What is lost, per sink

The grant transaction itself moves no funds. The loss is realised at the second step, when the authority it hands over is exercised. For the resolver sink that second step is the impact's own first example.

| Sink                                                        | Authority or asset handed to the unintended address                                                                        | Ceiling                                                                                                                                                        |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Resolver root grant (`ROLE_SET_ADDR` at `Root (all nodes)`) | The ability to set the ETH address record of every name served by that resolver                                            | Every payment any third party subsequently sends to any of those names, for as long as the grant stands. Not bounded by the victim's own balance               |
| Registry role grant                                         | Registry-level authority over the name: subregistry management, subname creation, and changing the name's resolver address | The namespace beneath that name, including names already delegated to others. Changing the resolver reaches the same address-record authority as the row above |
| Subname creation                                            | The new subname is created owned by the unintended address (`create-subname.tsx:164`, `:332`)                              | That subname and everything under it                                                                                                                           |
| Name transfer                                               | The ENS name itself                                                                                                        | The name as an asset, plus every payment routed through it thereafter                                                                                          |

The resolver case is the severe one, and the ceiling is not speculative. `ROLE_SET_ADDR` is documented in the app itself as "Can set address records" (`lib/roles/resolverRoles.ts:61`). Once held at root scope, the holder decides what every name on that resolver resolves to, for every ENS client. That is the impact's first example reached as a consequence rather than assumed as a premise.

Recovery requires the victim to notice a grant they were never shown and then revoke it. Nothing in the flow surfaces the grantee, so there is no point at which the mistake becomes visible without an independent audit of on-chain role state.

## Why the substitution is not an exotic state

The same substitution is produced by ordinary configurations with no adversary at all. These are not separate findings. They show the mechanism is a routine outcome of the resolution path, not a corner case an attacker engineers. All three are demonstrated in the PoC.

* **Controller is not registrant.** For an unwrapped `.eth` 2LD, Portal substitutes the registry controller and discards the registrant. Any name whose registration is held separately from its management resolves, under the fallback, to an address its holder never published.
* **Expired name.** For a 2LD past expiry that has not been re-registered, Portal still yields the address of the party that controlled it before it lapsed. The name is owned by nobody, and Portal still supplies a recipient.
* **Resolution failure, including offchain.** A name on a CCIP-read resolver whose gateway errors raises `HttpError`, which the unset-`strict` allowlist maps to `null`. The published offchain address and the on-chain owner differ by construction, so a transport failure becomes a different valid address rather than "no address".

## The precondition

The victim must type a name the attacker controls.

The selected impact turns on what the application displays and what the user then submits. Nothing in its main clause depends on how the victim came to type the name, and typing a counterparty's name is the Add-user sheet's documented purpose, so this is not an action outside a normal workflow.

The difference from a social-engineering finding is what is misrepresented, and by whom. No misrepresentation is made to the victim about the name. They typed it, they can read it back, and it is exactly what they intended. The misrepresentation is made by the application about the address: it presents an address derived from ownership under a label reserved for a published record, and removes every signal that would distinguish the two.

A careful victim can verify the name they typed. They cannot verify whether the address shown is what that name publishes or merely who currently holds it, because no layer of the application distinguishes the two. A victim exercising ordinary care is defeated by the application's output, not by a claim the attacker made.

## Why omitting the address record is not equivalent to publishing one

An attacker who owns the name could publish an `addr` record pointing at themselves and reach the same address with no application defect involved. The two are not equivalent in what they leave behind.

A published `addr` record is an on-chain artefact. It is what forward resolution returns, so any client performing that lookup obtains it, and it remains visible before and after the fact.

Omitting the record produces something structurally different, and the PoC shows exactly what. Where a record exists, resolution returns it and `findOwner` is never called. Where resolution is empty, `findOwner` and `findRegistries` are called and the address comes from there. The address the victim sees in the fallback case is therefore not a resolution result at all. It is produced by a separate ownership read that Portal performs on its own initiative, after resolution has already answered "nothing".

Forward resolution for that name continues to answer "nothing" to everyone, including to Portal. Nothing on-chain records that Portal answered differently.

Within this codebase the behaviour is also not a shared convention. `resolveAddressOrName` has one implementation and one call chain, and `apps/manager` performs no forward name-to-address resolution at all.

## Scope and commit

Asset `apps/portal` on branch `audit-comp-ready`. Commit tested: `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, the branch head. All links below are permalinks at that commit.

The competition repository imported the upstream monorepo at `63772fd872af472ced58b009499355f3430c2a86` in commit `cda79ac`. Between that import and the commit tested here, the only changes are a `hono` version bump in three `package.json` files and the corresponding `pnpm-lock.yaml` entries. No file under `apps/portal` changed, and the resolved `viem` and `@ensdomains/ensjs` versions are identical. Verified with `git diff cda79ac..1c9b47f`.

## Novelty

The Known Issues document (last updated 2026-08-14) states verbatim:

> "Anything not listed here is in scope under the normal competition rules, including: issues we fixed incorrectly or incompletely (a bypass of a shipped fix is a new finding); new consequences of a listed root cause that materially change its severity; everything found in areas these audits did not cover."

Checked against every entry that touches names, addresses or resolution.

**`EXP-INPUT-005`, "ENS name validators accept deceptive characters".** The entry states that "The practical impact is homograph-style display confusion in our UI rather than incorrect resolution." This report does not depend on a deceptive character. The substitution occurs for an exactly-typed, correctly-normalised name that the victim reads back correctly.

**`WEB-1301`.** "The address view displays owner, registered, and expires values taken from the wallet's primary name, which is confusing in an address context." A different view, different data, and no transaction.

**`WEB-1249`.** Subname creation replacing an existing child registry. A different operation and a different root cause.

**`WEB-1304`.** Wrapped V1 renewal leaving the NameWrapper expiry stale, so that "its owner reads as the zero address". Adjacent, and checked directly: a zero owner cannot reach `resolvedAddress`. `ensjs`'s `nullableAddress` maps it to `null`, and both `resolveEnsOwner.ts:53` (V2) and `:96` (V1) reject it, as does `apps/manager`'s `profileOwner.ts:43` and `:93`. That path fails closed and is not part of this report.

**`WEB-626`.** The renewal modal implies a grace-period name you own is owned by someone else. Adjacent to the expired-name configuration only in that both involve a lapsed name. `WEB-626` is a wording defect in a modal about a name the user already holds, with no resolution and no counterparty. This report concerns a name the user does not hold being resolved to an address it does not publish.

**`QA-01`, `WEB-1257`, `WEB-1305`.** Post-transfer role and primary-name breakage. Downstream failures of transfers that already happened, not the selection of who receives one.

No listed entry names owner fallback, a missing address record, the `isNullUniversalResolverError` allowlist, resolution provenance, or `addUser.handlers.ts`.

## Link to Proof of Concept

<https://gist.github.com/Vastargazing/250d1f82f973a817cf1e8e91edf95777>

## Proof of Concept

### Step by step

The resolver-role path, which is the severe one. Sinks 1, 3 and 4 differ only in which builder consumes the same resolution result.

{% stepper %}
{% step %}

## The attacker registers a lapsed ENS name

This is permissionless and needs no relationship with the victim. On that name the attacker sets **no** ETH `addr` record.
{% endstep %}

{% step %}

## The victim enters the attacker's name

The victim opens the resolver Add-user sheet in Portal to delegate record management, and types the attacker's name. This is the action the sheet exists to perform.
{% endstep %}

{% step %}

## Portal resolves the input

Portal calls `useAddressResolution` → `getResolvedAddress` → `resolveAddressOrName` (`apps/portal/src/features/roles/helpers/addUser.handlers.ts:19-39`). `getEnsAddress` returns `null`, because the name publishes no address record.
{% endstep %}

{% step %}

## Portal substitutes the owner

Instead of reporting that absence, `resolveAddressOrName` calls `getEnsOwner` and returns the name's current registry owner, which is the attacker's address.
{% endstep %}

{% step %}

## The form renders the substituted address as resolved

The form renders `Resolved: 0x1000…0001`. This is the identical string Portal renders for a genuine published record. `useAddressResolution.ts:77` collapses both cases to `status: 'resolved'`, and the returned object carries no field describing where the address came from.
{% endstep %}

{% step %}

## Root scope is selected by default

The scope field already reads `Root (all nodes)`. `ROOT_NODE_VALUE = ''` is the initial state (`ResolverAddUserSheet.tsx:39` and `:127`), so the widest available grant is the default and requires no choice by the victim.
{% endstep %}

{% step %}

## The victim selects the role and confirms

The victim selects `ROLE_SET_ADDR`, described in the app as "Can set address records", and confirms.
{% endstep %}

{% step %}

## The confirmation dialog omits the grantee

The confirmation dialog reads `Grant resolver roles for (root)`. That string is the permission scope, not the grantee. No grantee address appears anywhere in the dialog, and no component decodes the request bytes.
{% endstep %}

{% step %}

## The victim signs

The transaction is `grantRootRoles(1, 0x1000…0001)` against the victim's own resolver, with the attacker's address at `args[1]`. The `1` is the role bitmap, not a count: `ROLE_SET_ADDR` is `1n << 0n` (`lib/roles/resolverRoles.ts:25`).
{% endstep %}

{% step %}

## The attacker receives root-scope address-record authority

The attacker now holds address-record authority at root scope on that resolver and can set what every name it serves resolves to, for every ENS client.
{% endstep %}
{% endstepper %}

At no point in this sequence is the victim shown anything that distinguishes an address the name publishes from an address the name merely happens to be owned by.

### Reproduction

Two deterministic, network-free Vitest harnesses. No mainnet, public testnet, or external gateway is contacted. The resolution branch taken is fixed by asserting on the exact contract reads that the installed `viem` and `@ensdomains/ensjs` issue against a substituted client.

**Environment as measured:** Node `v22.22.1`, pnpm `10.27.0` via Corepack, Vitest `4.1.5`.

**Setup.** Check out `1c9b47f18fcddd2e864dfe385c4171061c9811ae` and run `pnpm install` at the repository root, so that `apps/portal/node_modules` exists.

The harness files are in the linked Gist, packaged as a single self-extracting installer. Run it from the repository root:

```bash
bash adr-01-poc-setup.sh
```

It creates the two harness directories at these paths:

```
.security-audit/security-recon/experiments/portal-address-resolution-first-pass/
.security-audit/security-recon/experiments/portal-confirmation-path-first-pass/
```

The depth matters: each `vitest.config.ts` resolves `apps/portal` as `../../../../apps/portal` relative to itself, so the directories have to sit four levels below the repository root. The installer refuses to run anywhere else, and writes nothing outside `.security-audit/security-recon/experiments/`. The SHA-256 of every file it writes is listed at the end of this section, so the extracted tree can be verified against this report.

Run both commands from the repository root.

#### Harness A: resolution branches and role-grant calldata

```bash
corepack pnpm --dir apps/portal exec vitest run \
  --config "$PWD/.security-audit/security-recon/experiments/portal-address-resolution-first-pass/vitest.config.ts"
```

Observed:

```
Test Files  1 passed (1)
Tests       14 passed (14)
```

It drives the real `resolveAddressOrName` against the installed libraries. The assertions are on which contract reads each branch actually issues, which is what fixes the branch taken.

The run contains 14 tests. The cases below are the ones this report relies on.

| Case                                           | Result                           | Distinguishing evidence                                      |
| ---------------------------------------------- | -------------------------------- | ------------------------------------------------------------ |
| Published record differs from owner            | The published record is returned | `resolveWithGateways` only. `findOwner` is never called      |
| ABI-encoded zero `addr()`                      | The owner is returned            | `resolveWithGateways` ×2, then `findOwner`, `findRegistries` |
| Resolver absent (`ResolverNotFound` revert)    | The owner is returned            | `resolveWithGateways`, then `findOwner`, `findRegistries`    |
| CCIP-read gateway failure (`HttpError` revert) | The owner is returned            | `resolveWithGateways` ×2, then `findOwner`, `findRegistries` |
| Empty UniversalResolver response               | The owner is returned            | `findOwner` is reached                                       |
| Non-existent, unowned name                     | `null` is returned               | Fails closed                                                 |
| Non-allowlisted resolution error               | `null` is returned               | Fails closed, `resolveWithGateways` only                     |
| Exported no-React `queryFn`                    | The owner is returned            | Same fallback off the React path                             |

The first row is the positive control. The fallback is not an unconditional ownership lookup: it fires precisely when forward resolution yields no address, and when a record exists the ownership read never happens at all.

The three adversary-free configurations are driven directly, not inferred:

| Configuration                    | Setup                                                                | Result                                                                    |
| -------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Controller is not registrant     | Registrar NFT held by `0x70…07`, registry controller `0x80…08`       | The **controller** is returned. The test asserts it is not the registrant |
| Expired second-level `.eth` name | Registrar read yields no owner, registry entry still holds `0x90…09` | `0x90…09` is returned. A name owned by nobody still produces a recipient  |
| CCIP-read gateway failure        | `HttpError` revert from `resolveWithGateways`                        | The owner is returned                                                     |

On the expired-name row: the branch under test is `registrarOwner === null`, which `ensjs` reaches either from a revert (via `isNullRegistrarOwnerOfError`) or from a zero address (via `nullableAddress`). The test drives the zero-address route, because a revert constructed inside the harness cannot satisfy `ensjs`'s `instanceof BaseError` across the harness's module boundary. That is a limitation of the harness, not of the application. Both routes converge on the same `null` before the branch is taken.

The four remaining tests are not load-bearing for this report. Three are bounded controls that keep it from overclaiming: a wrong-checksum address is passed through byte-for-byte and not "corrected"; an exactly-typed name is distinguished from the homoglyph display confusion already listed as `EXP-INPUT-005`; and a cached address stays put until its query is invalidated. The fourth is the calldata test described below.

The same run decodes the calldata for the two role-grant sinks and prints it:

```json
{"attacker":"0x1000000000000000000000000000000000000001",
 "registry":{"functionName":"grantRootRoles",
   "calldata":"0x072d5d77…0001",
   "decodedArgs":["16777216","0x1000000000000000000000000000000000000001"]},
 "resolver":{"functionName":"grantRootRoles",
   "calldata":"0x072d5d77…0001",
   "decodedArgs":["1","0x1000000000000000000000000000000000000001"]}}
```

The substituted address occupies `args[1]` of `grantRootRoles` in both, with `request.to` set to the registry and the resolver respectively. This is the same `request.data` the app carries into the signing intent.

Resolver **name-scoped** grants place the address at `args[2]` of `authorizeNameRoles`, since `@ensdomains/ensjs` builds `[dnsEncode(name), roleBitmap, targetAccount, true]` at `src/actions/wallet/v2/grantResolverRoles.ts:129-134`. That position is established from source; the harness decodes the two root grants above and does not print this one.

The two remaining sinks are established from source in the same way, and are not decoded by the harness: `create-subname.tsx:164` and `:332` pass the resolved address as the new subname's `owner`, and `SendNameForm` passes it as the transfer `recipient` into `useTransferName` (`:38`, `:43`).

#### Harness B: what the confirmation dialog shows

```bash
corepack pnpm --dir apps/portal exec vitest run --reporter verbose \
  --config "$PWD/.security-audit/security-recon/experiments/portal-confirmation-path-first-pass/vitest.config.ts"
```

Observed:

```
Test Files  1 passed (1)
Tests       2 passed (2)
```

Two renders of the real `TransactionModal`, in the same state, differing only in what the caller's descriptor carries. Both lines below are verbatim console output:

```
CNF-RENDER-EVIDENCE       "Grant rolesGrant roles for 0x1234…5678Open wallet"
CNF-FULL-ADDRESS-EVIDENCE "Grant rolesGrant roles for 0x1234567890abcdef1234567890abcdef12345678Open wallet"
```

The first reproduces what `RolesAddUserSheet` actually supplies. The second is the distinguishing control: given a full 42-character address, the same modal in the same state renders all 42 characters. The omission at sinks 1 to 3 and the truncation at sink 4 are therefore properties of the call sites' descriptors, not a limitation of the modal.

### PoC artifacts, with SHA-256

```
7edadb55cc69fa9524e3aa38df77e1ba6e19976d2b4f50c678bb9bdc05bfa40e  portal-address-resolution-first-pass/resolution-and-calldata.test.ts
1679edc1dfd3cf8304fcf7685786563ef4feee328c5e53f473f077353bcbcfc7  portal-address-resolution-first-pass/vitest.config.ts
3fefdf9b7dbea89c0ae75790d772a383c3f02e662f5247c1034bf0e4b06609db  portal-address-resolution-first-pass/transaction-manager.mock.ts
4cd463703cae180c8be66ff994b6be4c141c7a3e2621b86a248ccd656de20d68  portal-address-resolution-first-pass/universal-resolver.mock.ts
30b5fe58b3ee5f6ee0d53ff8e86e1e01571b66d78051bef623571c00f3169d94  portal-address-resolution-first-pass/wagmi-helpers.mock.ts
02b8a90fc1b8653c822b0d0c5f8c646292a4984bb3c8bb61933ba55284ac1a7f  portal-confirmation-path-first-pass/transaction-modal.test.tsx
46093991a9d0764a4d9b1c465092926ef6ee1fc34eaf0803e00364e23a2898e3  portal-confirmation-path-first-pass/vitest.config.ts
c6d84abbf635887b5759e2fc0041de5fb13204ace837949a5039375614e032d2  portal-confirmation-path-first-pass/dialog.mock.tsx
c3e34e13e1941b536fa020c6e4e709a502d24cbee2dfe18bd9338ad5294d7688  portal-confirmation-path-first-pass/transaction-manager.mock.ts
7188a001c4ea830039a32ea4c2fa6e000c7ac5c76f84d1b8f43b3e766879e483  portal-confirmation-path-first-pass/wagmi.mock.ts
```
