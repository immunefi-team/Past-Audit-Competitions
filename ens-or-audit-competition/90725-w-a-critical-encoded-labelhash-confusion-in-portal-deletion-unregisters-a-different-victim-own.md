# 90725 w a critical encoded labelhash confusion in portal deletion unregisters a different victim owned subname and enables registrar takeover

**Submitted on Aug 26th 2026 at 18:19:08 UTC by @sl1de for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90725
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address
  * Subdomain takeover with already-connected wallet interaction

## Description

## Brief/Intro

Portal fetches the authoritative `labelhash` for every ENSv2 subname from Panoptes, but drops that identity before rendering the deletion table. It later reconstructs a label from the displayed name and sends the string through the frozen ENSjs and viem encoded-labelhash parser. A registrar-only attacker can register the literal label `[H]`, where `H` is the public labelhash of a valuable victim-owned subname. The literal attacker token is keyed by `B = keccak256(bytes("[H]"))`, but Portal interprets its display string as the precomputed hash `H`. The victim sees and confirms deletion of the attacker-owned `[H]` row, while their connected wallet signs `unregister(H)` for the different victim-owned subname. The attacker row remains registered, the victim label becomes available, and the attacker uses only its existing `ROLE_REGISTRAR` permission to take over the valuable subname. I reproduced the complete path with authentic Panoptes, the production Portal route and modal, the production transaction manager, a connected victim wallet, and a private local Sepolia fork.

## Vulnerability Details

### Attack scenario

There are two actors:

* The victim administers an ENSv2 parent registry, owns an existing valuable child, and has the legitimate `ROLE_UNREGISTER` authority required for Portal to show its delete controls.
* The attacker is a delegated registrar for that parent. At the parent root and victim-label scopes, the attacker has only `ROLE_REGISTRAR` and no `ROLE_UNREGISTER`, `ROLE_CAN_TRANSFER_ADMIN`, `ROLE_SET_RESOLVER`, admin role, or control of the victim wallet. Any roles attached to the attacker's own newly registered token do not authorize actions against the victim token.

The proof uses:

```
Parent:
bracket-parent-1889999999.eth

Victim-owned child:
valuable.bracket-parent-1889999999.eth

H = labelhash("valuable"):
0x75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f

Attacker-owned literal child:
[75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f].bracket-parent-1889999999.eth

B = keccak256(bytes("[75aaf3...cb81f]")):
0x0209ff1d12ccaa5b5979292eaed2e5f1227a9a05e506cd2f96ac40cbcf81bd93
```

`H` and `B` are different on-chain resources. Both registrations coexist before exploitation, and each has a different owner.

{% stepper %}
{% step %}

## Register the victim's valuable child

The victim's valuable child is registered under token key `H`.
{% endstep %}

{% step %}

## Register the attacker-controlled literal label

The attacker uses only delegated `ROLE_REGISTRAR` to register the literal label `[H]` under token key `B`.
{% endstep %}

{% step %}

## Index both labels

Panoptes correctly indexes both rows. For the attacker row it returns the literal display `name`, literal `labelName`, and authoritative `labelhash = B`.
{% endstep %}

{% step %}

## Drop canonical identity in Portal

Portal fetches all three fields, but converts the result into a `SubnameRow` containing only display `name`, owner, and `canDelete`.
{% endstep %}

{% step %}

## Select the attacker-owned row

The victim opens the normal Subnames page and chooses Delete for the attacker-owned literal row. The inline confirmation and transaction modal continue naming that exact row.
{% endstep %}

{% step %}

## Reconstruct the displayed label

Portal strips the parent suffix from the displayed name and reconstructs the string `[H]`.
{% endstep %}

{% step %}

## Parse the label as an encoded labelhash

Portal passes `[H]` to `deleteSubnameWriteParameters`. The frozen encoded-labelhash behavior interprets bracket notation as the precomputed value `H` rather than hashing the literal bracket bytes to `B`.
{% endstep %}

{% step %}

## Sign the wrong unregister operation

The connected victim wallet signs `unregister(H)`.
{% endstep %}

{% step %}

## Release the victim label

The valuable victim child becomes available. The displayed attacker child under `B` remains registered and still belongs to the attacker.
{% endstep %}

{% step %}

## Take over the valuable label

The attacker immediately uses its existing registrar-only permission to register `valuable` to itself.
{% endstep %}
{% endstepper %}

The identity split is:

