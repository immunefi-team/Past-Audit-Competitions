# 90544 w a critical post confirmation unicode normalization substitutes an attacker owned primary name causing manager to render an attacker controlled optimism recipient&#x20;

**Submitted on Aug 25th 2026 at 15:51:17 UTC by @sl1de for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90544
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

Manager preserves a raw decomposed Unicode (NFD) ENS name through registration, Dashboard ownership display, selection, and confirmation, but `setPrimaryNameWithHca` normalizes it to the composed NFC form only when constructing the owner-signed calldata. The joined local proof shows that these canonically equivalent strings identify distinct on-chain registrations: the victim owns the selected NFD name while the attacker owns the NFC name ultimately written as the victim’s primary name. By setting the NFC name’s ETH record to the victim, the attacker makes Manager’s production `getEnsName` call return the attacker-owned NFC name for the victim; an independent ENSjs strict-resolution control reports the same name with `match=true`. The attacker can simultaneously set that name’s Optimism record to an attacker-controlled address. A sender who starts from the victim’s address profile is therefore directed to a profile where Manager renders and copies the attacker’s address as the chain-specific Optimism recipient. Funds sent using that copied address go to the attacker instead of the intended victim.

## Vulnerability Details

### Attack scenario

There are three relevant actors:

* The attacker owns the composed NFC name and controls its resolver.
* The victim owns the visually identical decomposed NFD name and uses Manager’s normal primary-name flow.
* A sender relies on Manager’s receiving-address UI to obtain the victim’s Optimism address.

For example, these names render identically but contain different Unicode code points:

* Victim-owned raw NFD: `e\u0301-hca-1811111111.eth` (`U+0065 U+0301`)
* Attacker-owned NFC: `\u00E9-hca-1811111111.eth` (`U+00E9`)

The strings are canonically equivalent:

```
NFD.normalize("NFC") == NFC
```

However, their raw labels produce distinct registrar identities and can be registered to different owners.

The complete attack is:

{% stepper %}
{% step %}

## Register the NFC name

The attacker registers the NFC name and retains control of its resolver.
{% endstep %}

{% step %}

## Register the raw NFD name

The victim registers the visually identical raw NFD name using Manager’s accepted registration path. The frozen parser, availability call, and EOA/HCA builders preserve the raw bytes, and the fork proves the resulting raw registration remains distinct from NFC.
{% endstep %}

{% step %}

## Point the NFC ETH record to the victim

The attacker sets the NFC name’s ETH record, coin type 60, to the victim’s public EVM address.
{% endstep %}

{% step %}

## Select the NFD name in Manager

Manager’s Dashboard displays the victim-owned NFD row using Panoptes’ raw `name` field.

The victim selects that row as their primary name and approves the normal owner-signed HCA operation.
{% endstep %}

{% step %}

## Normalize after confirmation

Only after confirmation, `setPrimaryNameWithHca` calls `viem/ens.normalize`, changing the selected NFD string into the attacker-owned NFC string.

Manager encodes and submits `setNameWithHCA(victimAddress, NFCName)`.
{% endstep %}

{% step %}

## Retain distinct ownership

The victim retains ownership of the NFD name, while the attacker retains ownership and resolver control over the NFC name.
{% endstep %}

{% step %}

## Pass reverse resolution

Reverse resolution for the victim succeeds because Manager verifies only the default ETH record, and the attacker deliberately keeps NFC’s ETH(60) record equal to the victim.
{% endstep %}

{% step %}

## Set an attacker-controlled Optimism record

The attacker sets NFC’s Optimism record, coin type `2147483658`, to the attacker’s EVM address. This does not invalidate the ETH reverse match.
{% endstep %}

{% step %}

## Open the victim profile

A sender opens the victim’s address profile and follows Manager’s “View profile” link for the strictly resolved primary name.
{% endstep %}

{% step %}

## Render and copy the attacker recipient

Manager fetches the NFC name’s ETH and Optimism records. It treats ETH as the main receiving address and the differing attacker-controlled Optimism value as a “Chain-specific address.”

Clicking the Optimism card copies the attacker’s complete address to the clipboard. An Optimism payment made using that value goes to the attacker rather than the intended victim.
{% endstep %}
{% endstepper %}

The resulting identity transition is:

```
Victim selects and confirms victim-owned NFD name
                         |
                         v
Manager normalizes after confirmation
                         |
                         v
Owner-signed calldata contains attacker-owned NFC name
                         |
                         v
NFC ETH(60) = victim, so strict reverse match succeeds
                         |
                         v
NFC Optimism record = attacker
                         |
                         v
Manager renders and copies attacker as Optimism recipient
```

The attacker can subsequently rotate the Optimism record without further victim interaction because the attacker still owns the NFC name and controls its resolver.

No compromised RPC, indexer, backend, wallet, HCA, session, private key, or privileged contract role is required.

### Reachability prerequisite 1: registration preserves non-canonical label bytes

The registration parser uses a variable called `normalized`, but only trims and lowercases the input:

```typescript
const normalized = name.trim().toLowerCase()
```

The URL helper similarly adds `.eth` and truncates the label without applying ENSIP-15 normalization:

```typescript
export function normalizeDomainNameFromUrl(name: string): string {
  const label = name.toLowerCase().endsWith('.eth')
    ? name.slice(0, -4)
    : name

  return truncateToMaxBytes(label, MAX_DOMAIN_BYTES) + '.eth'
}
```

