# 90794 w a critical explorer confirms an uppercase ensv2 transfer but signs the distinct lowercase token id causing direct nft theft

**Submitted on Aug 27th 2026 at 10:14:57 UTC by @sl1de for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90794
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

Portal can display and confirm the transfer of an attacker-created uppercase ENS gift while its destructive transfer pipeline silently derives and signs the token ID of the victim's distinct lowercase ENS name. An attacker can register an uppercase raw V2 label directly to a victim who already owns the corresponding lowercase name. When the victim uses Portal to transfer the uppercase gift to the attacker, the transaction modal continues to identify the uppercase name, but the connected wallet signs `safeTransferFrom` for the lowercase name's ERC-1155 token ID. The verified result is that the attacker receives the victim's separate lowercase ENS NFT while the uppercase gift remains owned by the victim.

## Vulnerability Details

### Attack scenario

There are two actors and two distinct ENS V2 tokens:

* The victim owns the legitimate lowercase name, for example `caseasset-1908262601.eth`.
* The attacker registers the raw uppercase label `CASEASSET-1908262601` directly to the victim as an unsolicited gift.
* The attacker is the intended recipient when the victim later transfers that gift.

The private-fork proof confirms that the uppercase and lowercase labels remain distinct registrations with different on-chain token IDs:

| Registration               | Owner before transfer | Token ID                                                                         |
| -------------------------- | --------------------- | -------------------------------------------------------------------------------- |
| `caseasset-1908262601.eth` | Victim                | `113537142548103639294285796724011834889495088856512828544734495664743420788736` |
| `CASEASSET-1908262601.eth` | Victim                | `30959008988886769882135030643907345685439962380014932284671577023150141997056`  |

{% stepper %}
{% step %}

## The victim already owns the lowercase ENS NFT

{% endstep %}

{% step %}

## The attacker pays to register the raw uppercase variant directly to the victim

{% endstep %}

{% step %}

## Authentic Panoptes indexing returns both registrations as separate victim-owned rows and preserves the uppercase name

{% endstep %}

{% step %}

## The victim clicks the uppercase row in Portal

{% endstep %}

{% step %}

## The overview child displays `Invalid name`, but the enclosing profile layout and raw-name sidebar remain active

{% endstep %}

{% step %}

## The victim follows the normal `Ownership` and `Transfer` links exposed by that sidebar

{% endstep %}

{% step %}

## The victim enters the attacker-controlled recipient and starts the transfer

{% endstep %}

{% step %}

## Portal's transaction modal says `Transfer name - CASEASSET-1908262601.eth`

{% endstep %}

{% step %}

## The victim opens the connected wallet and approves the transaction described by Portal

{% endstep %}

{% step %}

## Portal signs and broadcasts `safeTransferFrom(victim, attacker, LOWERCASE_TOKEN_ID, 1, 0x)`

{% endstep %}

{% step %}

## The attacker becomes owner of `caseasset-1908262601.eth` while `CASEASSET-1908262601.eth` remains owned by the victim

{% endstep %}
{% endstepper %}

The security invariant should be:

```
selected registration == displayed registration == permission-checked registration == signed token ID
```

The vulnerable flow instead produces:

```
selected, displayed, and owner-checked registration = uppercase gift
role and token lookup                              = lowercase target name
signed ERC-1155 token ID                           = lowercase target name
```

### Root cause 1: Portal imports and routes the raw indexed name

The owned-name query returns Panoptes' raw `domain.name` without requiring it to equal its normalized form:

```ts
domains(where: { owner: $account }) {
  name
  expiryDate
  subdomainCount
  recordCount
}

for (const domain of domains) {
  result.push({
    name: domain.name,
    // ...
  })
}
```

The names table renders that value as a name entity. `EntityBadge` uses the same raw value as the route parameter:

```tsx
<Link to="/$name" params={{ name }} className={primaryWrapperClass}>
  {pillNode}
</Link>
```

Authentic local Panoptes returned the uppercase registration exactly as:

```json
{
  "id": "CASEASSET-1908262601.eth",
  "name": "CASEASSET-1908262601.eth",
  "labelName": "CASEASSET-1908262601",
  "owner": {
    "id": "0x70997970c51812dc3a010c7d01b50e0d17dc79c8"
  },
  "parent": null
}
```

### Root cause 2: the normalization guard protects only the overview child

The overview route correctly checks whether the name is normalized and renders `Invalid name` for the uppercase form:

```ts
const isValidName = isValidEnsName(name)

if (!isValidName) {
  return <InvalidNameMessage title="Invalid name" />
}
```

However, the parent `/$name` layout still resolves the raw route value and always renders `ProfileSidebar` around the child outlet:

```tsx
<SidebarProvider>
  <ProfileSidebar name={name} />
  <SidebarInset>
    <Outlet />
  </SidebarInset>
</SidebarProvider>
```

`ProfileSidebar` constructs the `Ownership` link from that same raw name. The nested ownership and transfer routes do not repeat the canonical-name guard. The victim can therefore follow production UI controls from the rejected overview into the destructive transfer path.

The browser proof explicitly asserts all three facts:

```
overviewGuardRejected=true
productionSidebarNavigation=true
routeName=CASEASSET-1908262601.eth
```

### Root cause 3: the owner check stays uppercase while the role check switches to lowercase

The transfer route accepts any raw string with two dot-separated labels:

```ts
export const is2LD = (name: string): boolean =>
  name.split('.').length === 2
```

It then asks the shared ENS owner resolver for the raw uppercase route name. That resolver passes the same raw string to ENSjs V2 owner and registry lookup. ENSjs serializes that name directly as DNS bytes, so the owner gate correctly verifies the victim's ownership of the uppercase gift. ENSjs and viem behave correctly for the raw input Portal supplies here. The application flaw is that Portal later changes the identity used for the role and token lookups without binding it back to this owner-checked registration.

The split begins only after that owner authorization. The transfer-role check explicitly calls `getLabel(name)`:

```ts
const label = useMemo(() => {
  try {
    return getLabel(name)
  } catch {
    return null
  }
}, [name])
```

`getLabel` first applies ENS normalization:

```ts
export function getLabel(name: string): string {
  const normalized = ens_normalize(name)
  const labels = ens_split(normalized)
  const first = labels[0]
  // ...
  return String.fromCodePoint(...first.input)
}
```

For the proof values:

```
getLabel("CASEASSET-1908262601.eth") == "caseasset-1908262601"
```

Portal therefore checks `ROLE_CAN_TRANSFER_ADMIN` for the lowercase token rather than for the uppercase registration the victim selected. The registry-role helper and ENSjs correctly hash the caller-supplied label. Portal has already substituted that label through `getLabel(name)` before calling them.

### Root cause 4: the final token lookup uses the normalized label while preview metadata keeps the raw name

The transfer preparation obtains its authoritative token ID by normalizing the route name through `getLabel`:

```ts
mutationFn: ({ recipient, options }) =>
  getEnsTokenId({
    label: getLabel(name),
    registryAddress,
  }).map((tokenId) => ({ recipient, tokenId, options }))
```

That result is stored and later supplied to the transfer step:

```ts
transferToken({
  ...common,
  registryAddress,
  tokenId,
  recipient,
  id,
})
```

ENSjs correctly hashes the label supplied by Portal and asks the selected registry for that label's token ID. The substituted identity is introduced by Portal's preceding `getLabel(name)` call, not by ENSjs.

At the same time, the transaction presented in the modal continues to use the unmodified raw route value:

```ts
return steps.map((step, i) => ({
  id: `transfer-${name}-${step}`,
  title: STEP_LABELS[step],
  transactionName: `${STEP_LABELS[step]} - ${name}`,
  // ...
}))
```

The production transaction-state component renders this raw-name field as the visible transaction heading:

```tsx
<h3 className="text-base font-medium whitespace-normal leading-snug break-all min-w-0">
  {transaction.transactionName}
</h3>
```

The final helper also uses the raw name only for transaction-manager metadata:

```ts
{
  id,
  description: `Transfer ${name}`,
  publicClient,
  chainId,
}
```

The authorization-bearing calldata instead uses the already substituted token ID:

```ts
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})
```

There is no comparison between the selected raw registration's identity and the normalized token ID before the wallet is opened.

### Full call path

```
Attacker-created uppercase V2 registration
        |
        v
Panoptes domains.name preserves uppercase bytes
        |
        v
getV2NamesWithRolesForAddress returns raw name
        |
        v
Names table and EntityBadge route to raw /$name
        |
        v
Profile layout preserves sidebar after overview says Invalid name
        |
        v
Ownership and Transfer routes accept raw two-label name
        |
        +--> owner check authorizes the raw uppercase gift
        |
        +--> role check and getLabel(raw uppercase) use the lowercase target
        |
        v
getEnsTokenId(lowercase label) returns victim's target token ID
        |
        +--> modal metadata still says uppercase gift
        |
        v
transferToken encodes safeTransferFrom with lowercase token ID
        |
        v
Victim approves connected-wallet transaction
        |
        v
Lowercase NFT moves to attacker; uppercase gift does not move
```

### Verified browser-to-chain result

I reproduced the complete path in a private local Sepolia fork using the frozen Portal source, the real V2 registrar and registry state, authentic local Panoptes indexing, the production names list, profile layout, sidebar navigation, ownership and transfer routes, transaction modal, transaction manager, a connected headless wallet, and the final ERC-1155 state transition.

The latest isolated evidence run produced:

| Item                        | Verified value                                                                   |
| --------------------------- | -------------------------------------------------------------------------------- |
| Competition commit          | `cda79acaad59711b943fc68207ebb3f1d0ff8596`                                       |
| Imported upstream commit    | `63772fd872af472ced58b009499355f3430c2a86`                                       |
| Node                        | `v22.23.2`                                                                       |
| pnpm                        | `10.27.0`                                                                        |
| Raw gift shown by Portal    | `CASEASSET-1908262601.eth`                                                       |
| Target victim name          | `caseasset-1908262601.eth`                                                       |
| Uppercase gift token ID     | `30959008988886769882135030643907345685439962380014932284671577023150141997056`  |
| Lowercase target token ID   | `113537142548103639294285796724011834889495088856512828544734495664743420788736` |
| Transaction modal           | `Transfer name - CASEASSET-1908262601.eth`                                       |
| Signed function             | `safeTransferFrom`                                                               |
| Signed token ID             | Lowercase target token ID                                                        |
| Local transaction           | `0xa95352c5ba00e640cbf5461e7839ff2b6e776fb0a8c8cffbd1191f88d11055f5`             |
| Lowercase owner before      | Victim                                                                           |
| Lowercase owner after       | Attacker                                                                         |
| Uppercase gift owner before | Victim                                                                           |
| Uppercase gift owner after  | Victim                                                                           |
| Playwright result           | `1 passed`                                                                       |
| Joined result               | `PASS`                                                                           |

The decoded browser-signed arguments were:

```
safeTransferFrom(
  from  = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8,
  to    = 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC,
  id    = 113537142548103639294285796724011834889495088856512828544734495664743420788736,
  value = 1,
  data  = 0x
)
```

That token ID is the lowercase name's ID and is not the uppercase gift's ID.

### Preconditions and boundaries

The following requirements are explicit:

* The victim owns a lowercase V2 `.eth` name.
* The attacker registers the corresponding raw uppercase label directly to the victim.
* The victim chooses to transfer the unsolicited uppercase gift to an attacker-controlled address.
* The victim opens the connected wallet and approves the transaction after Portal identifies it as the uppercase name.
* Both names must still exist and be victim-owned when the flow begins. The raw owner gate therefore authorizes the uppercase gift, while the victim also has authority over the distinct lowercase token selected later.

The following are not required:

* No stolen private key, session key, HCA authority, or browser storage access.
* No malicious RPC, indexer, backend, wallet extension, or third-party provider.
* No modification of transaction bytes after wallet approval.
* No privileged contract role held by the attacker on the lowercase name.
* No public-network transaction in the proof.