```
Displayed and selected row identity = B = keccak256(bytes("[H]"))
Signed unregister argument          = H = labelhash("valuable")

B != H
```

This is not a hash collision. It is a display-to-calldata type confusion between a literal string label and encoded-labelhash sentinel notation.

### Root cause 1: Panoptes supplies the correct canonical identity

Portal's own subname type includes the display fields and canonical token identity:

```ts
type Subname = {
  name: string | null
  labelName: string | null
  labelhash: Hex
  owner: Address
}
```

Its ENSv2 query requests all three relevant values:

```graphql
subdomains {
  name
  labelName
  labelhash
  owner { id }
}
```

The final private-fork query returned these two authentic rows simultaneously:

```json
{
  "name": "[75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f].bracket-parent-1889999999.eth",
  "labelName": "[75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f]",
  "labelhash": "0x0209ff1d12ccaa5b5979292eaed2e5f1227a9a05e506cd2f96ac40cbcf81bd93",
  "owner": {
    "id": "0x3c44cdddb6a900fa2b585dd299e03d12fa4293bc"
  }
}
```

```json
{
  "name": "valuable.bracket-parent-1889999999.eth",
  "labelName": "valuable",
  "labelhash": "0x75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f",
  "owner": {
    "id": "0x70997970c51812dc3a010c7d01b50e0d17dc79c8"
  }
}
```

The indexer is not confused. It provides the exact distinction needed to delete the selected resource safely.

### Root cause 2: the route drops the authoritative labelhash

`SubnameRow` contains no `labelhash` or `labelName` field. The route converts each indexed row as follows:

```ts
const subnameRows: SubnameRow[] = (subnames || [])
  .filter((subname) => !optimisticallyDeleted.has(subname.name || ''))
  .map((subname) => ({
    name: subname.name || '',
    owner: subname.owner,
    canDelete: canDeleteSubname,
  }))
```

At this boundary, the security identity changes from an indexed on-chain token key to a display string.

### Root cause 3: the victim's decision is bound to the display string

The table renders `row.original.name`, labels the action `Delete ${row.original.name}`, and asks:

```
Remove [75aaf3...cb81f].bracket-parent-1889999999.eth?
```

The transaction modal then names the operation:

```
Delete [75aaf3...cb81f].bracket-parent-1889999999.eth
```

The screenshot in the evidence bundle captures this production modal. No visible field identifies the different token key `H` that will be signed.

### Root cause 4: Portal re-parses display text into a different resource

The route reconstructs a label from the displayed full name:

```ts
const getLabel = useCallback(
  (subname: string) => {
    const suffix = `.${name}`
    if (subname.endsWith(suffix)) {
      return subname.slice(0, -suffix.length)
    }
    return subname.split('.')[0]
  },
  [name],
)
```

It then supplies that string to both the preview intent and the submitted mutation:

```ts
deleteSubnameAsync({
  subname: subname.name,
  label: getLabel(subname.name),
  id,
})
```

```ts
prepareDeleteSubnameTransaction({
  registryAddress: subregistryAddress,
  label: getLabel(subname.name),
  walletClient,
  chainId,
})
```

The builder forwards that re-derived string to the frozen ENSjs export:

```ts
const writeParams = deleteSubnameWriteParameters(walletClient, {
  registryAddress,
  label,
})
```

For a normal label, this path produces `keccak256(bytes(label))`, and the ordinary-label negative control passes. For the reserved `[64 lowercase hex]` spelling, it produces the embedded hash. The application already had the canonical value `B` but discarded it before this parser boundary.

### Root cause 5: the displayed description and final intent use different identity representations

The modal title remains derived from `subname.name`:

```ts
transactionName: `Delete ${subname.name}`
```

The transaction data is independently derived from `getLabel(subname.name)`. The production transaction manager correctly signs and broadcasts the intent it receives. The transaction manager is not the vulnerable component. Portal gives it calldata for a different resource than the one named in the victim-facing description.

### Full vulnerable call path

```
Panoptes subdomain row
  name = "[H].parent.eth"
  labelName = "[H]"
  labelhash = B
        |
        v
getSubnames()
        |
        v
subnameRows mapping drops labelName and labelhash
        |
        v
SubnamesTable displays and confirms "[H].parent.eth"
        |
        v
getLabel("[H].parent.eth") returns "[H]"
        |
        v
prepareDeleteSubnameTransaction(label = "[H]")
        |
        v
deleteSubnameWriteParameters parses encoded-labelhash notation
        |
        v
transaction manager receives unregister(H)
        |
        v
victim wallet signs and broadcasts unregister(H)
        |
        v
victim valuable label is available, attacker literal B remains registered
        |
        v
registrar-only attacker registers valuable to itself
```

