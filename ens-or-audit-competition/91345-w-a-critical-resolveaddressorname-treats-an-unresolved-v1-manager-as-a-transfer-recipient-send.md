# 91345 w a critical resolveaddressorname treats an unresolved v1 manager as a transfer recipient sending a v2 name nft to an unintended address

**Submitted on Aug 31st 2026 at 19:18:16 UTC by @go4ko for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91345
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

Explorer's transfer form accepts either an address or ENS name as the recipient. Its resolution path reuses `resolveAddressOrName`, a helper whose source comment explains that it falls back to an ENS owner's address so a role can still be granted when the name has no address record.

That fallback is unsafe in an asset-transfer workflow. When canonical forward resolution returns `null`, the helper calls `getEnsOwner` and returns an authority address instead. For an active, unwrapped V1 `.eth` name, the returned `owner` is the ENS Registry owner—the delegated **Manager**—which can differ from the BaseRegistrar registrant. Neither address is an ETH address record, and Registry management authority over the recipient name does not authorize receipt of the sender's unrelated V2 name NFT.

The fallback's provenance is discarded. `useAddressResolution` classifies the non-null Manager as `resolved`; `AddressNameInput` therefore suppresses its unresolved-name warning; `SendNameForm` considers the Manager a valid recipient; and `transferToken` encodes it directly into the V2 Registry's ERC-1155 `safeTransferFrom` calldata.

The final browser E2E demonstrates the complete path without mocking resolution, form validation, transaction construction, wallet approval, mining, calldata decoding, receipts, or token balances. Canonical Universal Resolver lookup of `fgeorgescu.eth` is `null`, its V1 Manager and registrant are distinct, the live Explorer DOM displays the Manager and no unresolved warning, and the real default transfer flow mines:

```
safeTransferFrom(sender, manager, tokenId, 1, 0x)
```

The sender's balance changes from `1` to `0`, the Manager's from `0` to `1`, and the registrant's remains `0`. A later Manager-to-vault transfer succeeds without further sender participation. A former-sender transfer reverts, and even the V1 registrant's successful `reclaim` does not recover the unrelated V2 token.

### Security invariant and actors

The transfer form promises an `ENS name or address` recipient. For a name input, the required invariant is:

```
If canonical forward resolution does not return a non-zero address,
the transfer recipient must remain unresolved and submission must stay disabled.

Registry ownership, BaseRegistrar registration ownership, wrapper ownership,
and role authority must never substitute for a missing address record.
```

The recorded PoC uses four distinct deterministic local-fork actors:

* **S** — sender; owns the source V2 name and authorizes the Explorer transfer;
* **R** — BaseRegistrar registrant of the V1 name entered into the recipient field;
* **M** — delegated V1 Registry Manager returned by the vulnerable fallback;
* **V** — a separate attacker vault to which M moves the wrongly received NFT.

The addresses preserved in the successful run are:

| Actor | Address                                      |
| ----- | -------------------------------------------- |
| S     | `0x70997970C51812dc3A010C7d01b50e0d17dc79C8` |
| R     | `0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc` |
| M     | `0x976EA74026E726554dB657fA54763abd0C3a0aa9` |
| V     | `0x14dC79964da2C08b23698B3D3cc7Ca32193d9955` |

M is not an ENS protocol administrator, project operator, or trusted role over S's token. M only manages the separate V1 name supplied as the recipient input. The defect crosses that authority boundary and turns name-management authority into ownership of S's unrelated NFT.

The vulnerable state transition is:

```
V1 name is active and unwrapped
    -> BaseRegistrar registrant = R
    -> ENS Registry owner / Manager = M
    -> resolver = 0x0
    -> canonical forward resolution = null
    -> S enters the V1 name in Explorer's V2 transfer form
    -> role-grant helper substitutes M for the missing address record
    -> UI classifies the input as resolved and suppresses its warning
    -> transfer button becomes enabled
    -> S approves Explorer's default transfer sequence
    -> V2 Registry mines safeTransferFrom(S, M, tokenId, 1, 0x)
    -> M transfers the token to V
    -> S can no longer transfer it
    -> R reclaims V1 management, but the V2 token remains at V
```

## Vulnerability Details

### 1. The fallback was expressly added so roles can still be granted