The recipient is not substituted. The victim enters the attacker-controlled address. The exploit substitutes the name and ERC-1155 token ID being transferred.

The overview page's `Invalid name` message is a real partial control and is disclosed. It does not block the parent profile sidebar, the Ownership page, the Transfer page, the enabled Transfer action, the transaction modal, or the wallet submission. The final Portal modal still names the uppercase gift.

### Difference from known issue EXP-INPUT-005

This report does not submit the known Medium display-confusion issue.

EXP-INPUT-005 describes validators accepting homoglyphs, bidi and control characters, and leading or trailing dots. This proof uses ASCII case-folding on an otherwise ordinary ASCII label. More importantly, it establishes a new Critical consequence:

* The uppercase and lowercase strings are separate, simultaneously active on-chain registrations.
* Portal imports and displays the uppercase registration as a victim-owned asset.
* Portal derives a different lowercase token ID only inside the destructive authorization path.
* The connected wallet signs that different token ID.
* The victim's lowercase NFT is transferred to the attacker while the selected uppercase NFT remains with the victim.

The competition instructions explicitly distinguish display confusion from a security consequence that changes the identity or transaction being authorized. This is a signed-token and direct-NFT-theft consequence, not a visual spoof alone.

### Controls that work but do not stop the exploit

* The overview route rejects the uppercase name as invalid. Destructive nested routes remain reachable through the still-mounted sidebar.
* Malformed names that make `getLabel` throw fail closed. ASCII uppercase is valid input to normalization and silently becomes lowercase.
* The transfer route correctly checks that the connected address owns the raw uppercase gift. That owner identity is never bound to the different lowercase token ID derived later.
* The transfer-role guard checks `ROLE_CAN_TRANSFER_ADMIN`. It checks that role for the substituted lowercase label and therefore authorizes the wrong token.
* Recipient validation rejects self-transfer and the zero address. The attacker supplies an ordinary valid recipient.
* The chain, registry contract, `from`, recipient, amount, and calldata are internally consistent. The wrong field is the authorization-bearing token ID.
* The chain enforces the signed calldata exactly. This proves the victim approved the substituted lowercase token ID rather than protecting the victim from the misleading application description.

### Recommended remediation

{% stepper %}
{% step %}

## Require canonical names on actionable routes

Require every actionable `/$name/*` route to reject a name unless its supplied bytes equal the one canonical form accepted by the application. Do this at the parent route before rendering the sidebar or any nested destructive route.
{% endstep %}

{% step %}

## Preserve the selected asset identity

Do not normalize an indexed or selected asset into another asset during authorization. Carry an immutable domain identifier and the exact token ID from selection through owner checks, preview, signing, and post-state verification.
{% endstep %}

{% step %}

## Validate the token before opening the wallet

Before opening the wallet, verify that the token ID to be signed is the token ID of the exact registration the user selected. Abort if normalization, case-folding, or any parser transformation changes that identity.
{% endstep %}

{% step %}

## Show exact security-critical transaction details

Include the exact name and token ID in the security-critical transaction review. Do not build the visible transaction name from one string while encoding the token from another.
{% endstep %}

{% step %}

## Add regression tests

Add browser-to-chain regression tests for uppercase/lowercase pairs, NFD/NFC pairs, variation selectors, and every other accepted input whose normalized representation differs from its indexed representation.
{% endstep %}
{% endstepper %}

## Impact Details

This maps directly to two listed Critical Websites & Applications impacts.

First, it causes direct theft of a user NFT. The verified execution transferred the victim's lowercase ENS V2 ERC-1155 NFT to the attacker. The victim-authorized uppercase gift did not move and remained owned by the victim.

Second, it causes a malicious interaction with an already-connected wallet by modifying a transaction argument. Portal tells the victim that the transaction is `Transfer name - CASEASSET-1908262601.eth`, but supplies the lowercase name's distinct token ID as the third argument to `safeTransferFrom`. The wallet signs and the chain executes that substituted argument.