### The authorization checks work but do not prevent the exploit

Portal correctly checks whether the connected victim has `ROLE_UNREGISTER` before exposing deletion. The proof does not bypass this guard. The victim is legitimately authorized to delete subnames, and Portal misdirects that authority to a different token.

The attacker was verified to have exactly these parent-root permissions:

```
ROLE_REGISTRAR:           true
ROLE_UNREGISTER:          false
ROLE_CAN_TRANSFER_ADMIN:  false
ROLE_SET_RESOLVER:        false
```

The direct attacker attempt to unregister `H` was rejected. This matters because ENS documents root `ROLE_REGISTRAR` as a non-dangerous role that can register available names but does not endanger existing registered names. Root or victim-scoped `ROLE_UNREGISTER` is the separate dangerous authority. Portal turns the victim's valid delete interaction into the missing destructive step.

### Verified end-to-end result

The final clean run used the competition import `cda79acaad59711b943fc68207ebb3f1d0ff8596`, which imports upstream commit `63772fd872af472ced58b009499355f3430c2a86`.

| Item                                    | Verified result                                                              |
| --------------------------------------- | ---------------------------------------------------------------------------- |
| Node                                    | v22.23.2                                                                     |
| pnpm                                    | 10.27.0                                                                      |
| Environment                             | Private disposable Sepolia fork, authentic local Panoptes, production Portal |
| Parent registry                         | `0xb1B93814E54b5868DD3f3018b372Ecf00b9B9942`                                 |
| Victim                                  | `0x70997970C51812dc3A010C7d01b50e0d17dc79C8`                                 |
| Attacker                                | `0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC`                                 |
| Victim token key                        | `0x75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f`         |
| Displayed attacker token key            | `0x0209ff1d12ccaa5b5979292eaed2e5f1227a9a05e506cd2f96ac40cbcf81bd93`         |
| Signed function                         | `unregister`                                                                 |
| Signed calldata                         | `0xa02b161e75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f` |
| Browser delete transaction              | `0xc446aa23fee9828ffa00f657db2e5e5470e1e100378221b7b50efe10fc9f0fa5`         |
| Victim token after approval             | Status `0`, available                                                        |
| Displayed attacker token after approval | Status `2`, still registered to attacker                                     |
| Attacker takeover transaction           | `0x047d0adb7340a9502b50d26b822b9235f35c05131695755a901f16aa9c08104b`         |
| Valuable owner after takeover           | Attacker                                                                     |
| Browser result                          | `1 passed (18.1s)`                                                           |
| Runner markers                          | `browserResult=PASS`, `joinedResult=PASS`                                    |
| Cleanup marker                          | `servicesOwnedByRunnerStopped=true`                                          |

The transactions are private-fork evidence and will differ on reruns.

### Negative controls

* Both token identities were registered simultaneously before deletion.
* Panoptes returned the correct distinct `labelhash` for each row.
* The attacker had root registrar authority but lacked root or victim-scoped unregister, transfer-admin, and resolver authority.
* A direct attacker unregister simulation was rejected.
* An ordinary-label control produced matching literal and builder hashes.
* The production modal named the attacker literal row.
* The mined browser transaction decoded to `unregister(H)`, not `unregister(B)`.
* After the victim transaction, only the valuable token became available.
* The attacker then successfully registered the valuable label.
* Portal's own registration entry points correctly reject bracket-form labels. The attacker uses an authorized direct registry call, which those client-side entry points cannot constrain.

### Difference from known issues

This report does not submit `EXP-INPUT-005` as-is. That known Medium issue concerns validators accepting homoglyphs, bidi and control characters, and leading or trailing dots. This proof uses only lowercase ASCII hex and bracket syntax. Portal's current registration validation correctly rejects the attacker's literal label at Portal registration entry points.

The submitted root cause and consequence are different:

* Two different on-chain token identities coexist under different owners.
* Panoptes supplies the correct identity for both.
* Portal discards the selected row's canonical identity.
* Portal re-parses its display representation into another token ID.
* A connected victim wallet mines a state-changing deletion against that other token.
* A registrar-only attacker takes over the freed victim subname.