Neither `parseName` nor `validateENSName` requires the supplied string to equal its canonical ENSIP-15 form.

On the HCA registration path, `cleanLabel` removes only the `.eth` suffix:

```typescript
const cleanLabel = (name: string): string => name.replace(/\.eth$/, '')
```

The unchanged label is then passed to the registrar:

```typescript
args: [
  params.label,
  params.wallet,
  params.secret,
  ZERO_ADDRESS,
  params.resolver,
  params.duration,
  c.usdc,
  REFERER,
]
```

The frozen ENSjs availability export was also verified to call `isAvailable` with the decomposed label unchanged. Consequently, the target permits registration of the raw NFD label even when its NFC equivalent is already separately registered.

### Reachability prerequisite 2: Dashboard selects the raw victim-owned registration

Panoptes exposes `name` and `normalizedName` separately. A bounded run of the real Panoptes implementation against a raw NFD registration on the local fork returned the victim-owned row in this form:

```json
{
  "name": "e\u0301-indexer-1788888888.eth",
  "normalizedName": "\u00e9-indexer-1788888888.eth",
  "owner": {
    "id": "0xc9eec1b174a646d7c282820afe94acfba6c00a12"
  }
}
```

The Panoptes control used a pinned image and a test-only contract-address override matching the frozen deployment. It independently proves the raw and normalized field behavior used by the Dashboard test; it does not claim byte identity with an undisclosed production indexer build.

Manager deliberately prefers the raw field:

```typescript
export const resolveDomainLabel = (domain: {
  id: string
  name?: string | null
  normalizedName?: string | null
}): string => domain.name ?? domain.normalizedName ?? domain.id
```

`ChoosePrimaryNameDialog` renders that value and uses it as the selected identity:

```typescript
const label = resolveDomainLabel(domain)

<button
  aria-label={t`Select ${label} as primary name`}
  onClick={() => onSelectName(label)}
>
  {label}
</button>
```

After the normal resolver/address preflight, that same raw value is submitted:

```typescript
await submitPrimaryName({
  name: selectedName,
  owner: account.ownerAddress as Address,
})
```

At this point, the displayed, selected, and preflight-checked registration is still the victim-owned raw NFD name.

### Root cause: the selected identity changes only after confirmation

The submitted root cause is that Manager re-derives the identity after confirmation and changes the selected namehash without binding the signed operation to the selected owned registration.

The HCA primary-name service applies `viem/ens.normalize` after the raw name has already been selected and confirmed:

```typescript
const withEthSuffix = (name: string) =>
  normalize(name.endsWith('.eth') ? name : `${name}.eth`)
```

It then uses the changed string as the final contract argument:

```typescript
const cleanName = withEthSuffix(name)

data: encodeFunctionData({
  abi: reverseAdapterAbi,
  functionName: 'setNameWithHCA',
  args: [ownerAddress, cleanName],
})
```

The wallet check verifies that the signer controls `ownerAddress`, but there is no check that `cleanName` still represents the same on-chain registration selected in the dialog or that the victim owns the normalized name.

The resulting request is submitted through an owner signer and the production transaction manager:

```typescript
const ownerSigner: RhinestoneSigner = {
  ...signer,
  session: undefined,
}

const txId = transactionManager.startTransaction(
  { type: 'custom', request },
  ownerSigner,
  {
    description: `Set primary name to ${cleanName}`,
    operation: 'set-primary-name',
    name: cleanName,
  },
)
```

Normalizing an ENS name is not itself the vulnerability. The vulnerability is performing that normalization only after an ownership-bearing raw registration has already been displayed, checked, selected, and confirmed, without checking whether normalization changes it into a registration owned by another account.

The required invariant is:

```
selected registration == confirmed registration == signed primary identity
```

The vulnerable flow instead produces:

```
selected and checked registration = victim-owned NFD
signed primary identity           = attacker-owned NFC
```

Because the two strings render identically, ordinary rendered-name comparison gives the victim no visual indication of the code-point change.

### Impact-enabling trust mismatch: strict reverse verification covers ETH but not Optimism

Manager resolves an address’s primary name using:

```typescript
getEnsName(client, { address })
```

It does not supply a `coinType`, so the strict bidirectional check uses the default ETH coin type 60.

The attacker preserves:

```
Attacker-owned NFC ETH(60) = victim address
```

Manager’s production `getEnsName` call therefore returns the attacker-owned NFC name for the victim. The separate ENSjs strict-resolution control returns `{ name: NFCName, match: true }`, confirming why the Universal Resolver accepted it.

That verification places no constraint on:

```
Attacker-owned NFC Optimism(2147483658) = attacker address
```

This is not a fail-open in the ETH check. The guard correctly checks equality of the default ETH(60) forward record; it establishes neither name ownership nor the other coin records.

The negative control confirmed this boundary:

* Changing NFC’s ETH record from the victim to the attacker produced `ReverseAddressMismatch` and removed the victim’s primary result.
* Changing only the Optimism record left the strict victim match intact.

### Impact sink: Manager propagates the substituted identity into a copyable Optimism receiving address

The victim address route uses the strict reverse result as its primary name:

```typescript
const resolvedName = await queryClient.ensureQueryData(
  profileReverseNameQuery(address),
)

return <AddressProfileView address={address} primaryName={resolvedName} />
```

