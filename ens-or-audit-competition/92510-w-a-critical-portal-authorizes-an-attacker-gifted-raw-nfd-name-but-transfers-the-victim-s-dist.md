# 92510 w a critical portal authorizes an attacker gifted raw nfd name but transfers the victim s distinct nfc erc 1155 name token

**Submitted on Sep 10th 2026 at 11:39:03 UTC by @IronsideSec for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92510
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs

## Description

## Brief/Intro

Portal (the in-scope **Explorer app**) uses two different on-chain identities in one ownership-transfer flow. It reads and displays the raw decomposed NFD name `cafe\u0301.eth`, but its role and token-ID helpers normalize that name to composed NFC `café.eth`. An attacker can permissionlessly register the raw-NFD name directly to a victim who already owns the visually identical NFC name. When the victim uses Explorer to return the gifted name to the attacker, the single wallet request transfers the victim's different NFC ERC-1155 token. The valuable NFC name and its roles move to the attacker, while the NFD name shown by Explorer remains with the victim. The victim cannot recover the stolen name without attacker cooperation.

This is not a request to recognize a new invalid character. At the frozen revision, Explorer's sibling `/$name` overview already calls `isValidEnsName` and rejects this exact raw name as **Invalid name**. The nested state-changing `/$name/ownership/transfer` route omits that established guard, authorizes the raw resource, and silently builds calldata for the normalized resource. Even if this shares the normalization primitive disclosed in `EXP-INPUT-005`, the competition expressly keeps new consequences that materially change severity in scope; this path changes Medium UI confusion into Critical direct NFT theft.

## Vulnerability Details

### Security boundary and root cause

ENS V2 can store non-normalized raw labels as distinct registry resources. That behavior alone is not the reported vulnerability. The in-scope Portal vulnerability is that one transaction workflow does not bind authorization, display, and token selection to the same resource:

1. Portal passes the raw route string to owner and registry discovery.
2. Portal displays that same raw route string and uses its owner to authorize the page.
3. Portal then calls its own `getLabel(name)`, which performs ENS normalization.
4. Portal uses the normalized label for the transfer-role check and ERC-1155 token lookup.
5. Portal labels the modal with the raw name but submits `safeTransferFrom` for the normalized name's different token ID.

If the raw and normalized labels identify different registered V2 resources, the owner gate applies to resource A while the wallet transaction transfers resource B.

### Production-valid colliding resources

The exploit uses two different UTF-8 labels that normal Unicode rendering displays identically:

| Field              | Attacker-gifted raw-NFD decoy                                        | Victim's NFC target                                                  |
| ------------------ | -------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Escaped name       | `cafe\u0301.eth`                                                     | `caf\u00e9.eth`                                                      |
| Visible text       | `café.eth`                                                          | `café.eth`                                                           |
| Code points        | `c a f e U+0301`                                                     | `c a f U+00E9`                                                       |
| Leaf UTF-8         | `0x63616665cc81`                                                     | `0x636166c3a9`                                                       |
| DNS packet         | `0x0663616665cc810365746800`                                         | `0x05636166c3a90365746800`                                           |
| Raw labelhash      | `0xe7739d357b90d35469fb9a3a5da362861a02e02c559108dc70489172a29a2fb4` | `0x9513447e2d376aacd434727887590dd448cda8f2d30c4ace903d31fe209f8ad8` |
| Version-0 token ID | `0xe7739d357b90d35469fb9a3a5da362861a02e02c559108dc7048917200000000` | `0x9513447e2d376aacd434727887590dd448cda8f2d30c4ace903d31fe00000000` |

`ens_normalize("cafe\u0301.eth")` returns `café.eth`, but the raw hashes, DNS packets, registry entries, and ERC-1155 IDs remain distinct.

A local Chromium `149.0.7827.55` rendering control used Explorer's checked-in PortalSans and PortalSerif fonts. In the sidebar, transaction modal, and serif-name layouts, the NFD and NFC strings had the same CSS bounds and **byte-identical PNG output**. The respective SHA-256 pairs were `2694d6c35d8e91cd1a1f23219f5e4e4f41b3679a758ba675b4320cb512623ae6`, `c6ad874bdbee4eb1096ac1e183981f368970993464a190f087aee185aaf721d8`, and `c6ee49cf1142b2d613a3b63d20d8390fdf64fc88aba88322963c0bb5ad81f741`. The same control confirmed that direct navigation preserves `/cafe%CC%81.eth/ownership/transfer` through the server and decodes the route parameter to NFD rather than silently converting it to NFC.