Even if bracket-label ambiguity is treated as touching a known display concern, the competition rules expressly keep new consequences that materially change severity in scope. This is a demonstrated Critical connected-wallet transaction-argument substitution and subdomain takeover, not display confusion alone.

This is also not:

* `QA-01`: no role-combination error occurs. The victim's unregister check and the attacker's registrar role are both evaluated correctly.
* `WEB-1249`: no child registry is replaced and no nested registry is orphaned.
* A viem or ENSjs vulnerability: the encoded-labelhash parser behaves as designed. Portal misuses display text where it already possesses a canonical token identity.
* A contract vulnerability: the Permissioned Registry correctly stores `H` and `B` as different labels and enforces separate registrar and unregister roles.
* The previously submitted Manager Unicode report: this proof has no Unicode normalization, primary-name flow, reverse resolution, or cross-chain address record. Fixing either root cause does not fix the other.

### Preconditions and honest boundaries

* The attacker must hold delegated `ROLE_REGISTRAR` on the victim's parent registry.
* The victim must hold `ROLE_UNREGISTER` and use Portal's normal delete workflow.
* The victim clicks Delete, confirms, and approves the connected-wallet transaction.
* The attacker cannot perform the deletion alone.
* The attacker's bracket row is visibly unusual. The exploit does not depend on the row looking like a normal label. It depends on Portal promising to delete that exact selected row while signing a different token ID.
* A parent administrator may later recover by unregistering the attacker registration and reissuing the label. The proven impact is attacker control until recovery, not irreversible destruction.
* No public-chain transaction, malicious RPC response, forged indexer response, backend compromise, wallet compromise, leaked key, session theft, or dangerous registry authority such as `ROLE_UNREGISTER` or an admin role is required.

## Impact Details

This maps directly to both selected Critical Websites & Applications impacts.

First, it causes subdomain takeover with already-connected wallet interaction. The victim approves deletion of the attacker-owned literal bracket row, but Portal unregisters the separate victim-owned valuable subname. The attacker immediately re-registers the valuable label and becomes its owner. The proof verifies the attacker as final owner while the original displayed attacker row remains untouched.

Second, it causes a malicious interaction with an already-connected wallet. The UI and transaction modal name resource `B`, but the wallet signs and broadcasts `unregister(H)`. The security-sensitive `uint256` resource argument differs from the selected and confirmed on-chain row. This is a concrete what-you-see-is-not-what-you-sign transaction-argument substitution.

From the attacker's perspective, a permission intentionally limited to registering available names is enough to seed the ambiguous row. The attacker waits for the namespace administrator to remove that row through Portal, then claims the valuable label that Portal deleted.

From the victim's perspective, they select the attacker-owned `[H]` registration and approve its removal. Their valid unregister authority is silently redirected to their different `valuable` registration.

The affected value is the ENS subdomain and any identity, reputation, routing, or service association attached to that semantic name. I do not additionally claim direct NFT theft because unregister and re-registration use different token generations, and I do not claim an unintended recipient because `unregister()` has no recipient argument.

## Recommended Remediation

{% stepper %}
{% step %}

## Preserve canonical identity

Preserve the indexed `labelhash` in `SubnameRow` and every deletion state object.
{% endstep %}

{% step %}

## Encode from the selected token identity

Encode `unregister()` directly from the canonical selected token identity. Never reconstruct a security-sensitive token ID from display text.
{% endstep %}

{% step %}

## Bind UI and intent to one immutable object

Bind the displayed row, inline confirmation, transaction modal, gas-estimated intent, and submitted intent to one immutable object containing the registry address and authoritative labelhash.
{% endstep %}

{% step %}

## Verify any re-derived hash

If any code path re-derives a hash, require it to equal the indexed row's labelhash before enabling confirmation.
{% endstep %}

{% step %}

## Use separate internal types

Treat encoded-labelhash display notation and literal bracket labels as different typed values internally, even if their rendered strings are identical.
{% endstep %}

{% step %}

## Add regression coverage

Add regression coverage for a literal `[H]` label and ordinary label `x` where `labelhash(x) = H`, including single deletion, bulk deletion, and genuine unknown-label rows.
{% endstep %}
{% endstepper %}

## References