From the attacker's perspective, the attacker creates a raw uppercase gift for the victim and receives the victim's separate lowercase name if the victim later chooses to transfer that gift to the attacker.

From the victim's perspective, the victim selects the uppercase row, follows Portal's normal ownership flow, enters the intended recipient, and approves a modal that still names the uppercase asset. The on-chain result transfers a different ENS NFT.

The loss is the victim's ownership of the lowercase ENS name and the value and control associated with that name. The proof does not rely on estimating a market price. It demonstrates the program's listed direct-NFT-theft state transition exactly.

## References

[Panoptes domain names are returned without a canonical-form invariant](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/dashboard/hooks/useV2NamesWithRolesForAddress.ts#L41-L95)

[The names table renders the raw name as an EntityBadge](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/names/components/NamesTable/columns.tsx#L22-L64)

[EntityBadge routes the raw name into the profile route](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/components/EntityBadge.tsx#L258-L264)

[The parent name route resolves the raw name and always mounts ProfileSidebar](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name.tsx#L22-L74)

[The overview child displays Invalid name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/index.tsx#L438-L460)

[ProfileSidebar keeps the raw name in its Ownership link](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/components/ProfileSidebar.tsx#L50-L82)

[The Ownership route exposes Transfer without enforcing canonical name equality](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/ownership/index.tsx#L27-L120)

[The transfer route enforces wallet, ENSv2, owner, label-count, and role gates but omits canonical-name equality](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/ownership/transfer.tsx#L22-L177)

[Portal passes the supplied route name into its shared owner resolver](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/profile/hooks/useEnsOwner.ts#L34-L60)

[The shared owner resolver passes the raw name to ENSjs V2 owner and registry lookups](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/ens/resolveEnsOwner.ts#L44-L61)

[The shared owner resolver calls that V2 path with the supplied name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/ens/resolveEnsOwner.ts#L72-L92)

[ENSjs sends the supplied name directly to Universal Resolver findOwner as DNS bytes](https://github.com/ensdomains/ensjs/blob/e96662c22a73e6bc2e4eb23b66785defab4feb8c/packages/ensjs/src/actions/public/v2/getOwner.ts#L36-L54)

[ENSjs sends the same supplied name directly to Universal Resolver findRegistries as DNS bytes](https://github.com/ensdomains/ensjs/blob/e96662c22a73e6bc2e4eb23b66785defab4feb8c/packages/ensjs/src/actions/public/v2/getNameRegistries.ts#L37-L57)

[viem packetToBytes preserves label case and only strips a leading or trailing dot before byte encoding](https://github.com/wevm/viem/blob/1b6a8880005a41fc3fd410d6a8f9ab9e6749f452/src/utils/ens/packetToBytes.ts#L30-L52)

[getLabel normalizes before extracting the label](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/token/getLabel.ts#L10-L20)

[The transfer-role check uses the normalized label](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useCanTransferName.ts#L49-L81)

[Portal's registry-role hook forwards the label selected by the transfer-role check](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/registry/hooks/useHasRoles.ts#L49-L67)

[ENSjs registry-role lookup hashes the caller-supplied label and passes that resource to hasRoles](https://github.com/ensdomains/ensjs/blob/e96662c22a73e6bc2e4eb23b66785defab4feb8c/packages/ensjs/src/actions/public/v2/hasRoles.ts#L137-L156)

[The transfer flow derives the token ID from getLabel but keeps the raw name in modal metadata](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferName.ts#L106-L180)

[getEnsTokenId delegates the normalized label to ENSjs](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/profile/hooks/useTokenId.ts#L21-L33)

[ENSjs token lookup hashes the caller-supplied label and asks the selected registry for its current token ID](https://github.com/ensdomains/ensjs/blob/e96662c22a73e6bc2e4eb23b66785defab4feb8c/packages/ensjs/src/actions/public/v2/getTokenId.ts#L28-L50)

[The production transaction-state component displays transactionName](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transaction-manager/components/TransactionStateContent.tsx#L110-L114)

[The final helper encodes safeTransferFrom with the substituted token ID while describing the raw name](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/helpers/transferToken.ts#L46-L96)

## Proof of Concept

### Safety and environment

This proof runs entirely against an isolated local Anvil fork using throwaway development accounts. It does not send transactions to public Sepolia or any other public network. It does not use production funds, names, wallets, sessions, API credentials, or a public Panoptes deployment.

The supplied `ASCII_CASE_SEPOLIA_FORK_URL` is used only by Anvil to read the initial Sepolia state. Every registration, wallet approval, transfer, and ownership assertion executes locally inside the disposable fork.

The runner uses pinned Anvil and Panoptes container image digests. It:

* Requires a researcher-controlled Sepolia RPC URL and does not contain a fallback provider.
* Requires the exact frozen competition commit.
* Refuses to reuse its Compose project or any occupied local port.
* Uses dedicated loopback ports for Anvil, Panoptes, and Portal.
* Uses the standard throwaway Anvil mnemonic only inside the private fork.
* Starts a local Portal Vite instance from the frozen source.
* Uses authentic local Panoptes indexing rather than a mocked name response.
* Stops and removes only the containers, network, and volume it created, including after failure.

### PoC evidence bundle

The complete runnable PoC, clean verification log, and screenshots are available here:

[Download ens-portal-uppercase-token-substitution-poc.zip](https://drive.google.com/file/d/1wGX0P0RRYrQbWMJv7gSJBG7u1X0btjo4/view?usp=sharing)

SHA-256:

```
13787b8d6d8a3addea126aa1b7540c9e040ea8e0368943e02f054c7b872ce852  ens-portal-uppercase-token-substitution-poc.zip
```

The archive contains:

```
poc/registration-jawdrop/run-ascii-case-transfer-clean.sh
poc/registration-jawdrop/ascii-case-transfer.ts
poc/registration-jawdrop/ascii-case-transfer.compose.yml
poc/registration-jawdrop/ascii-case-panoptes.json
poc/registration-jawdrop/vite.ascii-case.config.ts
e2e/poc/portal-ascii-case-transfer.playwright.config.ts
e2e/projects/portal/tests/portal-ascii-case-transfer.poc.spec.ts
scans/registration-jawdrop-ascii-case-clean.log
scans/registration-jawdrop-ascii-case-dashboard.png
scans/registration-jawdrop-ascii-case-modal.png
scans/registration-jawdrop-ascii-case-panoptes.json
POC-README.md
SHA256SUMS.txt
```

### Prerequisites

* Linux with Bash.
* Node.js v22.
* Corepack and pnpm v10.
* Docker with Docker Compose v2.
* Chromium installed at `/usr/bin/chromium`.
* Git, curl, jq, unzip, and `ss`.
* A researcher-controlled Sepolia RPC URL with ordinary read and fork support.
* Local ports 19545, 19655, and 19301 available. Alternate ports can be supplied through the documented environment variables.
* Network access for the initial frozen dependency installation, pinned container image download, and read-only fork initialization.

{% stepper %}
{% step %}

## Obtain the frozen competition repository

Create a clean working directory and clone the in-scope repository:

```bash
mkdir ens-portal-uppercase-reproduction
cd ens-portal-uppercase-reproduction

git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-ens.git \
  audit-comp-ens

cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
```

Verify the competition commit:

```bash
test "$(git rev-parse HEAD)" = \
  "cda79acaad59711b943fc68207ebb3f1d0ff8596"

git show -s --format='%H%n%B' HEAD
```

The commit message identifies the imported upstream commit as:

```
63772fd872af472ced58b009499355f3430c2a86
```

{% endstep %}

{% step %}

## Install frozen dependencies

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

{% endstep %}

{% step %}

## Download, verify, and overlay the PoC

Download the Google Drive archive and save it as:

```
ens-portal-uppercase-token-substitution-poc.zip
```

Place it beside the cloned `audit-comp-ens` directory, then verify and extract it:

```bash
cd ..

printf '%s  %s\n' \
  '13787b8d6d8a3addea126aa1b7540c9e040ea8e0368943e02f054c7b872ce852' \
  'ens-portal-uppercase-token-substitution-poc.zip' | sha256sum -c -

unzip ens-portal-uppercase-token-substitution-poc.zip -d audit-comp-ens
chmod +x audit-comp-ens/poc/registration-jawdrop/run-ascii-case-transfer-clean.sh

cd audit-comp-ens
bash -n poc/registration-jawdrop/run-ascii-case-transfer-clean.sh
```

The checksum command must print:

```
ens-portal-uppercase-token-substitution-poc.zip: OK
```

The decisive included evidence files have these checksums:

```
006160ec3bcd9fcde97af20928549179fd62406d28b710f590b9a0ea371bdba2  scans/registration-jawdrop-ascii-case-clean.log
2d8b05fd685756c5214ad1722a4d2a37ab88b16f9a2f87601674220e63bd5673  scans/registration-jawdrop-ascii-case-dashboard.png
0a340a5b6d0372f0111c4cbf720e6ee526d9c40162fab289d14fb55cdd449b6  scans/registration-jawdrop-ascii-case-modal.png
b71e9a27428f340d77e365167b3a17359a563e3bbf2353ac86b03d8ba81bdc25  scans/registration-jawdrop-ascii-case-panoptes.json
```

{% endstep %}

{% step %}

## Supply the private-fork endpoint

Supply a researcher-controlled endpoint without putting it in source files or the shell command history:

```bash
read -rsp 'Researcher-controlled Sepolia RPC URL: ' ASCII_CASE_SEPOLIA_FORK_URL
export ASCII_CASE_SEPOLIA_FORK_URL
printf '\n'
```

Running without this variable fails before creating or changing any local service:

```
ASCII_CASE_SEPOLIA_FORK_URL must be set to a researcher-controlled Sepolia RPC URL.
```

{% endstep %}

{% step %}

## Execute the isolated browser-to-chain proof

Optionally choose a unique lowercase ASCII label containing at least one letter:

```bash
export CASE_LABEL="caseasset-$(date +%s)"
```

Run the joined proof from the repository root:

```bash
poc/registration-jawdrop/run-ascii-case-transfer-clean.sh \
  2>&1 | tee scans/triager-uppercase-transfer-run.log
```

The runner starts Anvil, local Panoptes indexing and API services, and Portal. It then executes one Playwright test with one worker.
{% endstep %}

{% step %}

## Observe two distinct registrations

The setup stage registers the lowercase name to the victim. The attacker then pays to register the uppercase raw label directly to the same victim.

The output must contain `SETUP_PASS` and show:

```json
{
  "lowerName": "caseasset-1908262601.eth",
  "upperName": "CASEASSET-1908262601.eth",
  "victim": "0x70997970C51812dc3A010C7d01b50e0d17dc79C8",
  "attacker": "0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC"
}
```

The actual numeric suffix follows `CASE_LABEL` and may differ. The two printed token IDs must be unequal. The runner aborts if either registration is missing, either owner is wrong, or the token IDs collapse.
{% endstep %}

{% step %}

## Observe authentic Panoptes preserving the uppercase gift

The runner waits for local Panoptes to index the fork transactions and then queries its GraphQL API. It must print:

```
authenticPanoptesUppercaseGift=PASS
```

The browser test independently reads that row and requires all of the following:

```
name      = CASEASSET-<suffix>.eth
labelName = CASEASSET-<suffix>
owner     = victim
parent    = null
```

No mocked indexer response is permitted. The test exits if `E2E_MOCK_INDEXER=true`.
{% endstep %}

{% step %}

## Observe the production Portal path

The Playwright test connects the victim's throwaway headless wallet and opens the production address names page.

It asserts that:

1. Authentic Panoptes supplies the uppercase gift row, while setup has already proved that its on-chain identity differs from the lowercase registration.
2. The victim clicks the uppercase row. The included dashboard screenshot also shows both registrations.
3. The raw uppercase value remains in the route.
4. The overview page displays the honest partial warning `Invalid name`.
5. The enclosing production sidebar still exposes `Ownership`.
6. The Ownership page exposes `Transfer`.
7. The victim enters the attacker address and starts the transfer.
8. The transaction modal displays `Transfer name - CASEASSET-<suffix>.eth`.

The runner writes the visual evidence to:

```
scans/registration-jawdrop-ascii-case-dashboard.png
scans/registration-jawdrop-ascii-case-modal.png
```

The dashboard screenshot shows both distinct registrations. The modal screenshot shows the uppercase gift as the named transfer immediately before the wallet is opened.
{% endstep %}

{% step %}

## Observe the substituted signed token ID

The test authorizes the transaction through the connected local wallet, locates the exact mined browser transaction, and decodes its calldata.

For the included evidence run, Portal displayed:

```
Transfer name - CASEASSET-1908262601.eth
```

The browser transaction decoded as:

```
safeTransferFrom(
  from  = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8,
  to    = 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC,
  id    = 113537142548103639294285796724011834889495088856512828544734495664743420788736,
  value = 1,
  data  = 0x
)
```

The signed ID is the lowercase target token ID. It is not the uppercase gift token ID:

```
uppercase gift token ID:
30959008988886769882135030643907345685439962380014932284671577023150141997056

lowercase target token ID:
113537142548103639294285796724011834889495088856512828544734495664743420788736
```

The included local browser transaction is:

```
0xa95352c5ba00e640cbf5461e7839ff2b6e776fb0a8c8cffbd1191f88d11055f5
```

This is a private-fork transaction hash and may differ on rerun.
{% endstep %}

{% step %}

## Observe the NFT ownership result

The browser test reads both token states before and after the transaction. It requires this exact outcome:

```
lowercase owner before = victim
lowercase owner after  = attacker

uppercase owner before = victim
uppercase owner after  = victim
```

This proves the selected and displayed gift did not transfer. The victim's different lowercase NFT transferred to the attacker.
{% endstep %}

{% step %}

## Observe successful completion and cleanup

The final output must contain:

```
1 passed
browserResult=PASS
joinedResult=PASS
```

The exit trap then removes the dedicated Panoptes API, Panoptes indexer, Anvil container, volume, and network.

Confirm that no PoC services remain:

```bash
ASCII_CASE_SEPOLIA_FORK_URL="$ASCII_CASE_SEPOLIA_FORK_URL" \
docker compose \
  --project-name asciicasetransferpoc \
  -f poc/registration-jawdrop/ascii-case-transfer.compose.yml \
  ps --status running --services
```

It must print no service names.
{% endstep %}
{% endstepper %}

### Negative controls

* The uppercase and lowercase strings are unequal and their on-chain token IDs are unequal.
* Both registrations are owned by the victim before transfer.
* Authentic Panoptes returns the uppercase gift rather than a mocked or altered row.
* The overview child rejects the uppercase input, proving the report does not hide that warning.
* Production sidebar navigation still reaches Ownership and Transfer despite that warning.
* The modal continues to identify the uppercase gift before wallet approval.
* The exact browser transaction decodes to the lowercase token ID.
* The lowercase NFT changes owner and the uppercase gift does not.
* The recipient address, chain, registry, amount, and calldata decode are asserted. The divergence is isolated to the NFT token ID.
* The proof uses no public-chain writes, compromised RPC, malicious indexer, stolen key, session token, HCA authority, or privileged role on the victim's lowercase name.

### Proof boundaries

The victim must intentionally approve a transfer to the attacker-controlled recipient. The security issue is that Portal names one NFT in its selection and confirmation but signs a different NFT's token ID.

The overview page visibly says `Invalid name`. The proof does not claim otherwise. The exploitable failure is that the parent layout keeps the raw-name sidebar active, the nested destructive routes omit the same guard, and the final transaction modal affirmatively identifies the uppercase gift.

The proof does not claim a smart-contract vulnerability. Contract behavior is used only to establish the two distinct registrations and verify the application-created state transition.