The address profile then presents a “View profile” link to that resolved name.

The profile service probes Manager’s complete supported address catalogue, including Optimism, even if the indexer reports no coin records:

```typescript
const coins = unique([
  ...alwaysProbeAddressRecords.map(Number),
  ...addressRecords.map((record) => record.coinType),
  ...indexerRecords.coins,
])

const records = await getRecords(client, {
  name,
  coins,
  contentHash: true,
  abi: true,
  ignoreInvalidCoinTypes: true,
})
```

Optimism is explicitly supported as coin type `2147483658`.

Manager chooses ETH as the main receiving address and classifies every differing record as chain-specific:

```typescript
const mainAddress =
  addresses.find((address) => address.coinType === ETH_COIN_TYPE) ??
  addresses[0]
```

```typescript
return getAddressItems(records)
  .filter((address) => address.value.toLowerCase() !== mainValue)
```

The resulting value is rendered under “Chain-specific addresses” and passed unchanged into `CopyableButton`:

```typescript
<CopyableButton value={address.value}>
  <ChainAddressValue value={address.value} />
</CopyableButton>
```

The click handler copies the complete value rather than the abbreviated visible representation:

```typescript
await copyToClipboard(value)
```

There is no Optimism-specific verification binding this copied recipient to the victim whose ETH strict-reverse result led the sender to the profile.

### Verified joined production-path proof

I reproduced the security-relevant chain through complementary joined proofs in a private local Sepolia fork. The complementary legs cover the frozen registration exports, a real Panoptes indexer control, the production Dashboard selection and primary-name service, HCA and transaction-manager execution through the local Warp/mockestrator path, real ENS registry and resolver state, production profile transformation, receiving-address rendering, and the production clipboard handler. Each mocked boundary is identified below and in the PoC.

The target was competition import `cda79acaad59711b943fc68207ebb3f1d0ff8596`, which imports upstream commit `63772fd872af472ced58b009499355f3430c2a86`.

| Item                             | Verified result                                                      |
| -------------------------------- | -------------------------------------------------------------------- |
| Victim-selected raw name         | `e\u0301-hca-1811111111.eth`                                         |
| Final owner-signed primary name  | `\u00E9-hca-1811111111.eth`                                          |
| Raw NFD owner                    | Victim: `0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12`                 |
| NFC owner                        | Attacker: `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`               |
| HCA/transaction-manager result   | Successful fill                                                      |
| Fill transaction                 | `0x0d8e3da3c75cb15ce030c919ac52ffe2451ee292a0905c15625f577043ee1c19` |
| Manager `getEnsName` result      | Attacker-owned NFC name                                              |
| Independent ENSjs strict control | Attacker-owned NFC name with `match=true`                            |
| Manager main ETH recipient       | Victim address                                                       |
| Optimism coin type               | `2147483658`                                                         |
| Manager Optimism recipient       | Attacker address                                                     |
| Clipboard result                 | Exact full attacker address                                          |
| ETH negative control             | Repointing ETH to attacker caused `ReverseAddressMismatch`           |

The clean joined component-to-fork run passed five tests across three files and exited successfully. The same fork state was preserved from the owner-signed HCA operation through the Manager record read, profile transformation, card rendering, and clipboard assertion. The additional frozen-package tests prove that Manager's parser, availability call, and both registration builders preserve the raw decomposed label.

The only mocked record-data response in the live recipient test is an empty Panoptes cache response. The mocked `safeGetClient` boundary injects the real local-fork client, and the happy-dom test environment’s `navigator.clipboard.writeText` is replaced with an observer; neither supplies an ENS or profile value. ENSjs, RPC, registry ownership, reverse resolution, ETH/BTC/Optimism record reads, profile transformation, address-card rendering, and the production copy handler all use the real local-fork state. Manager statically probes supported coin types, so an empty indexer coin list does not suppress the Optimism record.

BTC coin type 0 was also set to and copied as an attacker-selected recipient, confirming that the weakness is a general cross-coin identity split rather than an Optimism formatting issue.

### Difference from known issue EXP-INPUT-005

This report does not submit the known EXP-INPUT-005 validator/display-confusion issue.

EXP-INPUT-005 is described as a Medium issue involving validators accepting visually confusing Unicode, bidi/control characters, and leading or trailing dots. Raw registration acceptance is only a prerequisite here, not the submitted root cause. The new root cause is that Manager changes an already selected, ownership-bearing registration into a different actor’s registration after confirmation and then trusts that substituted identity across coin types. This produces a materially different consequence:

* Two distinct on-chain registrations are owned by different actors.
* Manager displays and preflight-checks the victim-owned raw registration.
* After confirmation, Manager signs the attacker-owned canonical identity.
* The attacker-owned name is accepted as the victim’s strict ETH primary result.
* Manager resolves, renders, and copies an attacker-controlled Optimism recipient from that substituted identity.

The competition rules state that materially severity-changing consequences of known issues remain eligible. They also specifically state that incorrect resolution beyond EXP-INPUT-005 is new and identify cross-chain resolver disagreement, including Optimism, as part of the Critical wrong-recipient attack surface.

### Preconditions and boundaries

The following requirements are explicit:

* The attacker must own the composed NFC name and control its resolver.
* The victim must own or register the visually equivalent raw NFD name.
* The victim must select that name as primary and approve the normal owner-signed request.
* The attacker must keep NFC’s ETH(60) record pointed at the victim.
* A sender must open the victim-linked profile and use the displayed/copied Optimism recipient.

The following are not claimed:

* The attacker does not take ownership of the victim’s raw NFD name.
* The attacker does not mutate the victim’s resolver.
* The attacker does not compromise the victim’s wallet, HCA, or session.
* Manager does not automatically broadcast the subsequent Optimism payment.
* The PoC does not claim that funds were transferred on a public network.
* A careful sender who separately inspects the NFC name’s registry owner could notice the discrepancy.

These boundaries do not remove the selected impact. Manager resolves a victim-linked primary identity to an attacker-controlled cross-chain receiving address and supplies that exact unintended recipient through its normal copy action.

### Recommended remediation

{% stepper %}
{% step %}

## Normalize at all identity boundaries

Apply ENSIP-15 normalization before availability, registration, indexing, display, selection, and every other identity boundary. Reject input when normalization changes the supplied bytes.
{% endstep %}

{% step %}

## Bind selection to registration ownership

Before requesting a primary-name signature, bind the transaction to the exact selected registration ID and recheck its live owner.
{% endstep %}

{% step %}

## Require explicit re-selection after normalization

If normalization changes the selected registration, stop and require a new explicit selection rather than silently continuing.
{% endstep %}

{% step %}

## Validate chain-specific receiving addresses

Do not treat an ETH-only strict reverse match as validation of chain-specific receiving addresses. Either verify each chain-specific association independently or clearly warn that those addresses are not bound by the ETH reverse check.
{% endstep %}

{% step %}

## Add regression coverage

Add regression tests using canonically equivalent names owned by different accounts.
{% endstep %}
{% endstepper %}

## Impact Details

This maps directly to the selected Critical Websites & Applications impact:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, including resolving or rendering an ENS name to an attacker-controlled address.

From the attacker’s perspective, the attacker retains ownership of the substituted NFC name, points its ETH record at the victim so the ETH strict-reverse check succeeds, and independently controls the Optimism receiving address.

From the victim’s perspective, the victim selects and approves their own raw NFD registration in Manager, but the signed primary-name operation instead associates their address with the attacker’s canonically equivalent NFC name.

From the sender’s perspective, Manager accepts the attacker-owned name as the victim address’s strict primary result, directs the sender to its profile, labels the attacker’s Optimism record as a chain-specific receiving address, and copies the complete attacker address. A payment made using that recipient goes to the attacker instead of the intended victim.

The possible loss is the amount the sender transfers using the substituted Optimism recipient. The attacker can rotate that record later while preserving the ETH record required for the victim’s strict reverse match.

This report deliberately selects only the program’s wrong-address/wrong-recipient Critical impact. It does not additionally claim direct theft, HCA takeover, name ownership takeover, or an automatically submitted malicious transaction.

No smart-contract vulnerability is claimed. The contracts are used only to demonstrate execution of the client-generated primary-name argument on a private fork.

## References