* [Panoptes query returns name, labelName, and labelhash](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/profile/hooks/useSubnames.ts#L48-L84)
* [SubnameRow has no labelName or labelhash](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/names/components/SubnamesTable.tsx#L36-L41)
* [Route drops both indexed identity fields when creating deletion rows](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/subnames.tsx#L370-L390)
* [Table displays row.original.name and labels the delete button with it](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/names/components/SubnamesTable.tsx#L93-L140)
* [Confirmation passes the selected display row back to the route](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/names/components/SubnamesTable.tsx#L158-L175)
* [Desktop confirmation says Remove row.original.name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/names/components/SubnamesTable.tsx#L413-L439)
* [Route extracts a new label from the displayed full name and supplies it to the mutation](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/subnames.tsx#L173-L231)
* [Preview re-derives that label while its title continues naming the displayed row](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/subnames.tsx#L257-L300)
* [Transaction builder sends the re-derived string to ENSjs and encodes its result](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/registry/utils/delete-subname.helpers.ts#L24-L54)
* [Hook forwards that label into the production deletion helper](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/registry/hooks/useDeleteSubname.ts#L67-L90)
* [Deletion helper starts the prepared intent through transaction manager](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/registry/helpers/deleteSubname.ts#L36-L69)
* [Bracket-label registration rejection](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/token/isNormalized.ts#L59-L82)
* [Create-subname invokes the guard and refuses transaction start](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/create-subname.tsx#L145-L171)
* [ENSv2 Permissioned Registry role and lifecycle semantics](https://docs.ens.domains/ensv2/permissioned-registry/)

## Proof of Concept

### Safety and environment

This proof runs entirely on a private disposable Anvil fork using standard throwaway development accounts. It does not send transactions to public Sepolia, Ethereum mainnet, or any other public chain. It does not use production names, funds, wallets, sessions, credentials, or services.

The required Sepolia endpoint is used only by local Anvil to read the state needed to initialize the fork. All registration, deletion, and takeover transactions are mined inside the local container. The runner never submits a transaction to the supplied endpoint.

The runner uses isolated host ports:

```
Anvil:        24545
Panoptes API: 24655
Portal:       24301
```

It refuses to start if those ports or its dedicated Compose project already exist. It starts only its pinned Anvil and Panoptes images, stores Panoptes data in its own temporary Compose volume, and removes only resources labeled as belonging to its own project. It prints `servicesOwnedByRunnerStopped=true` only after verifying that its containers, network, volume, and ports are gone.

### PoC evidence bundle

Download the complete runnable overlay and final evidence here:

[Download ens-portal-bracket-label-deletion-poc.zip from Google Drive](https://drive.google.com/file/d/1R9AVTrgOIZGoznfvzt68rhGcX1iAp8x1/view?usp=sharing)

Save it with this exact filename:

```
ens-portal-bracket-label-deletion-poc.zip
```

ZIP SHA-256:

```
50c36d0deb5753a70ed34fdd96a29676cfe19cde8fd702d10cd4b1e5db7afcbf  ens-portal-bracket-label-deletion-poc.zip
```

The archive contains the portable runner, pinned Compose definition, PoC-only Portal proxy configuration, on-chain seed harness, production Playwright test, fresh Panoptes response, final run log, and production UI screenshot. An internal `POC-SHA256SUMS.txt` authenticates every included source and evidence file except the checksum file itself.

### Prerequisites

* Linux with Bash.
* Node.js v22. The final verification used v22.23.2.
* Corepack and pnpm v10.27.0.
* Docker with Docker Compose v2 and permission to start containers.
* Git, curl, jq, `ss`, `setsid`, unzip, and sha256sum.
* Chromium at `/usr/bin/chromium`, or another executable supplied through `CHROMIUM_PATH`.
* A researcher-controlled Sepolia RPC URL with ordinary read and fork support.
* Local ports 24545, 24655, and 24301 available.
* Network access on the first run to clone the repository, install the frozen dependencies, pull the two pinned container images, and initialize the local fork.

{% stepper %}
{% step %}

## Obtain the frozen competition repository

Create a clean directory and clone the in-scope repository:

```bash
mkdir ens-portal-bracket-delete-reproduction
cd ens-portal-bracket-delete-reproduction

git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-ens.git \
  audit-comp-ens

cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
```

Verify the frozen competition import:

```bash
test "$(git rev-parse HEAD)" = \
  "cda79acaad59711b943fc68207ebb3f1d0ff8596"

git show -s --format='%H%n%B' HEAD
```

The commit message identifies upstream import:

```
63772fd872af472ced58b009499355f3430c2a86
```

{% endstep %}

{% step %}

## Install the frozen dependencies

```bash
node --version
corepack pnpm --version
corepack pnpm install --frozen-lockfile
```

Expected major versions:

```
Node.js: v22.x
pnpm:    10.x
```

The final evidence run used Node v22.23.2 and pnpm 10.27.0.
{% endstep %}

{% step %}

## Overlay and verify the PoC bundle

Download the Google Drive archive into the `audit-comp-ens` directory, then verify the outer checksum before extraction:

```bash
printf '%s  %s\n' \
  '50c36d0deb5753a70ed34fdd96a29676cfe19cde8fd702d10cd4b1e5db7afcbf' \
  'ens-portal-bracket-label-deletion-poc.zip' | sha256sum -c -
```

Expected output:

```
ens-portal-bracket-label-deletion-poc.zip: OK
```

Extract the overlay into the frozen repository root:

```bash
unzip ens-portal-bracket-label-deletion-poc.zip
chmod +x \
  poc/portal-bracket-delete/run-clean.sh \
  poc/portal-bracket-delete/test-runner-preflight.sh
```

Verify every internal file:

```bash
sha256sum -c POC-SHA256SUMS.txt
```

Every listed file must print `OK`.

The resulting relevant layout is:

```
audit-comp-ens/
|-- POC-README.md
|-- POC-SHA256SUMS.txt
|-- apps/portal/poc/
|   `-- portal-bracket-delete.vite.config.ts
|-- e2e/poc/
|   |-- portal-bracket-delete-onchain.ts
|   `-- portal-bracket-delete-playwright.config.ts
|-- e2e/projects/portal/tests/
|   `-- portal-bracket-delete.poc.spec.ts
|-- poc/portal-bracket-delete/
|   |-- docker-compose.yml
|   |-- panoptes.json
|   |-- run-clean.sh
|   `-- test-runner-preflight.sh
`-- scans/
    |-- portal-bracket-delete-api.log
    |-- portal-bracket-delete-clean.log
    |-- portal-bracket-delete-panoptes-runtime.json
    |-- portal-bracket-delete-panoptes.json
    |-- portal-bracket-delete-ui.png
    `-- portal-bracket-delete-vite.log
```

{% endstep %}

{% step %}

## Run the fail-closed preflight

```bash
poc/portal-bracket-delete/test-runner-preflight.sh
```

Expected result:

```
runnerPreflightTests=PASS
```

This checks shell syntax, verifies the PoC-specific Portal RPC proxy is present, verifies Vite will honor the isolated port, and confirms that the runner refuses to start without an explicit fork endpoint.
{% endstep %}

{% step %}

## Supply a read-only Sepolia fork endpoint

Supply a researcher-controlled endpoint without placing it in source files or the report:

```bash
read -rsp 'Researcher-controlled Sepolia RPC URL: ' \
  BRACKET_DELETE_SEPOLIA_FORK_URL
export BRACKET_DELETE_SEPOLIA_FORK_URL
printf '\n'
```

The runner exits before starting Docker if this variable is missing.

If Chromium is not at `/usr/bin/chromium`, also set:

```bash
export CHROMIUM_PATH='/absolute/path/to/chromium'
```

{% endstep %}

{% step %}

## Execute the joined proof

From the `audit-comp-ens` repository root, run:

```bash
poc/portal-bracket-delete/run-clean.sh
```

The runner writes the complete terminal evidence to:

```
scans/portal-bracket-delete-clean.log
```

It performs the following sequence:

1. Confirms the exact frozen commit, Node v22, pnpm v10, required commands, Chromium, free ports, and absence of prior PoC resources.
2. Starts pinned Anvil on port 24545 as a private Sepolia fork.
3. Starts pinned Panoptes indexer and API containers on the runner's private network.
4. Deploys the local parent registry and creates the victim and attacker labels.
5. Verifies the attacker's parent-root role is registrar only.
6. Waits for authentic Panoptes to return both distinct rows.
7. Starts the production Portal app on port 24301, with only its development RPC and indexer proxies pointed at the isolated local services.
8. Connects the throwaway victim wallet through the normal E2E wallet fixture.
9. Opens the production Subnames route, clicks Delete for the attacker's literal bracket row, confirms, and opens the connected wallet.
10. Authorizes the transaction and decodes the mined browser-submitted calldata.
11. Verifies the victim token became available while the selected attacker token remained registered.
12. Re-registers the valuable label with the registrar-only attacker and verifies final ownership.
13. Stops the Portal process group and removes only the PoC Compose resources.
    {% endstep %}

{% step %}

## Observe two distinct indexed registrations

The seed output must show:

```json
{
  "valuableName": "valuable.bracket-parent-1889999999.eth",
  "valuableLabelhash": "0x75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f",
  "literalBracketHash": "0x0209ff1d12ccaa5b5979292eaed2e5f1227a9a05e506cd2f96ac40cbcf81bd93",
  "valuableOwner": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "literalSpamOwner": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC",
  "attackerRoles": {
    "registrar": true,
    "unregister": false,
    "transferAdmin": false,
    "setResolver": false
  }
}
```

The role booleans are parent-root checks. Any roles on the attacker's own newly issued token do not authorize the victim resource.

After Panoptes catches up, the runner must print:

```
authenticPanoptesRows=PASS
```

The captured `scans/portal-bracket-delete-panoptes.json` must contain exactly two children under the parent:

```
Attacker literal display name -> labelhash 0x0209ff...bd93
Victim valuable display name  -> labelhash 0x75aaf3...cb81f
```

{% endstep %}

{% step %}

## Observe what the victim is shown

The production browser opens:

```
/bracket-parent-1889999999.eth/subnames
```

It selects the row:

```
[75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f].bracket-parent-1889999999.eth
```

The inline confirmation says:

```
Remove [75aaf3...cb81f].bracket-parent-1889999999.eth?
```

The production transaction modal says:

```
Delete [75aaf3...cb81f].bracket-parent-1889999999.eth
```

The runner captures this state in:

```
scans/portal-bracket-delete-ui.png
```

{% endstep %}

{% step %}

## Observe the signed resource divergence

The same browser workflow mines and decodes:

```
Function:
unregister(uint256)

Signed calldata:
0xa02b161e75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f

Signed token ID:
0x75aaf3c2cddba877fcca9e7d249445bb32514b4f3ce90b9017a5ae065e1cb81f

Selected row's authoritative token ID:
0x0209ff1d12ccaa5b5979292eaed2e5f1227a9a05e506cd2f96ac40cbcf81bd93
```

The two token IDs are unequal.

The final bundled run produced this local deletion transaction:

```
0xc446aa23fee9828ffa00f657db2e5e5470e1e100378221b7b50efe10fc9f0fa5
```

After inclusion, the assertions are:

```
valuableStatusAfterVictimApproval = 0
literalSpamStatusAfterVictimApproval = 2
```

Therefore the selected attacker row remains registered while the different victim row is deleted.
{% endstep %}

{% step %}

## Observe the registrar-only takeover

The same test uses the existing attacker registrar authority to register `valuable` after Portal frees it. The final bundled run produced:

```
0x047d0adb7340a9502b50d26b822b9235f35c05131695755a901f16aa9c08104b
```

The final assertions are:

```
valuableOwnerAfterTakeover = 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
literalSpamOwnerAfterTakeover = 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
```

The transaction hashes are private-fork evidence and can differ on reruns. The resource identities and state transition are deterministic.
{% endstep %}

{% step %}

## Observe final pass and cleanup markers

The proof must finish with:

```
1 passed
browserResult=PASS
joinedResult=PASS
servicesOwnedByRunnerStopped=true
```

The bundled final run reported:

```
1 passed (18.1s)
```

The success markers are printed only after the authentic Panoptes assertions, UI workflow, signed calldata decode, post-delete state checks, and attacker takeover all pass. The cleanup marker is printed only after no PoC container, network, volume, or isolated listening port remains.
{% endstep %}
{% endstepper %}

### Proof boundaries

* The proof demonstrates a subdomain takeover after one normal connected-wallet deletion workflow.
* The attacker starts with a delegated registrar role but has no root or victim-scoped unregister authority.
* The victim clicks Delete, confirms, and approves the wallet request.
* The proof does not claim interaction-free takeover, wallet compromise, permanent destruction, direct theft of the original NFT generation, or an unintended transaction recipient.
* The proof does not modify any public chain or exercise a third-party credential.
* Contracts and dependency behavior are used only as the correct execution environment. The submitted vulnerability is Portal dropping canonical identity and re-parsing display text into security-sensitive calldata.