[`apps/portal/src/features/roles/helpers/addUser.handlers.ts:11-40`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/roles/helpers/addUser.handlers.ts#L11-L40) first performs canonical forward resolution. If it returns no address, the helper replaces the missing result with ENS ownership data:

```ts
export async function resolveAddressOrName({
  client,
  nameOrAddress,
}: ResolveAddressOrNameParams): Promise<Address | null> {
  if (isAddress(nameOrAddress, { strict: false })) {
    return nameOrAddress as Address
  }

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
}
```

The comment identifies the intended semantic: allowing a **role** to be granted to the name's owner when no address record exists. That policy cannot safely be generalized to a token destination. An ownership/controller address is an authority over the input name; it is not the address to which the user asked to transfer another asset.

### 2. For an unwrapped V1 name, the retained `owner` is the Manager—not necessarily the registrant

[`apps/portal/src/utils/ens/resolveEnsOwner.ts:72-104`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/ens/resolveEnsOwner.ts#L72-L104) prefers V2 ownership for `.eth` names, then falls back to ENSjs V1 ownership:

```ts
const v1Owner = await getOwnerV1(client, { name })
if (v1Owner?.owner && v1Owner.owner !== zeroAddress) {
  return {
    owner: v1Owner.owner,
    registryAddress: v1EthRegistry,
    protocolVersion: 'ENSv1',
  }
}
```

For an active unwrapped V1 `.eth` 2LD, ENSjs exposes distinct Registry `owner` and BaseRegistrar `registrant` authorities. This function retains `.owner` and discards `.registrant`. Delegating management while retaining registration is a supported ENS state, so those addresses can legitimately differ.

The repository models that distinction correctly elsewhere. [`useMigrationStatus.ts:29-40`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/migration/hooks/useMigrationStatus.ts#L29-L40) explicitly documents that the BaseRegistrar registrant is the unwrapped token holder while the Registry `owner` is a controller that may be different. The transfer-resolution path loses that distinction.

### 3. The role-specific fallback is exported as the shared address resolver

[`apps/portal/src/features/address/queries/getResolvedAddress.ts:13-45`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/queries/getResolvedAddress.ts#L13-L45) imports the role helper and exposes it as the generic query for a name-or-address field:

```ts
/**
 * Resolve a name/address input (an ENS name or a 0x address) to an address.
 * Addresses pass through; names go through the universal resolver, falling
 * back to the ENS owner when the name has no address record.
 */
export const getResolvedAddress = ResultFn(async function* (
  params: GetResolvedAddressParams,
) {
  const client = yield* safeGetClient()
  const address = await resolveAddressOrName({
    client,
    nameOrAddress: params.nameOrAddress,
  })
  return ok<Address | null>(address)
})
```

The type is only `Address | null`; it carries no provenance such as `raw-address`, `address-record`, or `owner-fallback`. Downstream transfer code therefore cannot distinguish a user-supplied or canonically resolved destination from an authority address substituted for a failed lookup.

### 4. A fallback address becomes the `resolved` UI state

[`apps/portal/src/features/address/hooks/useAddressResolution.ts:47-89`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/hooks/useAddressResolution.ts#L47-L89) classifies every non-null query result as resolved:

```ts
.with({ resolved: P.nonNullable }, () => 'resolved' as const)
...
return {
  address: status === 'resolved' ? resolved : null,
  ...
}
```

[`AddressNameInput.tsx:54-87`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/components/AddressNameInput.tsx#L54-L87) renders its warning only for `unresolved`:

```tsx
.with('unresolved', () => (
  <p className="text-sm mt-1.5 text-danger">
    Could not resolve an address for “{value.trim()}”
  </p>
))
```

Because the fallback changed `null` into M before status derivation, Explorer suppresses the exact warning that should stop the user. The successful PoC records `unresolvedWarningCount == 0` even though an independent canonical Universal Resolver call returns `null`.

### 5. The transfer form treats the substituted Manager as a valid recipient

[`apps/portal/src/features/transfer/components/SendNameForm.tsx:81-108`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L81-L108) accepts any non-null address that is not the sender and not zero:

```ts
const resolution = useAddressResolution(recipientInput)
const { address: recipient, isResolving } = resolution

const isSelf = !!recipient && isAddressEqual(recipient, owner)
const isZeroAddress = !!recipient && isAddressEqual(recipient, zeroAddress)
const hasValidRecipient = !!recipient && !isSelf && !isZeroAddress

const canStart =
  hasValidRecipient && !isResolving && !isPreparing && detachTargetsSettled

const runTransfer = () => {
  if (!recipient || !canStart) return
  startTransfer({ recipient, options: effectiveOptions })
}
```

The returned Manager passes all three checks. `RecipientPreview` at [`SendNameForm.tsx:253-281`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L253-L281) then renders that address as the recipient and may decorate it with the address's own primary name and avatar. Nothing tells the sender that the displayed value came from Registry authority rather than the entered name's ETH address record.

### 6. The substituted Manager is encoded into a real ERC-1155 transfer

[`apps/portal/src/features/transfer/helpers/transferToken.ts:46-90`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/helpers/transferToken.ts#L46-L90) constructs the V2 name transfer:

```ts
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})
```

There is no later resolution or recipient-validation boundary. The address accepted by the UI becomes the on-chain `to` argument. The V2 Registry correctly enforces the calldata it receives; the defect is Explorer constructing that calldata from the wrong semantic source.

### State transition demonstrated by the E2E

| Stage                    | Canonical resolution of recipient name | V1 registrant | V1 Manager | V2 token holder | Security result                                                      |
| ------------------------ | -------------------------------------- | ------------- | ---------- | --------------- | -------------------------------------------------------------------- |
| Fixture established      | `null`                                 | R             | M          | S               | No address record designates M or R as transfer recipient.           |
| Explorer resolves input  | `null`                                 | R             | M          | S               | UI displays M, suppresses unresolved warning, and enables transfer.  |
| S approves default flow  | `null`                                 | R             | M          | S until mining  | Explorer constructs `safeTransferFrom(S, M, ...)`.                   |
| Portal transfer mines    | `null`                                 | R             | M          | **M**           | S loses the V2 name; R receives nothing.                             |
| M exfiltrates            | `null`                                 | R             | M          | **V**           | M disposes of the token without further S interaction.               |
| R reclaims V1 management | `null`                                 | R             | R          | **V**           | Repairing the input name's management does not recover S's V2 token. |

### Attack sequence

{% stepper %}
{% step %}

## S owns a transferable V2 name

S opens its real Explorer ownership page.
{% endstep %}

{% step %}

## The recipient name has split V1 authority

The intended recipient is represented by an active, unwrapped V1 `.eth` name whose BaseRegistrar registrant is R.

That V1 name has delegated Registry Manager M and no resolver/address record. Canonical forward resolution therefore returns `null`.
{% endstep %}

{% step %}

## S enters the V1 name into the transfer form

S clicks `Transfer` and enters the V1 name in the `ENS name or address` field.
{% endstep %}

{% step %}

## Explorer substitutes the Manager

Explorer's shared resolver calls the canonical Universal Resolver, receives `null`, and substitutes M through the role-grant owner fallback.
{% endstep %}

{% step %}

## The UI treats the input as resolved

`useAddressResolution` reports the input as resolved.

Explorer displays M's full address, omits `Could not resolve an address for …`, and enables `Transfer name`.
{% endstep %}

{% step %}

## S starts the default transfer flow

S leaves all transfer settings at their UI defaults and starts the transfer.

The default flow requests two wallet approvals: source-name resolver detachment and token transfer.
{% endstep %}

{% step %}

## Explorer transfers the V2 token to M

The second mined transaction targets the V2 Registry and decodes exactly to `safeTransferFrom(S, M, tokenId, 1, 0x)`.

S's token balance becomes `0`, M's becomes `1`, and R's remains `0`.
{% endstep %}

{% step %}

## The token is moved and cannot be recovered through V1 reclaim

A direct transfer attempt from former owner S reverts.

M transfers the token to separate vault V; the transaction succeeds and V's balance becomes `1`.

R successfully executes the canonical V1 `BaseRegistrar.reclaim` operation, restoring R as V1 Registry owner.

The unrelated V2 token remains at V. No further S action or signature is required after the Portal transfer.
{% endstep %}
{% endstepper %}

### Preconditions and practical likelihood

#### Internal preconditions

1. The victim controls a transferable V2 name.
2. The victim uses Explorer's supported `ENS name or address` recipient input.
3. The entered name has no canonical ETH address result.
4. ENS ownership lookup nevertheless returns a non-zero address different from the sender.
5. The victim trusts Explorer's resolution result and approves the displayed default transaction sequence.

#### V1 split-authority condition demonstrated by the PoC

The strongest concrete case is an active, unwrapped V1 `.eth` 2LD where:

```
BaseRegistrar.ownerOf(labelhash) = R
ENSRegistry.owner(namehash)      = M
ENSRegistry.resolver(namehash)   = 0x0
UniversalResolver address        = null
R != M
```

This is supported ENS state. The registrant can delegate Registry management without transferring the registration token. The PoC verifies the registration is unexpired and reads every value independently from the local Sepolia fork before the browser action.

The exploit does not require a malicious ENS administrator, protocol multisig, registrar controller, application operator, connected-wallet provider, RPC provider, or transaction relayer. M is simply an address controlling the separate recipient name. Its authority over that name is not authority over S or S's V2 token.

The report does not hide the required user interaction: Explorer shows M's full address and warns generally that transfers are irreversible. S must still approve the transactions. That is the interaction explicitly contemplated by impact 6285: the application displays or pre-fills an incorrect address or transaction detail and the user submits a transaction to an unintended recipient. The security defect is that Explorer labels M as the successful interpretation of the ENS name and suppresses the specific unresolved-name warning.

### Why existing controls do not stop the issue

| Existing control                      | What it establishes                              | Why the unintended transfer remains possible                                           |
| ------------------------------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------- |
| ENS name syntax validation            | Input is syntactically a name                    | It does not prove the name has an ETH address record.                                  |
| Universal Resolver lookup             | Produces the canonical address or `null`         | Its `null` result is overwritten by the owner fallback.                                |
| `useAddressResolution` status         | A non-null query result exists                   | It cannot distinguish an address record from ownership data.                           |
| Unresolved warning                    | Alerts users when status is `unresolved`         | The fallback changes the status to `resolved`, suppressing it.                         |
| Self/zero-address checks              | Recipient is neither S nor zero                  | M is a non-zero third-party address, so it passes.                                     |
| Recipient preview                     | Shows the address Explorer will use              | It presents the fallback as a resolved recipient and does not disclose its provenance. |
| General irreversible-transfer warning | Tells S to check the recipient                   | The application itself supplied M as the result of S's ENS-name input.                 |
| Default resolver-detach step          | Detaches the **source V2 name's** resolver       | It does not validate the recipient name and does not change the token destination.     |
| Wallet approval                       | S authorizes generated calldata                  | The approved calldata contains the incorrect recipient produced by Explorer.           |
| ERC-1155 receiver/ownership checks    | Enforce the submitted transfer                   | They correctly transfer the token to M; they do not know the user's ENS-name intent.   |
| V1 registrant reclaim                 | Restores management of the **recipient V1 name** | It cannot reverse the separate V2 NFT transfer.                                        |

### Known-issue distinction

This finding does not share a root cause with the disclosed competition issues most closely related to Explorer inputs or transfers:

| Disclosed issue | Distinction                                                                                                                                                                                                                                                                   |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `EXP-INPUT-005` | Concerns ENS-normalization-invalid or transformed inputs, including homoglyphs, bidi/control characters, and leading/trailing dots. ENS-004 uses the ordinary normalized name `fgeorgescu.eth`; the defect is substituting Registry ownership for missing forward resolution. |
| `WEB-1305`      | Concerns transferring V2 **roles** to a V1 name remaining pending instead of being blocked. ENS-004 completes an ERC-1155 V2 **token ownership transfer** to the wrong address.                                                                                               |
| `QA-01`         | Concerns certain transfer role combinations or names with subnames later breaking record updates or primary-name setting. ENS-004 changes the immediate token recipient and demonstrates completed loss plus exfiltration.                                                    |
| `WEB-1257`      | Concerns registry/primary-name state retained after a parent transfer. ENS-004 has no parent transfer or subname dependency; its encoded recipient is wrong.                                                                                                                  |
| `QA-06`         | Concerns a deleted or non-resolving subname remaining displayed as a primary name and later profile errors. ENS-004 concerns recipient resolution in the transfer form and a successfully mined V2 NFT transfer.                                                              |
| `EXP-4337-003`  | Concerns `request.from` not being compared with the connected EOA. Here the sender and connected wallet are correct; the incorrect value is the `to` argument nested inside Portal-generated `safeTransferFrom` calldata.                                                     |

The ENS contracts behave correctly in the PoC. The V2 Registry transfers the token exactly as Explorer requests. The root cause is the in-scope Explorer application reusing a role-oriented owner fallback in a transfer-recipient resolution boundary.

## Impact Details

The appropriate competition impact is:

> **Critical (6285) — Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as resolving or rendering an ENS name to an attacker-controlled address or showing the wrong recipient or amount on a transaction the user then approves.**

The E2E proves every element of that impact:

* **Incorrect displayed address:** canonical forward resolution is `null`, but Explorer visibly presents M as the resolution of `fgeorgescu.eth`.
* **Incorrect pre-filled transaction detail:** the production transfer helper inserts M into the ERC-1155 `to` argument.
* **User interaction:** S uses the normal transfer form and authorizes the two transactions requested by the default flow.
* **Submission to the unintended recipient:** the mined transfer receipt succeeds and calldata decodes exactly to `safeTransferFrom(S, M, tokenId, 1, 0x)`.
* **Concrete loss:** S's balance falls from `1` to `0`; R never receives the token; M receives it and transfers it onward.
* **No automatic recovery:** former owner S cannot transfer the token, and R reclaiming the V1 name's management does not recover it.

The affected asset is a V2 ENS name NFT. The report does not claim that the local-fork token has production market value, that every unresolved name is adversarial, or that protocol-level recovery is impossible. It demonstrates the scoped application failure and resulting ownership transfer deterministically on a pinned local fork without modifying live state.

## Risk Breakdown

| Gate                                                 | Verdict | Evidence                                                                                                                       |
| ---------------------------------------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
| Is the root cause in an in-scope asset?              | **YES** | The fallback, shared resolution query/hook, recipient form, and transfer helper are all under `apps/portal`.                   |
| Does canonical resolution actually fail?             | **YES** | The PoC asserts resolver `0x0` and Universal Resolver result `null`.                                                           |
| Does Explorer nevertheless present a recipient?      | **YES** | Live DOM evidence records M visible, zero unresolved warnings, and an enabled transfer button.                                 |
| Does the victim use the real UI and wallet flow?     | **YES** | Playwright drives Explorer and the repository's headless wallet provider through both default approvals.                       |
| Is the wrong address present in mined calldata?      | **YES** | The exact successful Registry transaction decodes to `safeTransferFrom(S, M, tokenId, 1, 0x)`.                                 |
| Does ownership actually change?                      | **YES** | Post-transfer balances are S = 0, M = 1, R = 0.                                                                                |
| Can the unintended recipient dispose of the NFT?     | **YES** | M-to-V `safeTransferFrom` succeeds; balances become M = 0 and V = 1.                                                           |
| Can S still transfer it?                             | **NO**  | The explicit post-transfer attempt from S reverts.                                                                             |
| Does V1 reclaim recover it?                          | **NO**  | R successfully reclaims V1 management while the V2 token remains at V.                                                         |
| Does exploitation require a malicious protocol role? | **NO**  | M has authority only over the separate recipient name; no protocol administrator or trusted application operator participates. |

Severity is Critical under the competition's explicitly scoped impact 6285. Likelihood depends on a user transferring to a name that lacks an address record but has a non-zero owner/Manager fallback, and then trusting Explorer's resolution presentation. That conditional likelihood does not change the demonstrated impact.

## Recommended Remediation

### Required fix

Separate role-grant subject resolution from transfer-recipient resolution.

For a transfer recipient, accept only:

1. a syntactically valid raw address supplied by the user; or
2. a non-zero address returned by canonical forward resolution of the entered ENS name.

If canonical forward resolution returns `null` or zero, return `null`, keep the unresolved warning visible, disable the transfer button, and do not build transaction calldata.

In particular, never substitute any of the following for a missing transfer address record:

* ENS Registry owner/Manager;
* BaseRegistrar registrant;
* NameWrapper owner;
* V2 token owner;
* role holder; or
* smart-account owner.

### Defense in depth

* Give `resolveAddressOrName` a purpose-specific API or split it into `resolveRoleSubject` and `resolveTransferRecipient` so unsafe reuse is difficult.
* Preserve resolution provenance in the return type, for example `{ address, source: 'raw' | 'address-record' | 'owner-fallback' }`, and require transfer code to accept only `raw` or `address-record`.
* Add a final transfer-boundary assertion before `startTransfer` and again before encoding calldata.
* In the review and wallet-confirmation UI, show the original input, final recipient, and resolution source.
* Add a regression test for an unwrapped V1 `.eth` name with `registrant != manager`, resolver `0x0`, and canonical address `null`; the form must show the unresolved warning, keep submission disabled, and encode neither authority address.
* Add equivalent tests for wrapped V1 names, V2 names, subnames, expired names, and resolver contracts that return zero or revert.
* Keep the ordinary positive controls: a raw address must pass through unchanged, and a name with a real address record must resolve to exactly that record.

Changing only the warning copy is insufficient. The security boundary must prevent owner-derived addresses from entering asset-transfer calldata.

## References

* [ENS competition scope and known issues](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [`resolveAddressOrName` owner fallback](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/roles/helpers/addUser.handlers.ts#L11-L40)
* [`resolveEnsOwner` V1 owner selection](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/ens/resolveEnsOwner.ts#L72-L104)
* [Repository documentation of registrant versus Registry controller](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/migration/hooks/useMigrationStatus.ts#L29-L40)
* [Shared `getResolvedAddress` query](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/queries/getResolvedAddress.ts#L13-L45)
* [`useAddressResolution` status derivation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/hooks/useAddressResolution.ts#L47-L89)
* [`AddressNameInput` unresolved warning](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/address/components/AddressNameInput.tsx#L54-L87)
* [`SendNameForm` recipient validation and submission](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L81-L108)
* [`RecipientPreview` presentation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L253-L281)
* [`transferToken` ERC-1155 encoding](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/helpers/transferToken.ts#L46-L90)
* [ENSIP-1 name resolution specification](https://docs.ens.domains/ensip/1/)
* [ENS terminology: Manager and Owner](https://docs.ens.domains/terminology/)

## Link to Proof of Concept

<https://gist.github.com/0xgo4ko/65fb7fa584049c2a6de952e4148f4737>

## Proof of Concept

### Secret Gist environment and integrity

The complete runnable PoC package is provided in the GitHub Secret Gist below. The Gist supports—but does not replace—the explanation, decisive assertions, reproduction steps, and preserved result in this report. A Gist is a Git repository and GitHub provides a Download ZIP action, so reviewers can either clone the exact pinned revision or download the complete package.

| Gist reference     | Value                                                              |
| ------------------ | ------------------------------------------------------------------ |
| Web URL            | <https://gist.github.com/0xgo4ko/65fb7fa584049c2a6de952e4148f4737> |
| Clone URL          | <https://gist.github.com/65fb7fa584049c2a6de952e4148f4737.git>     |
| Pinned Gist commit | `5b25ce36d437692cd4ddb1ec5b0fdb6b826dc17e`                         |

Reviewers should read `RUN.md` first and follow it as the authoritative package-verification, environment-setup, reproduction, and cleanup guide. The essential commands are reproduced below so this report remains self-contained.

GitHub Secret Gists are unlisted rather than access-controlled private repositories. The URL should therefore remain inside this private Immunefi submission. The package contains only deterministic local-fork fixture material and preserved test evidence; it contains no production private keys or live credentials.

The complete Gist repository root contains exactly these nine files:

| File                                   | Purpose                                                                                                       |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `RUN.md`                               | Locked-revision setup, integrity checks, execution, expected assertions, fixture boundary, and cleanup.       |
| `EVIDENCE.md`                          | Concise index of the preserved environment and decisive outcomes.                                             |
| `HYP-021-portal.e2e.spec.ts`           | Complete production-browser/local-fork exploit and negative controls.                                         |
| `HYP-021.playwright.config.mjs`        | Local Portal server, one-worker execution, isolated output, and trace configuration.                          |
| `ENS-004-mined-transaction-proof.json` | Preserved DOM observation, on-chain state, decoded transactions, receipts, balances, and controls.            |
| `ENS-004-playwright-report.json`       | Unmodified machine-generated successful Playwright report.                                                    |
| `ENS-004-success-trace.zip`            | Supplemental successful-run Playwright trace with executed source, DOM/actions, network, and console context. |
| `package.json`                         | ESM and pinned pnpm metadata.                                                                                 |
| `SHA256SUMS`                           | Integrity manifest covering all eight non-manifest package files.                                             |

There is intentionally no screenshot or video artifact. The decisive UI values are asserted against the live DOM and preserved in the proof JSON, while the trace supplies inspectable browser execution context. The exploit's security impact is established by decoded mined transactions and post-transaction token balances, not visual media.

**Role of the supplied trace.** `ENS-004-success-trace.zip` is not an input required for the exploit to execute and does not need to be extracted before running the PoC. The source and sidecar configuration run the browser exploit independently and generate a fresh trace below `reproduction-output/`. The supplied ZIP is preserved evidence of the original successful execution. It must remain unchanged in the submitted package because `SHA256SUMS` covers it, `RUN.md` validates its ZIP integrity, and reviewers can inspect the prior successful run before reproducing it. Removing or replacing it causes package verification to fail.

Clone and verify the exact submitted Gist revision:

```sh
git clone "https://gist.github.com/65fb7fa584049c2a6de952e4148f4737.git" ENS-004-PoC
git -C ENS-004-PoC checkout --detach \
  5b25ce36d437692cd4ddb1ec5b0fdb6b826dc17e
test "$(git -C ENS-004-PoC rev-parse HEAD)" = \
  5b25ce36d437692cd4ddb1ec5b0fdb6b826dc17e

cd ENS-004-PoC
if command -v shasum >/dev/null 2>&1; then
  shasum -a 256 -c SHA256SUMS
else
  sha256sum -c SHA256SUMS
fi
unzip -tq ENS-004-success-trace.zip
```

Every manifest entry must report `OK`, and ZIP validation must report no compressed-data errors.

### Evidence boundary

This is a production-browser/local-chain E2E:

* Explorer runs from the exact locked repository source through its real Vite entrypoint.
* Playwright interacts with the real ownership and transfer pages.
* The repository's headless E2E wallet provider receives and authorizes the actual requests.
* Canonical recipient resolution, form state, transaction construction, Registry calls, mining, receipt decoding, and balance reads are not mocked.
* The local fork is pinned to Sepolia block `11558468` and bound to `127.0.0.1`.
* The test checks the repository commit and clean tracked source before setup and again before writing evidence.

Direct harness transactions are used only to create deterministic preconditions and to prove post-transfer consequences: M's exfiltration, S's failed transfer, and R's V1 reclaim. The vulnerable transfer itself is performed entirely by the live Explorer UI and wallet flow.

The test impersonates the existing wrapped-name holder only to transform a canonical Sepolia fixture into an active unwrapped V1 name with R as registrant, M as Manager, and no resolver. This is a fixture shortcut to reach supported ENS state; it is not an attacker permission claimed by the report. The test also clears fork-inherited EIP-7702 code from deterministic fixture EOAs so unrelated ERC-1155 receiver behavior does not affect the result.

No live Sepolia transaction is sent. Every preserved transaction hash identifies a transaction mined only on the local Anvil fork.

### Decisive exploit assertions

The following abridged, non-contiguous excerpt from the complete pinned `HYP-021-portal.e2e.spec.ts` places the central proof directly in the report. Fixture helpers, polling utilities, error diagnostics, and evidence serialization are omitted; reproduction executes the complete Gist source.

```ts
// Establish and verify the supported split-authority recipient state.
const recipientRegistrantBeforeTransfer = await publicClient.readContract({
  address: CONTRACTS.ensBaseRegistrarImplementation.address,
  abi: BaseRegistrarAbi,
  functionName: "ownerOf",
  args: [BigInt(labelhash)],
})
const recipientManagerBeforeTransfer = await publicClient.readContract({
  address: CONTRACTS.ensLegacyRegistry.address,
  abi: LegacyRegistryAbi,
  functionName: "owner",
  args: [node],
})
const recipientResolverBeforeTransfer = await publicClient.readContract({
  address: CONTRACTS.ensLegacyRegistry.address,
  abi: LegacyRegistryAbi,
  functionName: "resolver",
  args: [node],
})
const canonicalForwardResolution = await getEnsAddress(ensClient, {
  name: RECIPIENT_NAME,
  universalResolverAddress: CONTRACTS.ensUniversalResolver.address,
})
expect(recipientRegistrantBeforeTransfer).toBe(registrant.address)
expect(recipientManagerBeforeTransfer).toBe(manager.address)
expect(recipientResolverBeforeTransfer).toBe(zeroAddress)
expect(canonicalForwardResolution).toBeNull()

// The real Explorer UI misclassifies M as the resolved recipient.
const recipientInput = page.getByPlaceholder("ENS name or address")
await recipientInput.fill(RECIPIENT_NAME)
const displayedManager = page.getByText(manager.address, { exact: true })
await expect(displayedManager).toBeVisible({ timeout: 30_000 })
const unresolvedWarning = page.getByText(
  `Could not resolve an address for “${RECIPIENT_NAME}”`,
)
await expect(unresolvedWarning).toHaveCount(0)

const transferButton = page.getByRole("button", { name: "Transfer name" })
await expect(transferButton).toBeEnabled({ timeout: 30_000 })
expect({
  recipientInput: await recipientInput.inputValue(),
  displayedManager: (await displayedManager.textContent())?.trim() ?? "",
  managerVisible: await displayedManager.isVisible(),
  unresolvedWarningCount: await unresolvedWarning.count(),
  transferButtonEnabled: await transferButton.isEnabled(),
}).toEqual({
  recipientInput: RECIPIENT_NAME,
  displayedManager: manager.address,
  managerVisible: true,
  unresolvedWarningCount: 0,
  transferButtonEnabled: true,
})

// The test never changes the UI defaults.
const defaultOptionStates = await readDefaultTransferOptions(page)
expect(defaultOptionStates).toEqual([
  {
    label: "Set the ETH address to the recipient",
    visible: false,
    checked: null,
    disabled: null,
  },
  {
    label: "Detach the resolver",
    visible: true,
    checked: true,
    disabled: false,
  },
  {
    label: "Detach the registry",
    visible: false,
    checked: null,
    disabled: null,
  },
])
const defaultTransferSteps = getDefaultTransferSteps(defaultOptionStates)
expect(defaultTransferSteps).toEqual([
  "detach-resolver",
  "transfer-token",
])

await transferButton.click()
const walletApprovalCount = await driveTransferToSuccess(
  page,
  wallet,
  defaultTransferSteps.map((step) => `transfer-${sourceName}-${step}`),
)

// Decode the two real sender-to-Registry transactions.
expect(senderRegistryTransactions).toHaveLength(2)
const [resolverDetach, submitted] = senderRegistryTransactions

const decodedResolverDetach = decodeFunctionData({
  abi: V2RegistryAbi,
  data: resolverDetach.input,
})
expect(decodedResolverDetach.functionName).toBe("setResolver")
expect(decodedResolverDetach.args).toEqual([
  BigInt(keccak256(stringToBytes(sourceLabel))),
  zeroAddress,
])
expect(resolverDetachReceipt.status).toBe("success")

const decoded = decodeFunctionData({
  abi: erc1155Abi,
  data: submitted.input,
})
expect(decoded.functionName).toBe("safeTransferFrom")
expect(decoded.args).toEqual([
  sender.address,
  manager.address,
  tokenId,
  1n,
  "0x",
])
expect(submitted.to?.toLowerCase()).toBe(V2_REGISTRY.toLowerCase())
expect(portalTransferReceipt.status).toBe("success")
expect(await balanceOf(sender.address)).toBe(0n)
expect(await balanceOf(manager.address)).toBe(1n)
expect(await balanceOf(registrant.address)).toBe(0n)

// Former owner S cannot transfer the NFT after Explorer sent it to M.
await expect(
  send(
    sender,
    V2_REGISTRY,
    encodeFunctionData({
      abi: erc1155Abi,
      functionName: "safeTransferFrom",
      args: [sender.address, attackerVault.address, tokenId, 1n, "0x"],
    }),
  ),
).rejects.toThrow()

// M can exfiltrate the token without another S action.
const exfiltrationData = encodeFunctionData({
  abi: erc1155Abi,
  functionName: "safeTransferFrom",
  args: [manager.address, attackerVault.address, tokenId, 1n, "0x"],
})
const exfiltrationHash = await send(manager, V2_REGISTRY, exfiltrationData)
expect(exfiltrationReceipt.status).toBe("success")
expect(await balanceOf(manager.address)).toBe(0n)
expect(await balanceOf(attackerVault.address)).toBe(1n)

// Reclaim repairs the V1 name's Manager but cannot recover the V2 token.
const reclaimHash = await send(
  registrant,
  CONTRACTS.ensBaseRegistrarImplementation.address,
  encodeFunctionData({
    abi: BaseRegistrarAbi,
    functionName: "reclaim",
    args: [BigInt(labelhash), registrant.address],
  }),
)
expect(reclaimReceipt.status).toBe("success")
expect(
  await publicClient.readContract({
    address: CONTRACTS.ensLegacyRegistry.address,
    abi: LegacyRegistryAbi,
    functionName: "owner",
    args: [node],
  }),
).toBe(registrant.address)
expect(await balanceOf(manager.address)).toBe(0n)
expect(await balanceOf(attackerVault.address)).toBe(1n)
```

### Preserved successful result

The machine-generated report records:

```
1 passed (23.0s)
test body: 18.4s
expected: 1
unexpected: 0
errors: []
```

The proof JSON preserves the exact successful run. The decisive subset is:

```json
{
  "repositoryCommit": "cda79acaad59711b943fc68207ebb3f1d0ff8596",
  "repositoryCommitVerified": true,
  "repositoryTrackedSourceClean": true,
  "upstreamSnapshot": "63772fd872af472ced58b009499355f3430c2a86",
  "recipientInput": "fgeorgescu.eth",
  "uiObservation": {
    "displayedManager": "0x976EA74026E726554dB657fA54763abd0C3a0aa9",
    "managerVisible": true,
    "unresolvedWarningCount": 0,
    "transferButtonEnabled": true
  },
  "forkBlockNumber": 11558468,
  "recipientStateBeforeTransfer": {
    "resolver": "0x0000000000000000000000000000000000000000",
    "canonicalForwardResolution": null,
    "manager": "0x976EA74026E726554dB657fA54763abd0C3a0aa9",
    "registrant": "0x9965507D1a55bcC2695C58ba16FB37d819B0A4dc"
  },
  "portalTransfer": {
    "transactionHash": "0x6e9a03d26672f3597c1918dbe9cef91ffe23f14d3a824a9e087d4762b0a70906",
    "functionName": "safeTransferFrom",
    "arguments": {
      "from": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
      "to": "0x976EA74026E726554dB657fA54763abd0C3a0aa9",
      "amount": "1",
      "data": "0x"
    },
    "receiptStatus": "success"
  },
  "defaultTransferOptions": {
    "changedByTest": false,
    "executedSteps": ["detach-resolver", "transfer-token"],
    "walletApprovalCount": 2
  },
  "balancesAfterPortalTransfer": {
    "sender": "0",
    "manager": "1",
    "registrant": "0"
  },
  "attackerExfiltration": {
    "transactionHash": "0xd6714411954e314c8711ec29923e30a58bbb6bc83abce0e69a1ca2f05f75667c",
    "receiptStatus": "success",
    "balancesAfter": { "manager": "0", "attackerVault": "1" }
  },
  "postTransferNegativeControl": {
    "formerSenderTransfer": "reverted"
  },
  "v1RegistrantReclaim": {
    "transactionHash": "0x05fdcc88f6f4eb10fdb86ff0c3d81103d7656007963bcaf18888c8364f28d427",
    "receiptStatus": "success",
    "recoveredV2Token": false
  }
}
```

All hashes in that JSON are local-fork transaction identifiers and will not appear on Sepolia Etherscan.

### Exact snapshot

| Snapshot                                         | Commit                                     |
| ------------------------------------------------ | ------------------------------------------ |
| Official audited upstream revision               | `63772fd872af472ced58b009499355f3430c2a86` |
| Immunefi mirror import used by the preserved run | `cda79acaad59711b943fc68207ebb3f1d0ff8596` |
| Pinned Secret Gist revision                      | `5b25ce36d437692cd4ddb1ec5b0fdb6b826dc17e` |

The preserved run checks the repository revision and clean tracked source before creating state and again before writing its proof. The Gist commit is pinned separately so later Gist edits cannot silently change the source or artifacts reviewed with this report.

### Required layout

```
ens004-repro/
├── audit-comp-ens/    # official repository at cda79aca...
└── ENS-004-PoC/       # complete Gist package at 5b25ce36...
```

When cloning, provide `ENS-004-PoC` as the destination directory:

```sh
git clone "https://gist.github.com/65fb7fa584049c2a6de952e4148f4737.git" ENS-004-PoC
```

This creates the correct name automatically. When using GitHub's **Download ZIP** action, extract the archive and rename its Gist-derived top-level directory to `ENS-004-PoC`, then place it beside `audit-comp-ens`. This makes every command in `RUN.md` work unchanged. The sibling layout matters because the sidecar spec and configuration import the frozen repository's E2E fixtures and start its Portal source through relative paths.

### Requirements

* macOS, Linux, or WSL2 with a POSIX shell
* Git
* Docker
* Node.js 22
* Corepack and pnpm 10.27.0
* `unzip`
* Chromium installed through Playwright
* a Sepolia RPC endpoint with historical state at block `11558468`, reachable from Docker
* free local ports `8545` and `3001`
* internet access to clone the repository, install its locked dependencies, fetch Chromium, pull the pinned Foundry image, and access the supplied historical RPC

### Prepare and verify the package

From a clean parent directory:

```sh
mkdir ens004-repro
cd ens004-repro

git clone "https://gist.github.com/65fb7fa584049c2a6de952e4148f4737.git" ENS-004-PoC
git -C ENS-004-PoC checkout --detach \
  5b25ce36d437692cd4ddb1ec5b0fdb6b826dc17e
test "$(git -C ENS-004-PoC rev-parse HEAD)" = \
  5b25ce36d437692cd4ddb1ec5b0fdb6b826dc17e

REPRO_ROOT="$(pwd)"
POC_DIR="$REPRO_ROOT/ENS-004-PoC"
REPO_DIR="$REPRO_ROOT/audit-comp-ens"

cd "$POC_DIR"
if command -v shasum >/dev/null 2>&1; then
  shasum -a 256 -c SHA256SUMS
else
  sha256sum -c SHA256SUMS
fi
unzip -tq ENS-004-success-trace.zip
```

### Prepare the exact audited source

```sh
git clone https://github.com/immunefi-team/audit-comp-ens.git "$REPO_DIR"
git -C "$REPO_DIR" checkout --detach \
  cda79acaad59711b943fc68207ebb3f1d0ff8596

test "$(git -C "$REPO_DIR" rev-parse HEAD)" = \
  cda79acaad59711b943fc68207ebb3f1d0ff8596
test "$(git -C "$REPO_DIR" show -s --format=%s HEAD)" = \
  "fork ensdomains/apps-monorepo at 63772fd872af472ced58b009499355f3430c2a86"
test -z "$(git -C "$REPO_DIR" status --porcelain --untracked-files=no)"

cd "$REPO_DIR"
corepack enable
corepack prepare pnpm@10.27.0 --activate
test "$(pnpm --version)" = "10.27.0"
pnpm install --frozen-lockfile
pnpm --dir e2e exec playwright install chromium

ln -sfn ../audit-comp-ens/e2e/node_modules "$POC_DIR/node_modules"
```

The relative symlink allows the standalone PoC to resolve Playwright, viem, ENSjs, and the repository's E2E fixtures without copying or modifying tracked repository source.

### Start the pinned local Anvil fork

In terminal 1, replace the placeholder with a historical Sepolia endpoint:

```sh
export SEPOLIA_FORK_URL='https://YOUR_SEPOLIA_RPC_ENDPOINT'
docker run --rm --name ens004-anvil -p 127.0.0.1:8545:8545 \
  --entrypoint anvil \
  ghcr.io/foundry-rs/foundry@sha256:8347b728d5d393dac1c018691b36f506d23b9dcd78341d40ea0fcb11c3a19cdd \
  --fork-url "$SEPOLIA_FORK_URL" \
  --fork-block-number 11558468 \
  --chain-id 11155111 --host 0.0.0.0
```

This is the only process that contacts the supplied historical RPC. All state modifications are mined locally.

### Run the browser E2E

In terminal 2:

```sh
cd /absolute/path/to/ens004-repro
REPRO_ROOT="$(pwd)"
POC_DIR="$REPRO_ROOT/ENS-004-PoC"
REPO_DIR="$REPRO_ROOT/audit-comp-ens"

docker exec ens004-anvil cast rpc anvil_nodeInfo \
  --rpc-url http://127.0.0.1:8545
# Continue only if forkConfig.forkBlockNumber is 11558468.

test ! -e "$POC_DIR/reproduction-output"

cd "$REPO_DIR"
ANVIL_RPC_URL=http://127.0.0.1:8545 \
PORTAL_APP_URL=http://127.0.0.1:3001 \
./e2e/node_modules/.bin/playwright test \
  --config "$POC_DIR/HYP-021.playwright.config.mjs"
```

The Playwright configuration starts Explorer itself on `127.0.0.1:3001`; no separate browser or application-server command is required. Chromium is used because the vulnerable boundary includes DOM state, recipient rendering, enabled/disabled form behavior, transaction-manager progression, and wallet requests in the real Portal UI.

The test writes fresh proof JSON, runner JSON, trace data, and attachments only below `ENS-004-PoC/reproduction-output/`. The preserved checksummed root evidence remains unchanged.

### Expected result and fresh-output verification

Expected terminal result:

```
1 passed
```

Verify the fresh runner and confirm the preserved package still matches its manifest:

```sh
cd "$POC_DIR"
node --input-type=module -e '
  import fs from "node:fs";
  const report = JSON.parse(fs.readFileSync("reproduction-output/ENS-004-playwright-report.json", "utf8"));
  const envKeys = Object.keys(report.config.webServer.env).sort();
  if (report.stats.expected !== 1 || report.stats.unexpected !== 0 || report.errors.length !== 0) process.exit(1);
  if (JSON.stringify(envKeys) !== JSON.stringify(["VITE_INDEXER_GRAPHQL_URL", "VITE_SEPOLIA_RPC_URL", "VITE_SEPOLIA_RPC_URL_SERVER"])) process.exit(1);
'

if command -v shasum >/dev/null 2>&1; then
  shasum -a 256 -c SHA256SUMS
else
  sha256sum -c SHA256SUMS
fi
```

### What the browser PoC proves

{% stepper %}
{% step %}

## The repository and fork are exact

The repository is at the exact locked competition mirror commit and its tracked source is clean.

The local Sepolia fork is pinned to block `11558468`.
{% endstep %}

{% step %}

## The recipient name has no canonical forward resolution

`fgeorgescu.eth` is active and unwrapped, R is its registrant, M is its distinct Manager, its resolver is zero, and canonical forward resolution is `null`.
{% endstep %}

{% step %}

## S owns a transferable V2 name

S owns one unit of a transferable V2 name token and has the required transfer role.
{% endstep %}

{% step %}

## Explorer misresolves the recipient

The real Explorer form accepts `fgeorgescu.eth`, visibly displays M, renders no unresolved warning, and enables transfer.
{% endstep %}

{% step %}

## The default transfer flow executes

The test changes no transfer option; the applicable resolver-detach option remains checked by default.

The real transaction manager obtains two wallet approvals and mines both default-flow transactions.
{% endstep %}

{% step %}

## The V2 token is sent to M

The second transaction targets the V2 Registry and decodes exactly to `safeTransferFrom(S, M, tokenId, 1, 0x)`.

The successful transfer changes balances to S = 0, M = 1, R = 0.
{% endstep %}

{% step %}

## The loss persists

Former owner S cannot transfer the token after losing it.

M can transfer the token to separate vault V without any further S action.

R can reclaim the V1 name's management, but that does not recover the separate V2 NFT.
{% endstep %}
{% endstepper %}

### Negative controls and scope boundaries

* **Canonical-resolution control:** the test independently calls `getEnsAddress` through the canonical Universal Resolver and requires `null` before the browser action.
* **Authority-separation control:** BaseRegistrar `ownerOf` must equal R while Registry `owner` equals distinct address M.
* **Transfer-role control:** S must hold `ROLE_CAN_TRANSFER_ADMIN` for the source V2 resource before opening Explorer.
* **Exact-calldata control:** the Portal transaction must target the V2 Registry and decode to the expected ERC-1155 function, sender, recipient, token ID, amount, and empty data.
* **Balance control:** S, M, and R balances are read from the V2 Registry after mining; impact is not inferred from UI success.
* **Former-sender control:** S's later transfer attempt must revert.
* **Exfiltration control:** M's transfer to V must succeed and produce V balance `1`.
* **Recovery control:** R's V1 reclaim must succeed but leave the V2 token at V.
* **Default-options control:** the test reads every transfer switch and proves it did not change them.
* **Repository-provenance control:** commit and tracked-source cleanliness must be equal before and after execution.

The PoC does not claim that all three optional switches are simultaneously displayed. In this fixture only `Detach the resolver` is applicable; it is visible, checked, and enabled by default. The other options are not rendered. That state is recorded explicitly so the result cannot be attributed to hidden test interaction.

### Test-only setup is not the exploit

The fixture uses Anvil-only capabilities to create deterministic starting state and source assets:

* impersonating the existing wrapped-name holder to unwrap the canonical recipient name;
* assigning R as registrant and M as Registry Manager;
* clearing the recipient resolver;
* funding deterministic local accounts;
* clearing fork-inherited EIP-7702 code from fixture EOAs; and
* creating a source V2 name for S through the repository's existing `makeV2Name` fixture.

Those actions create supported preconditions on an isolated fork. They do not patch Explorer, replace its resolver, modify the transfer implementation, alter tracked repository files, mock the wallet request, or perform the vulnerable transfer. The exploit phase begins when S enters `fgeorgescu.eth` in the live Portal form.

The mnemonic in the test is Anvil's standard public `test test … junk` mnemonic. It controls deterministic local fixture accounts only and must never be used for real assets.

### Evidence integrity

The manifest distributed in the pinned Gist is:

```
cd20e0c25087cfc81630cac09f9824f59d4b788cb33f542211b20b5d64e6985a  HYP-021-portal.e2e.spec.ts
f3356ca7cc444235a9d4064bb2ab69f75ff8f39cc97fcf20d94ccb2b6e7ec270  HYP-021.playwright.config.mjs
d83e9269cc6cd44cda9bba67c431940f4835d600c11dc714a72b1757351685f7  ENS-004-mined-transaction-proof.json
6b998d8c1b107241025a83737e6acdf782d74c924dc7b18bf92fb94a4b973c6b  ENS-004-playwright-report.json
8143b1442069230a161d64546a63d5a5a01b21fb10c0760d58215bb3cd1a8ce2  ENS-004-success-trace.zip
5909531a940084d0aa156d5ef3f99c61b437834b628b0068bb5343de3123a170  RUN.md
1e8fd6b9a6871bb2220c562144993ebed5b1e65a19171d2d31963ba51647627b  EVIDENCE.md
a25bb77f81b39437eb66fa81a71badf8bebfd958a117c6d591da124c36307d02  package.json
```

`SHA256SUMS` covers every distributed regular file except itself. Reproduction writes only below the new untracked `reproduction-output/` directory, preserving the submitted evidence. The successful trace is supplemental evidence, not a runtime input.

After review, stop only this reproduction's local fork:

```sh
docker stop ens004-anvil
```