* [Registration route accepts and forwards the route name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/routes/register/$name.tsx#L20-L58)
* [Registration parser trims and lowercases without ENSIP-15 normalization](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/register-v2/utils/name-parser.ts#L33-L82)
* [The URL helper does not perform Unicode canonicalization](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/utils/domain.ts#L23-L33)
* [Name validation does not require canonical form](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/shared/registration/nameUtils.ts#L82-L143)
* [HCA registration removes only the `.eth` suffix](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/transaction-manager/src/machines/registration/registration.hca.actors.ts#L492-L504)
* [The raw label is passed to the registrar](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/smart-account/src/providers/rhinestone/registration-calls.ts#L280-L297)
* [Panoptes exposes raw `name` separately from `normalizedName`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/packages/indexer/documents/fragments/Domain.graphql#L1-L14)
* [Dashboard prefers the raw `name` field](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/dashboard/utils.ts#L42-L46)
* [The primary-name dialog renders and selects the raw value](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx#L69-L123)
* [Confirmation submits the raw `selectedName`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/dashboard/components/ChoosePrimaryNameDialog.tsx#L441-L475)
* [The HCA branch calls `setPrimaryNameWithHca`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/profile/hooks/useSetPrimaryName.ts#L43-L75)
* [Late normalization and final `setNameWithHCA` argument](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/profile/service/setPrimaryName.ts#L46-L49)
* [The normalized name enters the owner-signed transaction](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/profile/service/setPrimaryName.ts#L128-L226)
* [Reverse verification defaults to ETH coin type 60](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/profile/service/profileReverseName.ts#L23-L40)
* [The victim address route consumes the reverse result](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/routes/$address/index.tsx#L5-L22)
* [The address profile links to the resolved name profile](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/profile/components/view/AddressProfileHeader.tsx#L198-L250)
* [Profile records statically probe supported coin types and call ENSjs](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/profile/service/profileRecords.ts#L82-L139)
* [ETH, BTC, and Optimism are prioritized address records](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/profile/data/records/address.tsx#L5-L25)
* [Optimism is supported as coin type 2147483658](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/profile/components/dialogs/edit-profile/tabs/addresses/addressPickerRecords.ts#L16-L28)
* [ETH is selected as main and differing values become chain-specific](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/profile/components/view/ProfileView.helpers.ts#L229-L277)
* [The exact chain-specific value enters `CopyableButton`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx#L127-L198)
* [`CopyableButton` writes the complete value to the clipboard](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/manager/src/components/atoms/CopyableButton/CopyableButton.tsx#L13-L48)

## Proof of Concept

### Safety and environment

This proof runs entirely against a private local Anvil fork using throwaway test accounts. It does not send transactions to public Sepolia, Optimism, or any other public chain. It does not use production funds, names, wallets, sessions, target or production API credentials, or the public Rhinestone service. The evaluator supplies a researcher-controlled Sepolia RPC endpoint only as Anvil's read-only upstream state source.

The runner requires `SEPOLIA_FORK_URL` to be supplied explicitly and exits if it is absent. This prevents it from silently using any provider URL or credential committed in the repository. Anvil uses the supplied endpoint only to read the Sepolia state needed to create the local fork.

The joined exploit runner starts only the repository's pinned `anvil` and `mockestrator` containers. It:

* Refuses to touch pre-existing running Anvil or mockestrator services.
* Waits until both pinned services are healthy.
* Creates an outer `evm_snapshot`.
* Executes the exploit and component assertions.
* Reverts the snapshot in an exit trap.
* Stops only the two services it attempted to start, including after a startup or test failure.

A separate bounded Panoptes control uses the same pinned Anvil plus an exact Panoptes image digest, isolated container names, and an isolated Docker volume. It removes only those resources after the control completes or fails.

The standard Anvil development accounts and keys used by the proof are throwaway local-chain values.

### PoC evidence bundle

The complete runnable PoC and captured verification logs are available here:

[Download `ens-unicode-primary-poc.zip` from Google Drive](https://drive.google.com/file/d/1yYwv7gdPkAoFTj6DbqHRtID7_2e6wYjM/view?usp=sharing)

The share link is publicly readable without authentication. The archive can also be retrieved using the direct-download command in Step 3.

SHA-256:

```
a1f6fe94b58c2647d5cb6545b9c199e3e0d297862aeb3a1ffac8a9c9f03d96b4  ens-unicode-primary-poc.zip
```

The archive contains:

```
poc/unicode-direct-critical/run-clean.sh
poc/unicode-direct-critical/run-panoptes-clean.sh
poc/unicode-direct-critical/panoptes-anvil.override.yml
poc/deep-manager-normalization-route/hca-strict-primary.ts
poc/deep-manager-normalization-route/indexer-raw-name.ts
poc/deep-manager-normalization-route/panoptes-fast-contracts.json
audit-comp-ens/apps/manager/poc/deep-primary-dialog-reachability.test.tsx
audit-comp-ens/apps/manager/poc/unicode-cross-coin-recipient-live.test.tsx
audit-comp-ens/apps/manager/poc/deep-name-normalization.test.ts
scans/unicode-direct-critical-joined-final-verify.log
scans/unicode-panoptes-final-verify.log
```

Individual SHA-256 values:

```
17024bc658b3d5ce277a1fe1a12f6ce5e33849b5ae552daa95df9b0faebd7674  poc/unicode-direct-critical/run-clean.sh
00f95ae9bd2910e8766d5d98f1e913cf525ec3fd971425e6ab2132ffd2884735  poc/unicode-direct-critical/run-panoptes-clean.sh
baf3d8527dd7c98f641ff02bf21167bfd477abbdbd86f0f1230e84cbdd49ef35  poc/unicode-direct-critical/panoptes-anvil.override.yml
b4b4bfa0b5fae71cc5f0cf580f0e799e141e358450a62fa14b3f69e5ca57a5ef  poc/deep-manager-normalization-route/hca-strict-primary.ts
2e5c95f0f31e2c772b15f2044781978248564d420bd10e48bdf100c2efdb2f98  poc/deep-manager-normalization-route/indexer-raw-name.ts
ce8ee5282c6dd0789d6b9eb05365b7d9f218f42cf9c4b0513a36c483f8f9ed24  poc/deep-manager-normalization-route/panoptes-fast-contracts.json
8471e4808e41f8301826a5485e6dfa89510e8de9dd788015248fd08d71a3b2cd  audit-comp-ens/apps/manager/poc/deep-name-normalization.test.ts
54227db2b5e9f54c82769d90026533729be2338d8d0ac91126559f9ae651cf11  audit-comp-ens/apps/manager/poc/deep-primary-dialog-reachability.test.tsx
c8ffdda56d8b5f19c597ff0163a1ad851ed3255b87a0800e2f56028f7a38fd11  audit-comp-ens/apps/manager/poc/unicode-cross-coin-recipient-live.test.tsx
e2008d2da99761a059326f4ff27ba1582cda5711c1ba627aa7a3a70043cd9e98  scans/unicode-direct-critical-joined-final-verify.log
6cd16e0a0ba8eeb5f357f939d8eba922832fb94700c4f55ee7da01550eb5a4e5  scans/unicode-panoptes-final-verify.log
```

### Prerequisites

* Linux with Bash 4 or newer and GNU coreutils.
* Node.js v22.
* Corepack and pnpm v10.27.0.
* Docker with Docker Compose v2 and permission to start containers.
* Git, unzip, curl, jq, nice, and GNU timeout.
* A researcher-controlled Sepolia RPC URL with archive access to block `11556800`.
* Local ports `8545`, `3007`, and `5656` available.

The first run needs network access to install the frozen dependencies, pull the three pinned Docker images, and read the Sepolia state used by the local forks.

{% stepper %}
{% step %}

## Obtain the frozen competition repository

Create a clean working directory and clone the in-scope repository:

```bash
mkdir ens-unicode-primary-reproduction
cd ens-unicode-primary-reproduction

git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-ens.git \
  audit-comp-ens

cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
```

Verify the competition import:

```bash
test "$(git rev-parse HEAD)" = \
  "cda79acaad59711b943fc68207ebb3f1d0ff8596"

git show -s --format='%H%n%B' HEAD
```

The commit message identifies this as the frozen import of upstream:

```
63772fd872af472ced58b009499355f3430c2a86
```

{% endstep %}

{% step %}

## Install the frozen dependencies

Confirm Node and pnpm, then install from the lockfile:

```bash
node --version
corepack pnpm --version
corepack pnpm install --frozen-lockfile
```

Expected versions:

```
Node.js: v22.x
pnpm:    10.27.0
```

The install should report that the lockfile is up to date and complete without an error.
{% endstep %}

{% step %}

## Download and overlay the PoC files

Return to the directory containing the cloned `audit-comp-ens` folder, download the evidence bundle, verify its checksum, and extract it:

```bash
cd ..

curl -fL \
  'https://drive.usercontent.google.com/download?id=1yYwv7gdPkAoFTj6DbqHRtID7_2e6wYjM&export=download&confirm=t' \
  -o ens-unicode-primary-poc.zip

printf '%s  %s\n' \
  'a1f6fe94b58c2647d5cb6545b9c199e3e0d297862aeb3a1ffac8a9c9f03d96b4' \
  'ens-unicode-primary-poc.zip' | sha256sum -c -

unzip ens-unicode-primary-poc.zip
chmod +x \
  poc/unicode-direct-critical/run-clean.sh \
  poc/unicode-direct-critical/run-panoptes-clean.sh
```

The checksum command must print:

```
ens-unicode-primary-poc.zip: OK
```

The resulting layout must be:

```
ens-unicode-primary-reproduction/
├── audit-comp-ens/
│   └── apps/manager/poc/
│       ├── deep-name-normalization.test.ts
│       ├── deep-primary-dialog-reachability.test.tsx
│       └── unicode-cross-coin-recipient-live.test.tsx
├── poc/
│   ├── deep-manager-normalization-route/
│   │   ├── hca-strict-primary.ts
│   │   ├── indexer-raw-name.ts
│   │   └── panoptes-fast-contracts.json
│   └── unicode-direct-critical/
│       ├── panoptes-anvil.override.yml
│       ├── run-clean.sh
│       └── run-panoptes-clean.sh
└── scans/
    ├── unicode-direct-critical-joined-final-verify.log
    └── unicode-panoptes-final-verify.log
```

Validate the extracted runner before execution:

```bash
bash -n poc/unicode-direct-critical/run-clean.sh
bash -n poc/unicode-direct-critical/run-panoptes-clean.sh
```

{% endstep %}

{% step %}

## Supply a Sepolia fork endpoint

Supply a researcher-controlled endpoint through the environment without placing it in the source or command history:

```bash
read -rsp 'Researcher-controlled Sepolia RPC URL: ' SEPOLIA_FORK_URL
export SEPOLIA_FORK_URL
printf '\n'
```

Running the proof without this variable fails before starting Docker:

```
SEPOLIA_FORK_URL must be set to a researcher-controlled Sepolia RPC URL.
```

{% endstep %}

{% step %}

## Execute the joined proof

From `ens-unicode-primary-reproduction/`, run:

```bash
set -o pipefail

poc/unicode-direct-critical/run-clean.sh \
  2>&1 | tee scans/triager-unicode-primary-run.log

poc_status=${PIPESTATUS[0]}
test "$poc_status" -eq 0
```

The runner starts only the pinned Anvil and mockestrator services, waits for them to become healthy, and takes an outer snapshot.
{% endstep %}

{% step %}

## Observe distinct canonical-equivalent registrations

The fork harness registers two visually equivalent names through the real V2 registrar and dedicated resolvers:

```
NFC: \u00e9-hca-1811111111.eth
NFD: e\u0301-hca-1811111111.eth
```

The NFC name is registered to the known-key local attacker:

```
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```

The raw NFD name is registered to the local victim:

```
0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12
```

The harness asserts that the strings are unequal but `NFD.normalize("NFC")` equals NFC. It then reads both owners from the real local-fork registry. The proof aborts if the two registrations collapse to the same owner or identity.

The joined runner also executes `deep-name-normalization.test.ts` against the frozen package exports. Its three test cases prove that Manager's parser and validator accept the raw NFD form, ENSjs availability calls `isAvailable` with the raw decomposed label, and both the EOA and HCA registration builders encode that raw label unchanged. Representative output is:

```json
{
  "managerValidationError": null,
  "rawUtf8Hex": "0x65cc8178616d706c65",
  "normalizedUtf8Hex": "0xc3a978616d706c65",
  "labelHashesDiffer": true,
  "eoaEncodedLabel": "e\u0301xample",
  "hcaEncodedLabel": "e\u0301xample",
  "hcaEncodedPrimaryName": "e\u0301xample.eth"
}
```

{% endstep %}

{% step %}

## Verify the real Panoptes raw-name boundary

The Dashboard component case in Step 11 uses a controlled Panoptes response so it can isolate the production UI and signing path. This separate control proves that the response shape is not invented by the test.

Run the bounded Panoptes control from `ens-unicode-primary-reproduction/` while `SEPOLIA_FORK_URL` remains exported:

```bash
chmod +x poc/unicode-direct-critical/run-panoptes-clean.sh
set -o pipefail

poc/unicode-direct-critical/run-panoptes-clean.sh \
  2>&1 | tee scans/triager-unicode-panoptes-run.log

panoptes_status=${PIPESTATUS[0]}
unset SEPOLIA_FORK_URL
test "$panoptes_status" -eq 0
```

The control starts the pinned Anvil and this exact Panoptes image:

```
ghcr.io/mdtanrikulu/panoptes@sha256:a5fd2896ddb9e8117ca2d781e09042afea2b56b96a4a52935e79c9c994b24b2b
```

It uses an isolated volume and a test-only contract-address override matching the frozen deployment. It registers a raw NFD name, queries the real Panoptes GraphQL API by victim owner, and requires output equivalent to:

```json
{
  "rawName": "e\u0301-indexer-1788888888.eth",
  "nfcName": "\u00e9-indexer-1788888888.eth",
  "rawAndNfcDiffer": true,
  "rawDomainFound": true,
  "rawDomainName": "e\u0301-indexer-1788888888.eth",
  "rawDomainNormalizedName": "\u00e9-indexer-1788888888.eth",
  "rawDomainOwner": "0xc9eec1b174a646d7c282820afe94acfba6c00a12",
  "expectedVictim": "0xc9eec1b174a646d7c282820afe94acfba6c00a12"
}
```

The run must print `panoptesResult=PASS`, revert its outer snapshot, remove the two isolated Panoptes containers and volume, and stop the Anvil service it started. This proves the real indexer exposes the victim-owned NFD bytes as `Domain.name` while independently returning NFC in `normalizedName`.

The pinned control proves the current Panoptes field behavior against the frozen local deployment. It does not claim byte identity with an undisclosed production indexer build.
{% endstep %}

{% step %}

## Observe the post-confirmation signed-name substitution

The harness calls the production `setPrimaryNameWithHca` service with the raw victim-owned NFD name. It uses the real smart-account initialization, transaction manager, owner signer, funding planner, Warp transport, pinned local mockestrator, and deployed fork contracts.

Expected transaction lifecycle:

```
[TRANSACTION MANAGER] ... state: submitting
[WARP] intent submitted
[WARP] intent filled
[TRANSACTION MANAGER] ... state: pending
[TRANSACTION MANAGER] ... state: success
```

The result JSON must contain:

```json
{
  "dashboardSelectedRawName": "e\u0301-hca-1811111111.eth",
  "hcaSignedNormalizedName": "\u00e9-hca-1811111111.eth",
  "fillReceiptStatus": "success",
  "nfcNameOwner": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "rawNfdNameOwner": "0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12"
}
```

This proves that the input supplied to the production primary-name service is the victim-owned NFD registration, while the successful owner-signed HCA operation contains the attacker-owned NFC name.

The fresh verification run included in the evidence bundle produced this local fill transaction:

```
0x0d8e3da3c75cb15ce030c919ac52ffe2451ee292a0905c15625f577043ee1c19
```

The fill hash is local-fork evidence and can differ between reruns.

The `IndexedDB unavailable` warning is expected because this leg executes the browser transaction manager under Node. The transaction manager explicitly falls back to in-memory persistence; the signed request and fork transaction still execute and reach `success`.
{% endstep %}

{% step %}

## Observe the ETH-only strict match and attacker Optimism recipient

The attacker-owned NFC resolver is configured as follows:

```
ETH coin type 60:
0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12
(victim)

Optimism coin type 2147483658:
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
(known-key local attacker)
```

The output must show all of the following simultaneously:

```json
{
  "strictVictimPrimaryAfterAttackerRecipient": "\u00e9-hca-1811111111.eth",
  "strictVictimPrimaryMatchAfterAttackerRecipient": true,
  "managerAddressProfileNameAfterAttackerRecipient": "\u00e9-hca-1811111111.eth",
  "verifiedVictimEthRecipientAfterAttackerMutation": "0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12",
  "attackerControlledOptimismRecipientRenderedByManager": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "optimismCoinType": 2147483658
}
```

This is the exploitable split: Manager's production viem call returns the NFC name, and the independent ENSjs strict control reports `match=true`, because the guard checks equality of ETH(60). That check establishes neither ownership of the NFC name nor its Optimism record, which the attacker independently controls and Manager later renders.

The BTC value in the output is only an attacker-selected secondary cross-coin test value. No private-key control over that public example BTC address is claimed. The direct known-key recipient proof and selected impact use the Optimism address.
{% endstep %}

{% step %}

## Observe the strict-reverse negative control

The harness temporarily changes the NFC ETH record from the victim to the attacker. The expected output is:

```json
{
  "strictVictimPrimaryAfterRepoint": null,
  "strictVictimPrimaryErrorAfterRepoint": "ReverseAddressMismatch",
  "managerAddressProfileNameAfterAttackerRepoint": null
}
```

This proves the ETH strict-reverse guard is genuinely enforced. The exploit does not bypass that guard; it preserves the verified ETH value while changing a different chain's recipient.

The harness restores ETH(60) to the victim before handing the same fork state to the live Manager component test.
{% endstep %}

{% step %}

## Observe the real Dashboard selection and final calldata

`deep-primary-dialog-reachability.test.tsx` mounts the production `ChoosePrimaryNameDialog` and production primary-name hook/service.

It supplies the Panoptes schema shape containing:

```json
{
  "name": "cafe\u0301-dialog.eth",
  "normalizedName": "caf\u00e9-dialog.eth"
}
```

The output must show:

```json
{
  "renderedAndSelected": "cafe\u0301-dialog.eth",
  "profileRecordQueryInput": "cafe\u0301-dialog.eth",
  "finalDecodedFunction": "setNameWithHCA",
  "finalDecodedArgs": [
    "0x3333333333333333333333333333333333333333",
    "caf\u00e9-dialog.eth"
  ],
  "selectedDiffersFromSigned": true
}
```

The Dashboard component test mocks the wallet/account context, Panoptes response, reverse/profile/resolver query results, record-write submission, resolver-setup boundary, and transaction/funding boundary. It uses the real production dialog rendering and selection logic, confirmation ordering, primary-name hook, `setPrimaryNameWithHca` normalization, and ABI construction. The separate fork leg proves resolver discovery, the ETH-record update, and successful HCA execution against live fork state.
{% endstep %}

{% step %}

## Observe the production Manager rendering and clipboard sink

`unicode-cross-coin-recipient-live.test.tsx` consumes the state left by the fork harness. It calls the real `getProfileRecords` service, real ENSjs/RPC reads, production record transformation, `ProfileAddressesSection`, and `CopyableButton`.

The output must contain:

```json
{
  "rawNfdOwner": "0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12",
  "nfcOwner": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "strictVictimPrimaryName": "\u00e9-hca-1811111111.eth",
  "strictVictimPrimaryMatch": true,
  "managerMainEthRecipient": "0xc9eec1B174A646D7c282820aFE94AcfBA6c00A12",
  "managerOptimismCoinType": 2147483658,
  "managerOptimismRecipient": "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266",
  "clipboardWrites": [
    "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
    "0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266"
  ],
  "appBuiltTransferClaimed": false
}
```

The only mocked record-data response in this live recipient test is an empty Panoptes cache response. Manager statically probes its complete `addressRecords` list, so that empty response does not supply or suppress the Optimism record. The mocked `safeGetClient` boundary injects the real local-fork client, and the happy-dom test environment's `navigator.clipboard.writeText` is replaced with an observer. Neither supplies an ENS or profile value. ENSjs, RPC, registry ownership, strict reverse resolution, ETH/Optimism record values, profile transformation, address-card rendering, and the production `CopyableButton` handler all use the live fork state.

The clipboard observer lets the test assert the exact bytes passed by the production click handler. The production `CopyableButton` itself is not mocked.
{% endstep %}

{% step %}

## Observe the final result and cleanup

The run must finish with:

```
Test Files  3 passed (3)
Tests       5 passed (5)
joinedResult=PASS
{"jsonrpc":"2.0","id":1,"result":true}
Container infra-mockestrator-1 Stopped
Container infra-anvil-1 Stopped
```

`joinedResult=PASS` is printed only after:

1. The real local-fork HCA transaction reaches success.
2. NFD and NFC ownership remain split between victim and attacker.
3. Manager's production viem reverse call returns the attacker-owned NFC name.
4. The independent ENSjs strict control remains `match=true`.
5. ETH still resolves to the victim.
6. Optimism resolves to the known-key attacker.
7. Manager's parser, availability call, and EOA/HCA registration builders preserve the raw NFD bytes.
8. The production Dashboard path converts the raw selected value into the NFC calldata value.
9. The production profile service renders the attacker Optimism value.
10. The production copy handler receives the exact full attacker address.

The JSON-RPC `result:true` is the outer `evm_revert` result.

As an additional cleanup check:

```bash
docker compose \
  -f audit-comp-ens/e2e/infra/docker-compose.yml \
  ps --status running --services anvil mockestrator
```

It should print no services.
{% endstep %}
{% endstepper %}

### Proof boundaries

* This is a joined proof, not a claim that one Playwright browser session drove registration, indexing, HCA execution, and profile rendering. The complementary legs prove each production boundary, while the same live fork state is retained from the successful HCA operation through ENS record reads and the final Manager render/copy assertion.
* The proof demonstrates incorrect recipient rendering and exact clipboard substitution. It does not claim that Manager contains an in-app Optimism payment flow or that a public-chain payment was already made.
* A sender must use the displayed/copied Optimism recipient for funds to be sent to the attacker. This is the exact user action described by the selected in-scope impact.
* The victim must own/register the raw NFD name, select it as primary, and approve the normal owner-signed operation.
* The attacker retains the NFC name and must keep its ETH record pointed at the victim for strict reverse verification to remain valid.
* No victim-name takeover, HCA/session takeover, malicious RPC response, backend compromise, or public-chain exploitation is claimed.
* No smart-contract vulnerability is claimed. The contracts are used only to demonstrate execution of the client-generated primary-name argument on a private fork.