This longer pair is important. The deployed price oracle rejects the tempting short `e\u0301` / `é` illustration because its leaves have only two and one code points. It accepts both `cafe\u0301` and `café` because they have five and four code points. The PoC includes a negative control for the short pair.

The split is not limited to accented or homoglyph-style names. Two additional passing controls use the exact pinned normalizer and official registry/registrar:

| Raw registered route/resource | `ens_normalize` / `getLabel` target | Why it matters                                                                                                                                              |
| ----------------------------- | ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `VITALIK.eth`                 | `vitalik.eth`                       | Pure ASCII case-folding: no homoglyph, bidirectional mark, or control character. Raw uppercase and lowercase labels have different hashes and ERC-1155 IDs. |
| `vitalik\uFE0F.eth`           | `vitalik.eth`                       | U+FE0F is ignored here and normally has no visible glyph after `k`. The collision can therefore target an ordinary ASCII `.eth` name.                       |

The mounted Explorer test proves both mappings using its pinned `@adraffy/ens-normalize` and `getLabel`. The official F21 registrar test registers each raw/canonical pair as two resources, executes the canonical token transfer selected by Explorer, and proves that the raw decoy remains with the victim. These are corroborating variants of the same report, not separate findings.

### An ordinary attacker can create the decoy without victim approval

The attacker does not need a privileged account, victim signature, malicious contract, or existing authority over the target name:

1. The victim already owns the normally registered NFC name `café.eth`.
2. The attacker commits to raw NFD label bytes `0x63616665cc81` through the public V2 registrar.
3. The attacker reveals/registers that raw label with `owner = victim`, `resolver = address(0)`, and `subregistry = address(0)`, paying the registration cost themselves.
4. Payer and owner are independent registrar arguments, so no gift-acceptance transaction is required from the victim.
5. After indexer ingestion, Explorer's own address dashboard queries `domains(where: { owner: $account })` and passes each returned `domain.name` unchanged into `NameList`.
6. [`NameList`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/dashboard/components/NameList.tsx#L23-L33) renders that raw name through `EntityBadge`; [`EntityBadge`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/components/EntityBadge.tsx#L259-L263) links to `/$name`; and the still-rendered [`ProfileSidebar`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/components/ProfileSidebar.tsx#L75-L82) links the same raw name to **Ownership**. The victim can therefore discover the gift in “names owned by this address,” then navigate to **Ownership → Transfer** without opening an attacker-crafted URL.
7. The attacker only needs to ask the victim to return the unexpected gift to the attacker's visible address. A direct `/cafe%CC%81.eth/ownership/transfer` link is an optional shortcut, not a technical prerequisite.

The official-contract PoC below executes this exact attacker-paid gift through `ETHRegistrar.makeCommitment`, `commit`, and `register`. The dashboard path is not trusted for authorization; the raw on-chain owner check still controls the mutation route.

### Portal authorizes the page using the raw-NFD resource

The [`/$name` route](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name.tsx#L22-L68) takes `params.name` directly from the route. It passes that raw string to `getEnsOwnerQueryOptions` and `getNameRegistriesQueryOptions`, then gives the same string to `ProfileSidebar`:

```tsx
const { name } = params
const ownerData = await queryClient.fetchQuery(
  getEnsOwnerQueryOptions({ name }),
)
// ...
getNameRegistriesQueryOptions({ name: params.name })
// ...
<ProfileSidebar name={name} />
```

The [transfer route](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/ownership/transfer.tsx#L21-L28) again queries the owner using raw `name`. The [transfer authorization branch](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/ownership/transfer.tsx#L103-L110) compares the connected address to that returned owner and passes the same raw name into [`AuthorizedTransfer`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/ownership/transfer.tsx#L141-L196) and `SendNameForm`:

```tsx
const ownerQuery = useQuery(getEnsOwnerQueryOptions({ name }))
// ...
isAddressEqual(address, data.owner)
// ...
<AuthorizedTransfer
  name={name}
  registryAddress={data.registryAddress}
  owner={data.owner}
/>
```

Pinned ENSjs owner/registry traversal encodes this raw string as DNS packet `0x0663616665cc810365746800`. Therefore the attacker-gifted NFD resource makes the victim pass the owner gate.

### Portal changes identity only for authorization roles and token selection

Portal's [`getLabel`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/token/getLabel.ts#L1-L20) does not merely split the leaf. It first normalizes the complete name:

```ts
export function getLabel(name: string): string {
  const normalized = ens_normalize(name)
  const labels = ens_split(normalized)
  const first = labels[0]
  if (!first || first.error) throw first?.error
  return String.fromCodePoint(...first.input)
}
```

For raw `cafe\u0301.eth`, this returns NFC `café`.

The [`useCanTransferName`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useCanTransferName.ts#L49-L75) uses that normalized label for the transfer-role query:

```ts
const label = useMemo(() => {
  try {
    return getLabel(name)
  } catch {
    return null
  }
}, [name])

getHasRolesQueryOptions({
  registryAddress,
  label,
  roles: [TRANSFER_ROLE],
  account,
})
```

The victim owns the NFC target and holds its transfer role, so this check passes even though the raw route was authorized as the separate NFD name.

The decisive sink is [`useTransferName`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferName.ts#L106-L181). Before opening the modal it requests a token ID for the normalized label:

```ts
getEnsTokenId({
  label: getLabel(name),
  registryAddress,
}).map((tokenId) => ({ recipient, tokenId, options }))
```

It later passes that token ID to `transferToken`, but constructs the visible modal identity from the unmodified raw route string:

```ts
transactionName: `${STEP_LABELS[step]} - ${name}`
```

Finally, [`transferToken`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/helpers/transferToken.ts#L46-L92) encodes and submits the standard ERC-1155 transfer:

```ts
const data = encodeFunctionData({
  abi: erc1155Abi,
  functionName: 'safeTransferFrom',
  args: [walletWithAccount.account.address, recipient, tokenId, 1n, '0x'],
})

request: {
  type: 'eoa',
  from: walletWithAccount.account.address,
  to: registryAddress,
  data,
  value: 0n,
  chainId,
}
```

Portal therefore reads and displays the NFD name but submits the NFC token ID.

### Explorer already rejects this input on the sibling overview, but not on the transfer route

The frozen overview route makes an explicit security decision for the same `name` parameter. [`routes/$name/index.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/index.tsx#L438-L460) calls the existing validator and rejects the primary NFD input, the uppercase variant, and the U+FE0F variant:

```tsx
const isValidName = isValidEnsName(name)
// ...
if (!isValidName) {
  return <InvalidNameMessage title="Invalid name" />
}
```

The strengthened mounted test asserts `isValidEnsName("cafe\u0301.eth") === false`. The canonical `café.eth` returns true. This means Explorer already distinguishes these two identities at the sibling route; no new character classification is needed.

By contrast, [`routes/$name/ownership/transfer.tsx`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/ownership/transfer.tsx#L21-L28) does not import or call `isValidEnsName`. It sends raw `name` to the owner query, authorizes the returned owner, and reaches `getLabel(name)` only inside the role/token helpers. Thus a raw name that the overview rejects as invalid remains accepted by the nested state-changing route and is retargeted to another token. This missing mutation-route guard is the in-scope Explorer defect.

### The minimal attacker gift produces one wallet request

The attacker registers the raw-NFD decoy with no resolver, child registry, or ETH address. The [`useTransferDetachTargets`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts#L51-L100) inspects those dependencies using the raw `name`. It finds nothing to detach. Although [`SendNameForm`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L63-L101) initializes all three cleanup toggles to `true`, it masks every one with `optionIsVisible`:

```ts
const effectiveOptions = {
  setEthAddress: options.setEthAddress && optionIsVisible.setEthAddress,
  detachResolver: options.detachResolver && optionIsVisible.detachResolver,
  detachRegistry: options.detachRegistry && optionIsVisible.detachRegistry,
}
```

All effective flags are false, so `buildTransferPlan` contains only `transfer-token`. The victim receives exactly one on-chain wallet request.

### Victim interaction

The victim must participate in the normal transfer flow. Starting from an already-connected wallet, the complete sequence is:

1. discover the unexpected gifted name in Explorer's own address dashboard (or open the equivalent direct route);
2. navigate through **Ownership → Transfer**;
3. enter or select the attacker's visible recipient address;
4. click **Transfer name**;
5. click **Start**;
6. click **Open wallet**; and
7. confirm the one wallet transaction.

The recipient is deliberately visible, and Portal warns that transfers are irreversible. Those controls address recipient mistakes. They do not reveal that the token selected for transfer belongs to a different registered name than the route, owner read, and modal identify. The victim's intended action is to return the apparent NFD gift to the named recipient; the unintended action is transferring their separate NFC NFT.

### Exact route, UI, and wallet calldata

The mounted production test uses this exact path:

```
/cafe%CC%81.eth/ownership/transfer
```

Portal preserves the NFD code points in the route, owner/registry inputs, sidebar, and modal. The transaction details say:

```
Transfer name - cafe\u0301.eth
```

No Portal view identifies the selected NFC ERC-1155 token as `café.eth`.

With victim `0x1111111111111111111111111111111111111111` and attacker `0x2222222222222222222222222222222222222222`, Portal sends one zero-value Sepolia request to the deployed V2 ETH registry `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2`:

```
0xf242432a000000000000000000000000111111111111111111111111111111111111111100000000000000000000000022222222222222222222222222222222222222229513447e2d376aacd434727887590dd448cda8f2d30c4ace903d31fe00000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000000
```

Decoded:

```solidity
safeTransferFrom(
    0x1111111111111111111111111111111111111111,
    0x2222222222222222222222222222222222222222,
    0x9513447e2d376aacd434727887590dd448cda8f2d30c4ace903d31fe00000000,
    1,
    0x
)
```

The requested token is the NFC target. It is not the displayed NFD token `0xe7739d35...00000000`.

The request itself contains only a numeric token ID, not the raw-NFD name that Portal used for authorization. Standard simulation succeeds because the victim genuinely owns the NFC token and holds its transfer role. A metadata-aware wallet could choose to perform extra lookups, but that is not a Portal control and does not bind the transaction to the name shown by the application.

### Exact post-state and lack of unilateral recovery

The official-contract test executes the exact Portal-shaped request and proves:

* victim NFC balance: `1 -> 0`;
* attacker NFC balance: `0 -> 1`;
* NFC ownership-linked roles, including `ROLE_CAN_TRANSFER_ADMIN`, move to the attacker;
* raw-NFD decoy balance and ownership remain with the victim;
* the NFC resolver and subregistry remain attached because cleanup inspected the empty NFD decoy;
* the victim cannot transfer the NFC token back from the attacker;
* the victim cannot use the old role to modify the stolen name's resolver; and
* the attacker can transfer it back, proving that immediate recovery requires attacker cooperation.

After success, Portal navigates back to the raw-NFD ownership page. That resource still belongs to the victim, so the UI can continue to show the victim as owner even though the valuable NFC token was transferred.

### Deployment binding

The Portal lockfile pins ENSjs/ABI commit `e96662c`. Read-only Sepolia checks bind its addresses and the official F21 source as follows:

| Component                         | Current Sepolia address                      |
| --------------------------------- | -------------------------------------------- |
| Universal Resolver entry          | `0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe` |
| Universal Resolver implementation | `0x4a1817d13e9cf196f471725176355c1234b63c70` |
| Root registry                     | `0x8115186e8f2e0b0281e86ab91f0f48ba90364354` |
| ETH registry                      | `0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2` |
| ETH registrar                     | `0xa88553F454b77203B0D036A05c894d555EAAa2Cc` |
| Price oracle                      | `0x8914b66260EB8C4fff795650c3AE8Cd335958987` |
| Label store                       | `0x532cd0cc4ac0793d838f71a67d29b2d790d18777` |

The registry, registrar, oracle, resolver implementation, root registry, and label store runtime match official contracts-v2 commit `f21e4b4a38c949a6fc5c7d3bdd1b418292c880f1` after masking only compiler-declared immutable regions.

A separate immutable local fork at Sepolia block `11,672,206` (block hash `0x2670898a8974e578bb7d07ba917d53a824e527ed1a6bfb3e7a84f1dc9eeaffa3`) provisions both resources through the deployed public registrar `0xa885...a2Cc`, executes the Explorer-shaped request against deployed registry `0xBDC8...F0E2`, and reproduces the same `1 -> 0` / `0 -> 1` target balance transition while the raw decoy remains with the victim. It also asserts the deployed root registry's `getResolver("eth") == address(0)`, the raw gift's resolver and subregistry are zero, and the Universal Resolver resolves the raw DNS packet to the victim. Those facts are why the empty decoy yields one wallet request rather than preliminary detach calls.

The fork uses `auditproof-cafe\u0301-926431` / `auditproof-café-926431`, not the short report labels. The deterministic suffix makes both resources provably available at the pinned historical block, so the test can assign controlled victim/attacker state without assuming or altering the existing ownership of `café.eth`. The normalization relation, deployed contracts, owner split, token-selection split, calldata shape, and theft post-state are otherwise identical. Consequently, the fork's label hashes intentionally differ from the primary illustrative pair. No public transaction was sent.

### Eligibility despite `EXP-INPUT-005`: existing guard bypass and materially higher severity

The eligibility basis is explicit: the competition keeps in scope **“new consequences of a listed root cause that materially change its severity.”** This report does not hide the shared normalization primitive. Even if that primitive is grouped with `EXP-INPUT-005`, the demonstrated consequence changes from stated Medium display confusion to Critical direct NFT theft and therefore fits the stated exception.

`EXP-INPUT-005` says:

> Our name validators accept inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI rather than incorrect resolution.

The overlap ends at normalization transforming an input. The security boundary and consequence are materially different:

| `EXP-INPUT-005`                                                                | This report                                                                                                      |
| ------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| Input-validation/display issue                                                 | inconsistent resource identity inside a state-changing transfer                                                  |
| consequence expressly limited to UI confusion rather than incorrect resolution | raw resource A authorizes the page; normalized resource B becomes the ERC-1155 calldata target                   |
| no second on-chain asset or token sink                                         | two registered resources and two ERC-1155 IDs execute through official contracts                                 |
| no ownership post-state                                                        | victim NFT and ownership-linked roles move to attacker with no unilateral recovery                               |
| validator must classify a problematic input                                    | Explorer's sibling overview already rejects the exact input; the nested transfer route omits that existing guard |

The pure ASCII `VITALIK`/`vitalik` control further shows that the exploitable invariant is not specific to homoglyphs, bidirectional marks, or control characters. A stricter name validator is one possible defense, but it is not the only fix. The direct invariant is that owner discovery, role checks, token selection, display, and calldata must refer to one resource. Explorer breaks that invariant by applying `getLabel` only to role and token selection after raw-resource authorization.

### Distinction from ENSjs PR #361

Public ENSjs PR [#361](https://github.com/ensdomains/ensjs/pull/361) describes a raw-versus-normalized “wrong id” in the dependency's `getAvailable` read path. It does not identify or change:

* Portal's ownership-transfer route;
* Portal's raw owner authorization;
* Portal's `getLabel` use in `useCanTransferName` and `useTransferName`;
* ERC-1155 `safeTransferFrom` construction;
* the raw-name transaction modal; or
* direct NFT ownership loss.

This report does not claim ENSjs `getAvailable` as the vulnerable asset. The in-scope bug is Portal's own mixed raw/normalized identity policy in a state-changing transfer. The exact Portal sink and Critical post-state are not described by that read-only availability fix.

### Recommended remediation

For every state-changing name route:

1. Reject the operation unless the decoded route is byte-for-byte equal to `ens_normalize(routeName)`. Do not silently redirect a raw registered resource to a canonical-equivalent resource.
2. If legacy raw names must remain manageable, create one explicit identity object that binds raw DNS bytes, raw label hash, registry resource, token version/ID, and display string.
3. Use that same identity for owner discovery, role checks, detach discovery, token selection, confirmation text, and calldata.
4. Immediately before wallet submission, assert that the selected token ID belongs to the exact raw label whose owner authorized the page.
5. Show an explicit noncanonical-name warning and both the raw identity and token ID where legacy resources are supported.
6. Add paired NFD/NFC, casefold, variation-selector, width-mapping, and ignored-code-point tests to every mutation flow.

This must be enforced as a shared identity policy, not patched only in one component: the same `getLabel` normalization helper is also used in renewal checks/builders and subname actions. Those routes are not claimed as separate findings here, but they should be reviewed for the same raw-read/normalized-write invariant.

Silently normalizing only the displayed name is not sufficient; it can continue to switch the underlying resource.

## Impact Details

The selected impact is **Critical impact `44`: Direct theft of user NFTs**.

ENS V2 names are ERC-1155 NFTs. This exploit makes the victim approve a standard ownership transfer for a token other than the name Portal read and displayed:

1. The attacker pays to create and gift the raw-NFD decoy. The victim takes no setup action.
2. The victim follows the normal ownership-transfer flow for that decoy and enters the attacker's visible recipient.
3. Portal authorizes using the decoy's raw owner but selects the victim's distinct NFC token.
4. One wallet confirmation executes successfully against the deployed registry semantics.
5. The attacker receives the valuable NFC NFT and its ownership roles.
6. The decoy remains with the victim, so the transaction does not even achieve the operation represented by the page.
7. The victim has no immediate unilateral way to recover the stolen NFC name.

The recipient is not the hidden substitution: the victim intends to send the apparent gifted name to that recipient. The hidden substitution is the NFT itself. This maps directly to theft of a user NFT, rather than merely to a wrong transaction label or generic Unicode confusion.

The same proved facts also satisfy two additional **Critical** application impacts. They are supporting classifications; impact `44` remains the selected primary impact:

| Critical impact                                                                                                                        | Exact mapping                                                                                                                                                                                                                       |
| -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `6285` — application displays or pre-fills an incorrect name or transaction detail, causing a user to submit an unintended transaction | Explorer says `Transfer name - cafe\u0301.eth`; the wallet request instead selects the distinct `café.eth` ERC-1155 token ID. The user approves a transaction for an NFT other than the name shown by the application.              |
| `6288` — malicious interaction with an already-connected wallet, including modifying transaction arguments or parameters               | Starting from the raw resource that passed the owner gate, Explorer's own helper normalizes only the transfer label and changes the `safeTransferFrom` `tokenId` argument to the canonical resource's ID before opening the wallet. |

These mappings matter because the listed impacts expressly contemplate a user approving the application-built transaction. User confirmation does not reveal or cure the hidden token substitution.

The exploit does not require leaked keys, a privileged role, a malicious resolver, a blanket approval, a compromised dependency, or a race. It uses an attacker-paid public registration and one ordinary Explorer transfer transaction.

## References

* [ENS competition scope, impacts, and Known Issues](https://immunefi.com/audit-competition/audit-competition-ens/scope/)
* [Scoped Portal asset at the frozen revision](https://github.com/immunefi-team/audit-comp-ens/tree/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal)
* [`/$name` route](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name.tsx#L22-L68)
* [transfer route](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/ownership/transfer.tsx#L21-L28)
* [`getLabel`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/utils/token/getLabel.ts#L1-L20)
* [`useCanTransferName`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useCanTransferName.ts#L49-L75)
* [`useTransferName`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferName.ts#L106-L181)
* [`useTransferDetachTargets`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts#L51-L100)
* [`SendNameForm`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/components/SendNameForm.tsx#L63-L101)
* [`transferToken`](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/transfer/helpers/transferToken.ts#L46-L92)
* [Sibling overview's `isValidEnsName` guard](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/routes/$name/index.tsx#L438-L460)
* [Dashboard V2 owner-domain query](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/dashboard/hooks/useV2NamesWithRolesForAddress.ts#L32-L82)
* [`NameList` raw name rendering](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/features/dashboard/components/NameList.tsx#L23-L33)
* [`EntityBadge` name route](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/components/EntityBadge.tsx#L259-L263)
* [`ProfileSidebar` ownership route](https://github.com/immunefi-team/audit-comp-ens/blob/cda79acaad59711b943fc68207ebb3f1d0ff8596/apps/portal/src/components/ProfileSidebar.tsx#L75-L82)
* [Pinned ENSjs revision `e96662c`](https://github.com/ensdomains/ensjs/commit/e96662c)
* [Official contracts-v2 revision `f21e4b4a38c949a6fc5c7d3bdd1b418292c880f1`](https://github.com/ensdomains/contracts-v2/commit/f21e4b4a38c949a6fc5c7d3bdd1b418292c880f1)
* [ENSjs PR #361](https://github.com/ensdomains/ensjs/pull/361) — read-only availability prior art, discussed above

## Link to Proof of Concept

<https://gist.github.com/IronsideSec/f437d137f3b73014d316799e7996d508>

## Proof of Concept

## Proof of Concept

The PoC has three independent layers:

1. a mounted Explorer test that runs the frozen route, transfer hooks, form, transaction modal, transaction manager, and ERC-1155 encoder and captures the exact wallet request;
2. a six-test official-contract suite that creates the primary and additional collision pairs through the real registrar, executes the requests, checks ownership/roles, and proves the victim cannot recover unilaterally; and
3. a deployed-Sepolia fork test at block `11,672,206` that provisions the primary collision class through the deployed public registrar and executes the Explorer-shaped request against the deployed registry.

All execution is local. No deployed ENS service is mutated and no customer is contacted. The recorded runs used Node `v22.19.0`, pnpm `10.27.0`, and Forge `1.5.1-stable`.

**Complete executable PoC and raw evidence (secret/unlisted Gist): `https://gist.github.com/IronsideSec/f437d137f3b73014d316799e7996d508`**

The linked Gist was fetched and validated as an unlisted, non-truncated Markdown Gist. It contains all source/config files, full transcripts, browser-render evidence, two deployment-binding records, and `SHA256SUMS`. GitHub “secret” Gists are unlisted rather than access-controlled; anyone with the submitted URL can read it.

| Gist file                          | SHA-256                                                            |
| ---------------------------------- | ------------------------------------------------------------------ |
| `vitest.config.ts`                 | `33de1ac09d3fa5813a2883f1861253cbf8fdb647d7c4d09a5cafb3eed5bbd010` |
| `portal-lead025.test.tsx`          | `338399e5103cad872129bb68972ad3b35882bbdc6c8564805675afe4bb5744c6` |
| `UnicodeTransferCollision.t.sol`   | `d8a0efd42f9888ee02afd75e083130a8403f886d258e4da46a8500bcd0782d7e` |
| `Lead025DeployedSepoliaFork.t.sol` | `886712b4850e9a2ad775f9960545a766f7cfe8d2ff8eda731ff46026438586a4` |
| `SHA256SUMS` manifest              | `b071187888e1863890dff1601591fb238fdb1369289a5350c186c6c3813162d5` |

The mounted test replaces network and wallet boundaries, not the vulnerable application code. Each load-bearing mocked read is independently bound to the fork:

| Mounted boundary                           | Value used by Explorer test                             | Deployed-fork assertion                                                                                      |
| ------------------------------------------ | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `getEnsOwnerQueryOptions(rawName)`         | victim owns raw NFD; protocol V2; deployed ETH registry | Universal Resolver `findOwner(rawNfdDns) == victim`; deployed registry owns raw token                        |
| `getNameRegistriesQueryOptions(rawName)`   | raw path reaches ETH registry then root                 | Universal Resolver `findRegistries(rawNfdDns)[1] == ETH_REGISTRY`                                            |
| `getHasRolesQueryOptions(normalizedLabel)` | victim can transfer normalized token                    | deployed registry holds target token for victim before transfer and accepts the exact `safeTransferFrom`     |
| `getEnsTokenId(normalizedLabel)`           | version-0 normalized token ID                           | deployed `ETH_REGISTRY.findTokenId(NFC_LABEL)` supplies the transferred ID                                   |
| `useNameResolverAddress(rawName)`          | no raw resolver                                         | deployed `ETH_REGISTRY.getResolver(NFD_LABEL) == address(0)`                                                 |
| `getEthAddress(rawName)`                   | no raw ETH address                                      | raw resolver is zero and deployed root `getResolver("eth") == address(0)`, so there is no inherited resolver |
| wallet `sendTransaction`                   | captures one request                                    | identical ABI call succeeds on the deployed fork and produces the asserted theft post-state                  |

The owner/registry reads and exact wallet execution are therefore not assumptions hidden by mocks. Gist file `Lead025DeployedSepoliaFork.t.sol` contains the complete deployed-fork test.

### A. Clone and install the scoped repository

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
cd audit-comp-ens
git checkout cda79acaad59711b943fc68207ebb3f1d0ff8596
pnpm install --frozen-lockfile
mkdir -p audit_workspace/pocs/lead-025-submission
```

Copy the complete Gist code blocks with these filenames and save them as:

```
audit_workspace/pocs/lead-025-submission/vitest.config.ts
audit_workspace/pocs/lead-025-submission/portal-lead025.test.tsx
```

Run:

```bash
pnpm --filter portal exec vitest run \
  --config ../../audit_workspace/pocs/lead-025-submission/vitest.config.ts \
  ../../audit_workspace/pocs/lead-025-submission/portal-lead025.test.tsx \
  --reporter=verbose
```

Expected result:

```
Test Files  1 passed (1)
Tests       2 passed (2)

encodedPath: /cafe%CC%81.eth/ownership/transfer
rawNfdHash: 0xe7739d357b90d35469fb9a3a5da362861a02e02c559108dc70489172a29a2fb4
rawNfcHash: 0x9513447e2d376aacd434727887590dd448cda8f2d30c4ace903d31fe209f8ad8
walletRequest.to: 0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
decoded.functionName: safeTransferFrom
decoded.tokenId: 67428657253604193222251220450000365237691932998204349417799831168307670548480
victimOnchainApprovals: 1
```

The test controls only network/wallet boundaries and unrelated shell widgets. The vulnerable route, transfer form/hooks, modal, transaction manager, and encoder are production modules.

### B. Run the official-contract post-state test

In a separate directory:

```bash
git clone --recursive https://github.com/ensdomains/contracts-v2.git contracts-v2-f21
cd contracts-v2-f21
git checkout f21e4b4a38c949a6fc5c7d3bdd1b418292c880f1
git submodule update --init --recursive
mkdir -p contracts/test/audit
```

Copy the complete Gist block `UnicodeTransferCollision.t.sol` and save it as:

```
contracts/test/audit/UnicodeTransferCollision.t.sol
```

Run:

```bash
cd contracts
forge test --match-contract UnicodeTransferCollisionF21Test -vv
```

Expected result:

```
Ran 6 tests for test/audit/UnicodeTransferCollision.t.sol:UnicodeTransferCollisionF21Test
[PASS] testDeployedPricingBoundaryRejectsShortIllustrationOnly()
[PASS] testExactPortalSafeTransferRequestStealsNfcAndLeavesDisplayedNfd()
[PASS] testIgnoredVariationSelectorCollisionTransfersAsciiToken()
[PASS] testPureAsciiCaseFoldCollisionTransfersLowercaseToken()
[PASS] testRawNfdAndNfcCoexistThroughOfficialRegistrarAndRegistry()
[PASS] testVictimCannotRecoverUnilaterallyButAttackerCanReturnName()
Suite result: ok. 6 passed; 0 failed; 0 skipped
```

The logged token IDs and calldata match the mounted Explorer test byte-for-byte.

### C. Run the deployed-Sepolia fork test

Still inside the official `contracts-v2-f21/contracts` directory, copy the complete Gist block `Lead025DeployedSepoliaFork.t.sol` and save it as:

```
test/audit/Lead025DeployedSepoliaFork.t.sol
```

Then run against the immutable historical block (a different Sepolia archive RPC may be substituted):

```bash
forge test \
  --match-contract Lead025DeployedSepoliaForkTest \
  --fork-url https://sepolia.gateway.tenderly.co \
  --fork-block-number 11672206 \
  -vv
```

Expected result:

```
Ran 1 test for test/audit/Lead025DeployedSepoliaFork.t.sol:Lead025DeployedSepoliaForkTest
[PASS] testDeployedRawReadThenPortalNfcTransferPostStateAndRecovery()
POST_THEFT victim NFC balance 0
POST_THEFT attacker NFC balance 1
POST_THEFT NFC owner 0x2222222222222222222222222222222222222222
POST_THEFT NFD decoy owner 0x1111111111111111111111111111111111111111
POST_THEFT victim transfer role false
POST_THEFT attacker transfer role true
deployed root resolver for eth 0x0000000000000000000000000000000000000000
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The fork provisions the suffixed pair described under **Deployment binding** because both labels were available at this fixed block. It calls only deployed contracts inside a local fork; it sends no public transaction.
