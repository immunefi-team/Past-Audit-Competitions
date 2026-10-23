# 92993 w a critical explorer send name checks the name in the url and signs away the owner s real name

**Submitted on Sep 13th 2026 at 18:17:22 UTC by @AlexSR for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92993
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow
  * Direct theft of user NFTs

## Description

## Brief/Intro

The Explorer's Send name flow at `/$name/ownership/transfer` takes the URL segment as typed and canonicalises it two different ways on one screen. The reads that identify the name, the gate that authorises the screen, and every name the screen prints use the segment as typed: the owner read, the registry read, the ownership gate `isAddressEqual(address, data.owner)`, and the three step labels in the transaction modal. Everything the wallet is asked to sign, and the two role checks that permit it, use `getLabel(name)`, which is the ENSIP-15 normalised first label: the token id, `setResolver`, `setSubregistry` and `safeTransferFrom` are all keyed on `labelhash(getLabel(name))`. For a segment that is not already normalised those are two different names in the same registry. `Alice.eth` and `alice.eth` are two labelhashes, two registry entries and two ERC-1155 tokens, and the registrar accepts both. An attacker registers the label `Alice`, sends that token to the owner of `alice.eth` (an unsolicited ERC-1155 transfer, which an externally owned account cannot refuse), and sends them a link to its transfer page. The victim owns `Alice.eth`, so the page renders; they hold the registration roles on `alice`, so both permission checks pass; they enter a recipient, leave the three options at their defaults, press Transfer name and approve three prompts. The wallet signs, in order, `setResolver(alice, 0x0)`, `setSubregistry(alice, 0x0)` and `safeTransferFrom(victim, recipient, tokenId(alice), 1, 0x)`. Every line on the screen reads `Alice.eth`. `Alice.eth` never leaves the wallet, and the flow returns the victim to the lookalike's ownership page.

The target is a `.eth` name registered under ENSv2 and held in an ordinary wallet by an owner who holds the registration roles on its label, which the registrar grants to whatever address the registration names as the owner, and a self-registration names the registrant (`ETHRegistrar.sol:17-23`, `:125`, `:151-158`); the attacker's cost is one registration of a label that ENSIP-15 folds onto the victim's, and a case change is enough. The victim's actions are the app's own Send name workflow with the app's own defaults, on a page reached from a first-party link. When the recipient is the attacker, the victim's name is stolen; whoever the recipient is, the name is moved with its resolver and its subregistry cleared, and it was never named on the screen that moved it. The same split is reached with no attacker, by a user who holds both a non-normalised name and its canonical twin and sends the former.

The defect is in the listed asset: one application computes two canonical forms of one route parameter and compares them nowhere. Every read that identifies the name, gates the screen or builds the calldata is a contract read through the Universal Resolver or the registry; none is the indexer, the subgraph or the metadata service. The consequence lands in the ETH registry, which is where a name lives and where the programme's own Critical entry for transfer flows places it. This report files at Critical on that entry, with the direct-theft-of-NFTs entry for the attacker-recipient case. The Proof of Concept section names the two offline tests that measure the rendered form and the decoded calldata, and carries the patch that closes the route.

## Vulnerability Details

### Root cause

One route parameter reaches the chain under two canonical forms, and the two halves of the screen never compare them.

Nothing on the path normalises the segment. The layout route `createFileRoute('/$name')` declares no parameter parser (`apps/portal/src/routes/$name.tsx:22-59`). Its `beforeLoad` calls `isValidEnsName(name)` once, inside the conjunction `isValidEnsName(name) && isTLD(name)` that decides a redirect for top-level names (`:29`), and otherwise hands the raw segment to the owner query (`:37-39`). The transfer route declares neither `parse` nor `validateSearch` (`routes/$name/ownership/transfer.tsx:16-19`) and reads `Route.useParams()` as typed (`:22`). The router is created with no parameter option (`main.tsx:32-44`). The app's own validator rejects this input: `isValidEnsName('Alice.eth')` is `false`, because `ens_normalize('Alice')` is not `'Alice'` (`utils/token/isNormalized.ts:16-22`, `:39-57`). The sibling page `/$name/` runs that validator and renders "Invalid name" when it fails (`routes/$name/index.tsx:441-442`, `:457-459`). The transfer route and the Ownership page that links to it import no validator at all.

The raw half is what identifies the name, to the user and to the ownership gate:

| What                              | Site                                                                                                                                                                                                                                                                                                                                                                           | Name used |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------- |
| Owner and registry of the name    | `transfer.tsx:27` -> `features/profile/hooks/useEnsOwner.ts:49-50` -> `utils/ens/resolveEnsOwner.ts:48-51`, which calls ensjs `getOwner` and `getNameRegistries`; both encode the name with `packetToBytes` (`@ensdomains/ensjs`, `dist/actions/public/v2/getOwner.js:24`, `getNameRegistries.js:25`)                                                                          | as typed  |
| The gate that renders the form    | `transfer.tsx:102-104`, `isAddressEqual(address, data.owner)` on that owner                                                                                                                                                                                                                                                                                                    | as typed  |
| Whether a resolver is attached    | `features/transfer/hooks/useTransferDetachTargets.ts:63` -> `features/records/hooks/useNameResolverAddress.ts:48-51` -> ensjs `getResolver`, `packetToBytes(name)` (`dist/actions/public/getResolver.js:44,52`)                                                                                                                                                                | as typed  |
| Whether a subregistry is attached | `useTransferDetachTargets.ts:64`, `:76-79`, the same `getNameRegistries` read                                                                                                                                                                                                                                                                                                  | as typed  |
| Every name on the screen          | `features/transfer/hooks/useTransferName.ts:177`, `transactionName` is the step label followed by `name`, rendered by the modal (`features/transaction-manager/components/TransactionStateContent.tsx:113`, `TransactionInfoContent.tsx:43`); the helper descriptions at `helpers/transferToken.ts:88`, `helpers/detachNameResolver.ts:91`, `helpers/detachNameRegistry.ts:92` | as typed  |

`packetToBytes` is a byte-level DNS encoder: it splits on dots and copies each label's UTF-8 bytes (`viem` 2.52.2, `utils/ens/packetToBytes.ts:30-52`, `stringToBytes(list[i])` at `:40`). It folds nothing.

The normalised half is everything the wallet is asked to sign, and the two role checks that decide whether it may be:

| What                                  | Site                                                                                                                                                                                                                         | Name used        |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| Transfer-role check                   | `features/transfer/hooks/useCanTransferName.ts:57-63` computes `getLabel(name)`; `:67-75` queries `hasRoles` with that label; ensjs hashes it, `BigInt(labelhash(params.label))` (`dist/actions/public/v2/hasRoles.js:70`)   | `getLabel(name)` |
| Detach-role check                     | `useTransferDetachTargets.ts:58-61`, `:67-74`; ensjs `roles(BigInt(labelhash(label)), account)` (`getNameRolesForAccount.js:36`)                                                                                             | `getLabel(name)` |
| Token id, read before the modal opens | `useTransferName.ts:112-115` -> `features/profile/hooks/useTokenId.ts:27-28` -> ensjs `getTokenId(BigInt(labelhash(label)))` (`getTokenId.js:21,26`)                                                                         | `getLabel(name)` |
| `setResolver`                         | `useTransferName.ts:132`, `:137` -> `detachNameResolver.ts:65-69` -> ensjs `args: [BigInt(labelhash(label)), resolverAddress]` (`dist/actions/wallet/v2/setResolver.js:9,14`)                                                | `getLabel(name)` |
| `setSubregistry`                      | `useTransferName.ts:140` -> `detachNameRegistry.ts:66-70` -> ensjs `setSubregistry.js:9,14`                                                                                                                                  | `getLabel(name)` |
| `safeTransferFrom`                    | `useTransferName.ts:143` -> `transferToken.ts:67-71`, whose sender argument is the connected wallet's account, which the flow binds to the owner (`useTransferName.ts:80`), followed by the recipient and the token id above | `getLabel(name)` |

`getLabel` runs `ens_normalize` on the whole name, `ens_split`s the result and returns the first label's code points (`utils/token/getLabel.ts:10-20`). The project's own test asserts the fold: `expect(getLabel('NICK.eth')).toBe('nick')` (`utils/token/getLabel.test.ts:26-29`). For a non-bracket label, `labelhash` is `keccak256(stringToBytes(label))`, and its docstring tells callers to normalise first (`viem`, `utils/ens/labelhash.ts:20-33`); the app does, on this half only.

Measured on the normaliser the Explorer resolves (`@adraffy/ens-normalize` 1.11.1) and on `keccak256` of the UTF-8 bytes, which is `labelhash`:

| URL segment                                                              | `getLabel` | `isValidEnsName` | `labelhash` of the label as typed                                    | `labelhash` of `getLabel` |
| ------------------------------------------------------------------------ | ---------- | ---------------- | -------------------------------------------------------------------- | ------------------------- |
| `alice.eth`                                                              | `alice`    | true             | `0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501` | the same                  |
| `Alice.eth`                                                              | `alice`    | false            | `0x81376b9868b292a46a1c486d344e427a3088657fda629b5f4a647822d329cd6a` | `0x9c02...0501`           |
| `ALICE.eth`                                                              | `alice`    | false            | `0xab1c679b95f02899dcb97b3404e8ac388504a2e78a30d99d5502d12353e2f8f7` | `0x9c02...0501`           |
| `alice.eth` with U+FF41 (fullwidth a) in place of the first `a`          | `alice`    | false            | `0x3832fd7bc0b177e8d63364dd2fd82ec44f7902bc326cdff5449e7b131e342bc0` | `0x9c02...0501`           |
| `alice.eth` with U+1D41A (mathematical bold a) in place of the first `a` | `alice`    | false            | `0x671898cb0bf313984300f4a13442a855b33abef12a868a45331f01b793a55b9e` | `0x9c02...0501`           |
| `alice` followed by U+FE0F (variation selector)                          | `alice`    | false            | `0xb9c6c55703a54275789b6ecd2ccd0509c9b686f3cabc8d425d10dd606205542e` | `0x9c02...0501`           |

Nothing between the halves compares them. `runStep` computes `label = getLabel(name)` and passes `name` and `label` side by side into the same helper calls (`useTransferName.ts:130-146`); each helper uses `label` for the calldata and `name` for its description string, and neither `useTransferName`, `useTransferDetachTargets`, `useCanTransferName` nor `SendNameForm` checks that `getLabel(name)` is the label of `name`.

The two names are two entries in one contract. In the registry, `LibLabel.id(label)` is `uint256(keccak256(bytes(label)))` (`LibLabel.sol:7-9`), the same value as `labelhash`; `_entry(anyId)` zeroes the version bits and returns the storage entry for that labelhash (`PermissionedRegistry.sol:593-595`, and the header at `:25-27`); `getTokenId` (`:306-308`), `setSubregistry` (`:142-147`) and `setResolver` (`:150-155`) each resolve their `anyId` through it. So `setResolver(labelhash('alice'), 0x0)` writes the `alice` entry and `safeTransferFrom` with `getTokenId(labelhash('alice'))` moves the `alice` token, whatever the page's URL said. The registry keeps the two apart: `_register` reverts `LabelAlreadyRegistered` only for the same label bytes while that entry is unexpired and owned (`:435-437`), and the rent price oracle the registrar prices through applies one length rule and states in its own docstring "Does not check if normalized." (`StandardRentPriceOracle.sol:272-277`, the length check at `:365-368`). `PermissionedRegistry.sol` and `StandardRentPriceOracle.sol` are byte-identical to the verified source of the two Sepolia deployments the Explorer's pinned client library targets, the registry and the rent price oracle (References 15 and 16); `LibLabel.sol` and `ETHRegistrar.sol` are quoted from the same branch of the same repository.

The registry the form is handed is the same for both names. `registryAddress` is `registries[1]` of the name as typed, the registry that contains the leaf label (`resolveEnsOwner.ts:55-59`); for any second-level `.eth` name that is the `.eth` registry, which also holds the canonical name's token. The route refuses subnames (`transfer.tsx:25`, `:71-92`), so on every path the flow reaches, the two names are two token ids in one contract and nothing reverts on a registry mismatch.

### Why every check passes for the wrong name

The ownership gate (`transfer.tsx:104`) requires the connected wallet to own the node as typed, so the attack needs the victim to hold the lookalike. A registration that names the victim as the owner does that, and so does one transfer (Trigger step 3). On the delivery path the attacker registered `Alice` naming themselves, so they hold `ROLE_CAN_TRANSFER_ADMIN` on it: `REGISTRATION_ROLE_BITMAP` is `ROLE_SET_SUBREGISTRY`, `ROLE_SET_SUBREGISTRY_ADMIN`, `ROLE_SET_RESOLVER`, `ROLE_SET_RESOLVER_ADMIN` and `ROLE_CAN_TRANSFER_ADMIN` (`ETHRegistrar.sol:17-23`), passed to the registry's `register` at `:151-158`, and that is the role `_update` checks on `from` (`PermissionedRegistry.sol:491`). The recipient is not consulted: an externally owned account has no `onERC1155Received`, and the standard invokes that hook for contract recipients only.

The transfer-role check (`useCanTransferName.ts:67-75`) then asks whether the victim holds `ROLE_CAN_TRANSFER_ADMIN` on the label `alice`. They own `alice.eth` and registered it through the registrar, so they do. The check the route added to keep a transfer from reverting is the first read that already points at the wrong name, and it passes for exactly that reason. Its docblock says the ensjs read "canonicalises to the same resource the contract checks" (`:21-23`); that is true of the label it hashes and says nothing about whether that label belongs to the name the route authorised.

The detach-role check (`useTransferDetachTargets.ts:67-74`, consumed at `:86-95`) asks whether the victim holds `ROLE_SET_RESOLVER` and `ROLE_SET_SUBREGISTRY` on `alice`. Both are in the registration bitmap above, so both options are permitted.

Option visibility is read on the lookalike: is there a resolver, is there a subregistry (`:63-64`, `:76-79`). The attacker sets both on `Alice` before sending it; the registrar's `register` takes a resolver and a subregistry as arguments (`ETHRegistrar.sol:151-158`). Both options then render, and they render on: the form initialises all three to `true` (`features/transfer/components/SendNameForm.tsx:69-73`), `effectiveOptions` keeps a visible option's value (`:92-96`), and `canStart` needs only a valid recipient and four successful lookups (`:100-101`; `useTransferDetachTargets.ts:97-101`). If the attacker sets nothing on the lookalike, no option renders, `buildTransferPlan` returns `['transfer-token']` alone (`features/transfer/utils/buildTransferPlan.ts:40`), and the victim still signs `safeTransferFrom` against the canonical token. Configuring the lookalike adds the two clears; it is not needed for the transfer.

One read on the visibility path does normalise: the `addr(60)` lookup behind the option that points the ETH address at the recipient goes through ensjs `getAddressRecord`, whose address coder hashes `normalize(name)` (`features/transfer/queries/getEthAddress.ts:13-16`; `@ensdomains/ensjs`, `dist/utils/coders/getAddress.js:3`, `:33-35`). It has no bearing on the outcome: that option's step is dropped from the plan whenever the resolver is being detached (`buildTransferPlan.ts:28-30`), which is the default.

### What the wallet signs, and what the screen shows

With the defaults, `buildTransferPlan` (`buildTransferPlan.ts:22-43`) yields `detach-resolver`, `detach-registry`, `transfer-token`, in that order, and the file's header says why the two writes precede the token move (`:1-5`). The modal lists them as "Detach resolver - Alice.eth", "Detach registry - Alice.eth", "Transfer name - Alice.eth" (`useTransferName.ts:174-181`, labels at `buildTransferPlan.ts:45-50`). The wallet receives, against the `.eth` registry:

1. `setResolver(labelhash('alice'), 0x0000000000000000000000000000000000000000)`: `alice.eth` stops resolving.
2. `setSubregistry(labelhash('alice'), 0x0000000000000000000000000000000000000000)`: every subname under its registry stops resolving.
3. `safeTransferFrom(victim, recipient, getTokenId(labelhash('alice')), 1, 0x)`: `alice.eth` leaves the wallet.

`finishFlow` then navigates to `/Alice.eth/ownership` (`useTransferName.ts:103`), the ownership page of the lookalike and not of the name that moved.

### Trigger and preconditions

The victim owns a `.eth` name registered under ENSv2 in an ordinary wallet and holds the registration roles on its label, `alice.eth` on Sepolia, the network the Explorer targets. The page renders only for a name in that state: it refuses anything the owner read on the URL segment does not report as ENSv2 (`transfer.tsx:57-70`), and the form appears only when `hasRoles` returns true for the connected owner on `getLabel(name)` (`transfer.tsx:150-154`, `:193-197`; `useCanTransferName.ts:57-75`), which the registrar's own registration bitmap grants to the address the registration named as the owner. They follow a link to a transfer page and complete the Send name workflow as designed: connect, enter a recipient, leave the options, press Transfer name, approve the prompts.

The attacker needs one registration, and a transfer only if that registration does not name the victim as the owner.

1. **Register `Alice`.** The registrar registers by commit and reveal (`ETHRegistrar.sol:25-29`) and prices through the oracle quoted above, which accepts the label. Neither application submits an uppercase label: the Explorer's register page rejects a name that is not normalised (`utils/token/nameValidation.ts:25-27`, rendered at `features/register/components/RegisterName.tsx:42-44`, `:84-91`), and the Manager lowercases before it validates (`apps/manager/src/features/register-v2/utils/name-parser.ts:46`). This label is registered by calling the registrar directly. A lookalike whose only difference is a character that `toLowerCase` leaves alone and ENSIP-15 maps, U+FF41, U+1D41A or a trailing U+FE0F as in the table above, is carried to the registrar by the Manager's own register route, whose only character rules are an ASCII punctuation blacklist plus whitespace and consecutive dots (`name-parser.ts:11`, `:49-56`, `:73-74`); the parsed label reaches the registration UI (`apps/manager/src/routes/register/$name.tsx:24`, `:57`) and is encoded into `register` with no ENSIP-15 step (`packages/transaction-manager/src/machines/registration/registration.actors.ts:242-244`). That registration step needs nothing outside the listed assets: the Manager app and the transaction-manager package are two of the five.
2. **Optionally attach a resolver and a subregistry to `Alice`**, in the same `register` call, so that both detach options render checked.
3. **Put it in the victim's wallet.** `register` takes the owner as an argument (`ETHRegistrar.sol:123-131`) and forwards it to the registry (`:151-158`), which mints the token to that owner and grants it the registration roles (`PermissionedRegistry.sol:461-469`), so a direct registrar call names the victim in step 1 and this step disappears. Registered through the Manager instead, the owner argument of the submitted `register` is `context.ownerAddress ?? context.accountAddress`: the Manager derives that value from the connected account context and sends it with the start event (`apps/manager/src/features/register-v2/state/registrationUi.machine.ts:482-483`, `:545-556`), the machine assigns it on `START_REGISTRATION` (`packages/transaction-manager/src/machines/registration/registration.machine.ts:551-552`) and hands it to the submitted registration (`:1382-1393`, the owner at `:1392`), whose actor passes it to `encodeRegisterCall` (`registration.actors.ts:924-930`), which places it as `register`'s second argument (`:245`). The register route takes no owner from the user, so delivery is a separate `safeTransferFrom(attacker, victim, tokenId(Alice), 1, 0x)` on the `.eth` registry.
4. **Send the link.** `/Alice.eth/ownership/transfer`, or `/Alice.eth/ownership`, whose Transfer button renders for the connected owner of the node as typed who holds the transfer role on the label `alice` (`routes/$name/ownership/index.tsx:49-53`, `:58-63`, `:102`, `:114-119`). A message offering to buy `Alice.eth` back, with the attacker's address as the recipient, is one pretext; the mechanism does not depend on it.

The reads that identify the name, gate the screen and build the calldata are contract reads: `findOwner`, `findRegistries` and `findResolver` on the Universal Resolver, `addr` resolved through it against the name's own resolver, and `hasRoles`, `roles` and `getTokenId` on the registry. The one identifier on the flow that names the indexer, `pollForIndexerSync` (`useTransferName.ts:102`), is a sleep-and-retry wrapper around the caller's own query invalidation (`utils/query/pollForIndexerSync.ts:50-73`), runs after the three transactions, and issues no query of its own.

The split is also reached with no attacker. A user who holds both a non-normalised name, registered through the Manager as in step 1, and its canonical twin, and sends the former from the Explorer, moves the latter. Same root cause, ordinary use.

Who is immune: a name held by a contract wallet that does not accept ERC-1155 tokens cannot be sent the lookalike.

### Recommended fix

Refuse a segment that is not already normalised before any child route reads it. The layout already imports `isValidEnsName` and already calls it in `beforeLoad`; make the call a gate:

```ts
    if (!isValidEnsName(name)) {
      throw notFound()
    }
```

at `routes/$name.tsx:27`, ahead of the TLD redirect. `throw notFound()` inside `beforeLoad` with a route-level `notFoundComponent` is the pattern `/registry/$address` already uses for a malformed address (`routes/registry/$address.tsx:12-17`), and `isValidEnsName` accepts encoded-labelhash labels (`isNormalized.ts:11-14`, `:45-47`), so names the app renders in bracket form still load. With it, the layout refuses the segment before any child route reads it, so the transfer route never mounts and nothing reaches the wallet; `/alice.eth/ownership/transfer` is unchanged. The layout already declares its own `notFoundComponent` (`$name.tsx:24`), which is what the router renders in the child's place (References 18). The patch is in the Proof of Concept section.

The structural fix is a `params.parse` on `/$name` that canonicalises the segment once, so every child route receives the ENSIP-15 form and the two halves cannot be handed different spellings. The alternative of deriving the owner and registry reads from `getLabel(name)` as well would also converge the halves, and is the worse change: it would point the page at a name the URL did not name.

## Impact Details

### The in-scope impact this maps to

The governing entry, quoted verbatim from the programme's Impacts in Scope list:

> Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

This report files at **Critical** on that entry's own sentence. The flow is a transfer flow; the name it acts on is one the page never named; it clears that name's resolver and its subregistry and then moves it. That is a name hijacked during a transfer flow. Both pointers are cleared before the token moves, so the name and every subname under its registry stop resolving, and the victim is left with no token for `alice` to move back.

The entry names transfer as one of its three flows, and the Explorer is the only listed application that ships one. `apps/manager/src/features` has eighteen feature directories and none is a transfer; the only file under `apps/manager` whose path contains "transfer" is a notification template, `features/notifications/notifications/name-transferred.tsx`; `apps/manager/src/routes` has no transfer route; and `apps/portal/src/features/transfer` is the only transfer feature in the repository. Read as naming the Manager application, the entry's third flow would name nothing. Read as naming the family of name-lifecycle flows, it names this one.

A second entry reaches the attacker-recipient case directly and carries the same severity:

> Direct theft of user NFTs

`safeTransferFrom(victim, attacker, tokenId(alice), 1, 0x)`, signed by the victim, is that.

A third entry describes what the connected wallet is handed:

> Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

The identifier argument of each of the three transactions resolves to the `alice` entry, a registration that appears nowhere on the screen that produced them: the two setters take `labelhash('alice')` in their `anyId` parameter (`PermissionedRegistry.sol:142`, `:150`), and `safeTransferFrom` takes `getTokenId(labelhash('alice'))`. That entry's head clause can be read as requiring an attacker at the wallet boundary; here the attacker supplies a link and the application supplies the argument.

### Loss mechanism and its bound

The victim loses the name: the ERC-1155 token for `alice` in the ETH registry, moved to the recipient, with its resolver pointer and its subregistry pointer cleared before it moved, so the name and every subname under its registry stop resolving before the recipient holds it. After the transfer the victim holds no token for `alice` and cannot sign a transfer of one they do not hold; `register` reverts `LabelAlreadyRegistered` while the entry is unexpired and owned (`PermissionedRegistry.sol:435-437`); and the registry declares no reclaim, restore, recover or rescue function. While the entry stands, the victim has no unilateral path back.

Per victim the loss is one name, and a name's value is a property of the name, so no figure is offered. The mechanism repeats per target at the cost of one registration each, plus a delivery transfer where the registration did not name the victim as the owner. The window is open from the moment the lookalike lands in the victim's wallet: nothing about the lookalike changes, and nothing on the transfer path distinguishes it from the victim's own name. The profile page under the same layout does refuse the segment (`routes/$name/index.tsx:441-442`, `:457-459`); the ownership page and the transfer page do not, and that asymmetry is the finding.

What the impact is not. `Alice.eth` is untouched throughout and stays in the victim's wallet. The three transactions carry no ETH value; what moves is the name itself, the ERC-1155 token in the ETH registry. The wallet is asked for exactly the three signatures the screen lists; what differs is the name they are for.

### Why not lower, and the out-of-scope entries to weigh

No High, Medium or Low entry describes a user-signed transfer of a name: those entries cover persistent and reflected static-content injection, disclosure of or changes to other users' details, open redirect, expired outgoing links and temporary denial of access. Critical is the top tier, so what could place this lower is an exclusion.

**"The ENS smart contracts, subgraph/indexer, and NFT metadata service are out of scope"** is the entry to weigh first, because the loss is written into the registry. Every read that identifies the name, gates the screen or builds the calldata is a contract read, and none is the indexer, the subgraph or the metadata service; the defect is two halves of a listed application disagreeing about which name one string denotes. What happens outside the two listed applications is the attacker's own setup, and it is permissionless calls the attacker makes from their own wallet: the commit-and-reveal registration of the lookalike, which the registrar takes as two transactions (`ETHRegistrar.sol:25-29`), and its delivery, which disappears when the registration names the victim as the owner, which `register` accepts as an argument and the registry honours by minting to that owner (Trigger step 3). Where the lookalike is registered through the Manager's own register route instead, which is available for a difference ENSIP-15 maps and `toLowerCase` does not, the registration moves inside a listed asset and delivery is a further call, `safeTransferFrom` on the registry, because the Manager fixes the owner to the account context's own address. None of these is a read either application performs, and the exclusion names assets rather than whether an attacker's own wallet touches a chain. The consequence has to land in the registry, because that is where a name is: the transfer entry's own first item names "the name, its ownership, registry, or resolver" as what a hijack assigns, and the already-connected-wallet entry names "setApprovalForAll on the registry or NameWrapper". A reading under which a registry-side consequence disqualifies a front-end finding leaves both entries with nothing to describe.

**"Impacts caused by vulnerabilities requiring un-prompted, in-app user actions that are not part of the normal app workflows"** does not apply. Opening a name's transfer page, entering a recipient, pressing Transfer name and approving the prompts is the Send name workflow, in the app's own UI, with the app's own defaults. The no-attacker case above needs nothing but that workflow.

**"UX and UI impacts that do not materially disrupt use of the platform"** does not apply. The defect is visible as a label on a screen; the outcome is an ERC-1155 token leaving the victim's wallet.

**"Impacts caused by attacks requiring access to privileged addresses (governance, strategist) except in such cases where the contracts are intended to have no privileged access to functions that make the attack possible"** does not apply. An unprivileged attacker, with one registration and at most one unsolicited token transfer, causes the owner of a `.eth` name to sign away that name from a screen that never named it. The only roles on the path are the registration roles the ETH registrar grants to the owner a registration names (`ETHRegistrar.sol:17-23`): the victim holds them on `alice`, the name they registered, and the attacker holds none there; and no governance, administrator or otherwise privileged address is used or needed at any step.

**"Third-party dependencies: unmodified third-party dependencies are out of scope"** does not apply. `@ensdomains/ensjs` and `viem` each behave as documented and are unmodified; they are cited only to show which string the application hands them, and each applies exactly the transformation the application asks of it. Both the omission and the split live in first-party files: `routes/$name.tsx:29` declines to gate on the validator it already imports, and `useTransferName.ts:130-146` hands `name` and `getLabel(name)` to the same helper call side by side. The patch changes one file inside the listed asset.

**"Attempting phishing or other social engineering attacks against our employees and/or customers"** is a testing prohibition. This report describes a link and sent none; nothing in it was performed against a person, a deployment or a public chain.

## Link to Proof of Concept

<https://gist.github.com/AlexQllx/3c7f3018edf5eb621f33dc51ce63b595>

## Proof of Concept

## Proof of Concept

Two test files, both offline, run on 2026-09-12 inside a loopback-only sandbox over a throwaway copy of the competition snapshot; the captured output of each run, unpatched and with the fix applied, is reproduced below. Runnable copies of both files, the fix patch, the complete capture (`CAPTURE_20260912.log`) and a `README.md` for placement and commands are to be published as a secret Gist before filing; its link and pinned revision go here: [GIST LINK](https://gist.github.com/AlexQllx/3c7f3018edf5eb621f33dc51ce63b595). Nothing below depends on opening it. Every function and component under test is production code.

| File                                                      | Enters at                                                                                                                                                  | Measures                                                                                                                                                                                                                           | Result                                                                                                                   |
| --------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| 1. `send_name_signs_for_the_normalised_twin.poc.test.tsx` | the shipped transfer route component mounted for `Alice.eth`, driven through the form's own controls; and the layout route's `beforeLoad`, called directly | which name each chain read was asked for, the three transactions the flow hands to the transaction manager (decoded), and every name the screen renders; the same run for `alice.eth` as the control; the layout gate the fix adds | 4 passed, 1 skipped on the unmodified snapshot; with the fix applied the one assertion written to flip fails, and 4 pass |
| 2. `get_label_folds_what_the_route_does_not.poc.test.ts`  | `getLabel`, `isValidEnsName`, `labelhash`, `namehash`, as an extension of the project's own `getLabel.test.ts`                                             | that `getLabel` folds every lookalike onto `alice`, that `isValidEnsName` rejects each, and that each is a different labelhash and node                                                                                            | 3 passed; unchanged with the fix applied                                                                                 |

File 1 measures what the application asks for, shows and signs. File 2 measures the fold that produces the split, on the functions the route uses.

### Where this is run, and what it never touches

Both files run in process under vitest with happy-dom, inside a copy of `apps/portal` from the competition snapshot, with the workspace's own `vitest.config.ts` and `src/test-utils/` providers. No chain, no wallet, no RPC endpoint and no signing: `fetch` is replaced with a recorder that refuses every request, and file 1 asserts the recorder saw nothing. Nothing on any public chain, mainnet or testnet, is read or written by any part of this proof of concept.

The runs were made with Node 20.19.2, vitest 4.1.5 and happy-dom 18.0.1, from `apps/portal/node_modules/.bin/vitest`, inside a bubblewrap sandbox with fresh user, network, pid and ipc namespaces, a cleared environment, the host filesystem mounted read-only and a throwaway overlay over the snapshot, so the frozen tree was never written. Before any test the sandbox checked its own isolation: no inherited file descriptor other than stdio pipes and `/dev/null`, the only interface is `lo`, no socket file is visible, and a tcp4, a tcp6 and a DNS probe each fail. That block, the hashes of the three Gist files as run, and the placement open the capture:

```
=== NAMESPACE IDENTITY ===
node: v20.19.2
=== FAILSAFE 0: no inherited fds except stdio pipes and /dev/null ===
fds clean: 0 1 2 3 
=== FAILSAFE 1: interfaces must be exactly lo ===
interfaces: lo 
=== FAILSAFE 2: no UNIX socket file visible anywhere ===
no socket files visible
=== FAILSAFE 3: egress probes (each must be blocked) ===
tcp4 1.1.1.1:443 -> blocked: ENETUNREACH
tcp6 [2606:4700:4700::1111]:443 -> blocked: ENETUNREACH
dns example.com -> blocked: EAI_AGAIN
=== SHA256 OF THE GIST FILES AS RUN ===
c36c2c2155a4bc1f7da46620547c83c4076df64e6b903859ff551a1954cfcc53  send_name_signs_for_the_normalised_twin.poc.test.tsx
fad164f5ffceb666ed28df94646c0d2ad111871e29a7fa5eee014afe96cdd5ab  get_label_folds_what_the_route_does_not.poc.test.ts
422967a79e924332234952ec4812108363768a23853a7d811145ab70f7f5427a  refuse_non_normalised_name_segments.patch
=== PLACEMENT (overlay only) ===
placed: apps/portal/src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx
placed: apps/portal/src/utils/token/get_label_folds_what_the_route_does_not.poc.test.ts
```

Every fenced block of captured output in this section is a contiguous run of lines from that capture, in order. The one alteration is the checkout's absolute path in the four `RUN v4.1.5` banner lines, written `<archive-root>`. No line of any run mentions an outbound request, and every render and gate test asserts the recorder saw none.

File 1's doubles, and what each replaces:

* `transactionManager.startTransaction` and `waitForTransaction` from `@ens-apps/transaction-manager` are recorders. That module is the app's own boundary between "what the flow decided to send" and "sending it": `detachNameResolver.ts:77-95`, `detachNameRegistry.ts:78-96` and `transferToken.ts:73-92` each build their request and hand it over there. The project's own `records/helpers/saveRecords.test.ts:7-15` replaces the same two symbols the same way. Everything above the recorder is production code: the route component, `useCanTransferName`, `useTransferDetachTargets`, `SendNameForm`, `useTransferName`, `buildTransferPlan`, the three helpers and the ensjs write-parameter builders they call.
* The chain reads are replaced at the ensjs boundary (`getOwner`, `getNameRegistries`, `getResolver`, `getAddressRecord`, `getName`, `hasRoles`, `getNameRolesForAccount`, `getTokenId`) with answers from one two-name table, and every replacement records the `name` or `label` it was asked for. The app's own query wrappers around them stay real. The table has the victim owning both `Alice.eth` and `alice.eth`, a resolver, a subregistry and an address record on the name, the registration bitmap for the victim on the label `alice`, and a distinct token id for `alice`. The project's own `transfer/helpers/setEthAddress.test.ts:25-27` mocks `@ensdomains/ensjs/public` the same way.
* `getWalletClient` returns a client literal for the victim; `@/lib/wagmi` is replaced with the project's own test chain so importing it does not build the WalletConnect connector, following `hooks/useContractAddress.test.ts:6-8`; `TransactionModal` and `useTransactionModal` are stubs, following `resolver/components/ChangeResolverForm.test.tsx:7-19`; the shipped modal is replaced because it estimates gas through the public client, and the strings file 1 reads out of the document are therefore rendered by the stub, which passes `transactionName` through exactly as the shipped modal does (`TransactionStateContent.tsx:113` and `TransactionInfoContent.tsx:43` each render `{transaction.transactionName}` with no transformation); `createFileRoute`, `useParams` and `useConnection` are stubs, following `routes/$name/registry.test.tsx:19-31`, so the route component is mounted directly and `notFound` is left real; `pollForIndexerSync` and the recipient's L2 primary-name lookup are stubbed.

The finding's authorisation frame is not skipped: mounting the transfer route component runs the ownership gate `isAddressEqual(address, data.owner)` on the raw node (`transfer.tsx:102-104`) and `useCanTransferName` on the normalised label (`:150-154`) itself. The one frame above it, the `/$name` layout's `beforeLoad`, does no authorisation on the unmodified snapshot; it is where the fix adds its guard, so it is exercised directly by file 1's third block rather than mounted, and the router fact that a `beforeLoad` throwing `notFound` renders the `notFoundComponent` in place of the child route is cited from the workspace's installed packages, not re-implemented: `@tanstack/router-core` 1.168.15, `dist/esm/load-matches.js:514-520` and `:543-565`, and `@tanstack/react-router` 1.168.23, `dist/esm/Match.js:73`, `:77` and `:93-97`.

### File 1: the route, rendered and driven

For `name = 'Alice.eth'`, the shipped route renders the Send name form (the raw-node owner read returns the victim and the canonical-label role read passes), all three options render checked, and pressing Transfer name records exactly three transactions. Each is addressed to the `.eth` registry and decodes to a write keyed on `labelhash('alice')` and the token id of `alice` - the victim's real name - while every rendered step names `Alice.eth`. The claim is a negative, so it is asserted over the whole recorded set:

```tsx
    // The load-bearing measurement: every write is keyed on `labelhash('alice')`,
    // the victim's REAL name, not on `labelhash('Alice')`, the name the URL and
    // the gate named. All three are addressed to the .eth registry.
    expect(calls[0]?.to).toBe(ETH_REGISTRY)
    expect(calls[0]?.args).toEqual([BigInt(labelhash(CANONICAL_LABEL)), zeroAddress])
    expect(calls[1]?.to).toBe(ETH_REGISTRY)
    expect(calls[1]?.args).toEqual([BigInt(labelhash(CANONICAL_LABEL)), zeroAddress])
    expect(calls[2]?.to).toBe(ETH_REGISTRY)
    const transferArgs = calls[2]?.args as readonly unknown[]
    expect(String(transferArgs[0]).toLowerCase()).toBe(h.VICTIM.toLowerCase())
    expect(String(transferArgs[1]).toLowerCase()).toBe(h.RECIPIENT.toLowerCase())
    expect(transferArgs[2]).toBe(h.TOKEN_ID_ALICE)
    expect(transferArgs[3]).toBe(1n)

    // The negative, over the whole recorded set: nothing carries the labelhash of
    // the name the URL named.
    for (const call of h.recorded) {
      expect(call.data.toLowerCase()).not.toContain(UPPER_HASH)
      expect(call.from.toLowerCase()).toBe(h.VICTIM.toLowerCase())
      expect(call.chainId).toBe(sepoliaWithEns.id)
    }

    // The rendered state: every step's on-screen name is the lookalike the URL
    // named; the canonical name the wallet signs for appears nowhere.
    const shown = document.body.textContent ?? ''
    expect(shown).toContain(`Detach resolver - ${LOOKALIKE}`)
    expect(shown).toContain(`Detach registry - ${LOOKALIKE}`)
    expect(shown).toContain(`Transfer name - ${LOOKALIKE}`)
    expect(shown).toContain(LOOKALIKE)
    expect(shown).not.toContain(CANONICAL)

    // Which name each read was asked for: the raw reads for the lookalike, the
    // signing reads for the canonical label.
    for (const name of h.ownerReads) expect(name).toBe(LOOKALIKE)
    for (const name of h.registriesReads) expect(name).toBe(LOOKALIKE)
    for (const name of h.resolverReads) expect(name).toBe(LOOKALIKE)
    expect(h.hasRoleReads.length).toBeGreaterThanOrEqual(1)
    for (const label of h.hasRoleReads) expect(label).toBe(CANONICAL_LABEL)
    for (const label of h.roleReads) expect(label).toBe(CANONICAL_LABEL)
    expect(h.tokenReads).toContain(CANONICAL_LABEL)
    expect(h.tokenReads).not.toContain('Alice')
```

The third argument, the token id, is a fixture value the mocked `getTokenId` returns for the label `alice`; what that line measures is which label the application asked for, and a mock keyed on `Alice` returns zero and fails it. The two `labelhash` arguments beside it are recomputable from the label.

The control, in the same file, runs the identical mount for `name = 'alice.eth'` and produces the same three writes on the same canonical label, so the divergence is the URL's canonicalisation and nothing else; there the screen names the name it signs for.

The layout gate the fix adds is exercised directly, entering at the exact function the patch edits. One assertion is marked FLIP: it passes on the unmodified snapshot (the layout admits the non-normalised segment) and fails with the patch applied (the layout refuses it). The patch is detected by reading `$name.tsx`'s own source, so the two arms need no flag:

```tsx
  it('the layout beforeLoad does not refuse the non-normalised segment', async () => {
    // FLIP. On the unmodified snapshot the layout admits `Alice.eth`, so every
    // child route reads the owner by it. With the patch this call throws
    // `notFound`, and this assertion fails; the patched behaviour is asserted in
    // the block below.
    const outcome = await runBeforeLoad(LOOKALIKE)
    expect(outcome.threw).toBe(false)
    expect(h.attemptedRequests).toEqual([])
  })

  it('the layout beforeLoad admits the canonical segment', async () => {
    const outcome = await runBeforeLoad(CANONICAL)
    expect(outcome.threw).toBe(false)
    expect(h.attemptedRequests).toEqual([])
  })
})

describe.skipIf(!PATCHED)('with the fix applied: the layout refuses the non-normalised segment', () => {
  it('throws notFound for the lookalike and admits the canonical name', async () => {
    const bad = await runBeforeLoad(LOOKALIKE)
    expect(bad.threw).toBe(true)
    expect(isNotFound(bad.error)).toBe(true)

    const good = await runBeforeLoad(CANONICAL)
    expect(good.threw).toBe(false)
    expect(h.attemptedRequests).toEqual([])
  })
})
```

The captured run of file 1 on the unmodified snapshot:

```
=== RUN 1/4: file 1, unpatched (apps/portal) ===

 RUN  v4.1.5 <archive-root>/apps/portal

stdout | src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the route authorises against the URL node and signs against a different one > renders the form for the raw segment, hands the wallet three writes keyed on the canonical label, and names the raw one at every step
[poc] url Alice.eth -> signed setResolver -> setSubregistry -> safeTransferFrom on label alice token 4660

 ✓ src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the route authorises against the URL node and signs against a different one > renders the form for the raw segment, hands the wallet three writes keyed on the canonical label, and names the raw one at every step 550ms
stdout | src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the route authorises against the URL node and signs against a different one > control: the canonical URL signs the same three writes, and here the screen names the name it signs
[poc] control url alice.eth -> signed on label alice (displayed name matches signed name)

 ✓ src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the route authorises against the URL node and signs against a different one > control: the canonical URL signs the same three writes, and here the screen names the name it signs 227ms
 ✓ src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the layout gate the fix adds > the layout beforeLoad does not refuse the non-normalised segment 1ms
 ✓ src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the layout gate the fix adds > the layout beforeLoad admits the canonical segment 1ms
 ↓ src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > with the fix applied: the layout refuses the non-normalised segment > throws notFound for the lookalike and admits the canonical name

 Test Files  1 passed (1)
      Tests  4 passed | 1 skipped (5)
   Start at  23:39:06
   Duration  4.10s (transform 867ms, setup 109ms, import 2.66s, tests 782ms, environment 297ms)

vitest exit code: 0
```

### File 2: the fold

An extension of the project's own `utils/token/getLabel.test.ts`, whose case `normalizes input via ens_normalize (case)` already asserts `getLabel('NICK.eth')` is `nick` (`:26-29`). It asserts that `getLabel` folds a plain ASCII case variant and three invisible or fullwidth variants onto `alice`, that `isValidEnsName` rejects each while accepting `alice.eth`, and that each variant's label and node as typed differ from the canonical ones while its folded label hashes to the canonical labelhash:

```ts
  it('each lookalike is a different labelhash and a different node than the canonical name', () => {
    const canonHash = labelhash(CANONICAL_LABEL)
    const canonNode = namehash(CANONICAL)
    for (const { note, name } of LOOKALIKES) {
      const rawLabel = name.slice(0, name.lastIndexOf('.'))
      const rawHash = labelhash(rawLabel)
      // The label as typed hashes to something other than the label the wallet
      // signs against.
      expect(rawHash).not.toBe(canonHash)
      // And its node (what the raw reads key on) differs from the canonical node.
      expect(namehash(name)).not.toBe(canonNode)
      // The folded label, however, hashes to exactly the canonical labelhash.
      expect(labelhash(getLabel(name))).toBe(canonHash)

      // biome-ignore lint/suspicious/noConsole: PoC output
      console.log(
        '[poc]', note,
        '| labelhash(as typed):', rawHash,
        '| labelhash(getLabel):', labelhash(getLabel(name)),
      )
    }
    // biome-ignore lint/suspicious/noConsole: PoC output
    console.log('[poc] labelhash(alice):', canonHash)
  })
```

The specific hash values are printed rather than asserted as literals; the assertions carry the claim (the fold, the rejection, the inequality), and the printed values cross-check against the Root cause table above.

The captured run of file 2 on the unmodified snapshot:

```
=== RUN 2/4: file 2, unpatched (apps/portal) ===

 RUN  v4.1.5 <archive-root>/apps/portal

 ✓ src/utils/token/get_label_folds_what_the_route_does_not.poc.test.ts > getLabel folds what the transfer route never checks > folds every lookalike onto the victim real label, which the route signs against 172ms
stdout | src/utils/token/get_label_folds_what_the_route_does_not.poc.test.ts > getLabel folds what the transfer route never checks > each lookalike is a different labelhash and a different node than the canonical name
[poc] ASCII case | labelhash(as typed): 0x81376b9868b292a46a1c486d344e427a3088657fda629b5f4a647822d329cd6a | labelhash(getLabel): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
[poc] ASCII upper | labelhash(as typed): 0xab1c679b95f02899dcb97b3404e8ac388504a2e78a30d99d5502d12353e2f8f7 | labelhash(getLabel): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
[poc] U+FF41 fullwidth a | labelhash(as typed): 0x3832fd7bc0b177e8d63364dd2fd82ec44f7902bc326cdff5449e7b131e342bc0 | labelhash(getLabel): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
[poc] U+1D41A math bold a | labelhash(as typed): 0x671898cb0bf313984300f4a13442a855b33abef12a868a45331f01b793a55b9e | labelhash(getLabel): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
[poc] trailing U+FE0F | labelhash(as typed): 0xb9c6c55703a54275789b6ecd2ccd0509c9b686f3cabc8d425d10dd606205542e | labelhash(getLabel): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
[poc] labelhash(alice): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501

 ✓ src/utils/token/get_label_folds_what_the_route_does_not.poc.test.ts > getLabel folds what the transfer route never checks > the app own validator rejects every lookalike, and accepts the canonical name 1ms
 ✓ src/utils/token/get_label_folds_what_the_route_does_not.poc.test.ts > getLabel folds what the transfer route never checks > each lookalike is a different labelhash and a different node than the canonical name 14ms

 Test Files  1 passed (1)
      Tests  3 passed (3)
   Start at  23:39:11
   Duration  1.03s (transform 60ms, setup 108ms, import 218ms, tests 189ms, environment 270ms)

vitest exit code: 0
```

### Attack steps: reproducing this by hand

Given `alice.eth` registered to a wallet `V` on Sepolia, an attacker wallet `A`, and a recipient `R` (which may be `A`):

1. As `A`, register the label `Alice` through the registrar's commit and register, naming any resolver and any subregistry so both detach options will render. The oracle accepts the label; the Explorer's and the Manager's own register pages would not submit it, so this is a direct call.
2. As `A`, `safeTransferFrom(A, V, getTokenId(labelhash('Alice')), 1, 0x)` on the `.eth` registry. `V` cannot refuse it.
3. As `V`, open `/Alice.eth/ownership/transfer` and connect. The page renders the Send name form: the owner read on `Alice.eth` returns `V`, and the transfer-role read on `alice` returns true.
4. Type `R`, leave the three options on, press Transfer name.
5. Approve "Detach resolver - Alice.eth". The wallet has signed `setResolver(labelhash('alice'), 0x0)`.
6. Approve "Detach registry - Alice.eth". The wallet has signed `setSubregistry(labelhash('alice'), 0x0)`.
7. Approve "Transfer name - Alice.eth". The wallet has signed `safeTransferFrom(V, R, getTokenId(labelhash('alice')), 1, 0x)`.
8. The page returns to `/Alice.eth/ownership`, which shows `V` as the owner of `Alice.eth`. `alice.eth` has no resolver, no subregistry, and belongs to `R`.

Step 3 is also reached from `/Alice.eth/ownership`, whose Transfer button renders for `V`.

### The counterfactual: the fix patch

The minimal change, and what it does to file 1. Save it as `refuse_non_normalised_name_segments.patch` and apply with `patch -p1` from the repository root.

```diff
Refuse a non-normalised name segment before any child route reads it.

Every route under `/$name/` reads the owner and the registry by the segment
as typed, and the transfer flow builds its calldata from `getLabel(name)`,
the ENSIP-15 form of the same string. For a segment that is not already
normalised those are two different names, so the page authorises one and
the wallet signs for the other. The layout already imports `isValidEnsName`
and already calls it in `beforeLoad`, but only inside the TLD-redirect
conjunction, where it rejects nothing. This makes the call a gate, the way
`/$name/` (`$name/index.tsx:442,458-459`) and `/registry/$address`
(`registry/$address.tsx:13-17`) already refuse a malformed segment.

`isValidEnsName` accepts encoded-labelhash labels (`isNormalized.ts:11-14`,
`:45-47`), so names the app renders in bracket form still load. The
redundant `isValidEnsName(name) &&` in the TLD condition is left as is.

Applies to apps/portal/src/routes/$name.tsx at the audited snapshot with
`patch -p1`; if line offsets have drifted, use `--fuzz=3` or apply by hand:
it is two edits.

--- a/apps/portal/src/routes/$name.tsx
+++ b/apps/portal/src/routes/$name.tsx
@@ -2,6 +2,7 @@
 import {
   createFileRoute,
+  notFound,
   Outlet,
   redirect,
   useParams,
 } from '@tanstack/react-router'
@@ -25,6 +26,15 @@
   beforeLoad: async ({ params }) => {
     const { name } = params
 
+    // One segment must name one node. The child routes read the owner by
+    // the segment as typed and the transfer flow signs against
+    // `getLabel(name)`, its ENSIP-15 form; a segment that is not already
+    // normalised would let those two halves address different names.
+    // Refuse it here, as `/$name/` does for its own page.
+    if (!isValidEnsName(name)) {
+      throw notFound()
+    }
+
     // Redirect valid TLDs to the dedicated /tld/$tld route
     if (isValidEnsName(name) && isTLD(name)) {
       const shouldRedirect =
```

The patch applies cleanly to the audited file with both `git apply --check -p1` and `patch -p1 --dry-run`. Under it, file 1's one FLIP assertion fails (the layout `beforeLoad` now refuses `Alice.eth`) and its `with the fix applied` block runs and passes (the layout throws `notFound` for the lookalike and admits the canonical name); the two render tests pass unchanged, because they enter one frame below the layout the patch guards, which is why the fix has to live in the layout. File 2 is unaffected, since the fold it measures is the correct behaviour of `getLabel`.

The patch applied inside the sandbox, then both files run again, followed by the check that nothing reached the frozen snapshot:

```
=== APPLY PATCH (overlay only) ===
patching file 'apps/portal/src/routes/$name.tsx'
patch exit code: 0
=== RUN 3/4: file 1, patched (apps/portal) ===

 RUN  v4.1.5 <archive-root>/apps/portal

stdout | src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the route authorises against the URL node and signs against a different one > renders the form for the raw segment, hands the wallet three writes keyed on the canonical label, and names the raw one at every step
[poc] url Alice.eth -> signed setResolver -> setSubregistry -> safeTransferFrom on label alice token 4660

 ✓ src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the route authorises against the URL node and signs against a different one > renders the form for the raw segment, hands the wallet three writes keyed on the canonical label, and names the raw one at every step 557ms
stdout | src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the route authorises against the URL node and signs against a different one > control: the canonical URL signs the same three writes, and here the screen names the name it signs
[poc] control url alice.eth -> signed on label alice (displayed name matches signed name)

 ✓ src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the route authorises against the URL node and signs against a different one > control: the canonical URL signs the same three writes, and here the screen names the name it signs 246ms
 × src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the layout gate the fix adds > the layout beforeLoad does not refuse the non-normalised segment 10ms
   → expected true to be false // Object.is equality
 ✓ src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the layout gate the fix adds > the layout beforeLoad admits the canonical segment 1ms
 ✓ src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > with the fix applied: the layout refuses the non-normalised segment > throws notFound for the lookalike and admits the canonical name 0ms

⎯⎯⎯⎯⎯⎯⎯ Failed Tests 1 ⎯⎯⎯⎯⎯⎯⎯

 FAIL  src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx > the layout gate the fix adds > the layout beforeLoad does not refuse the non-normalised segment
AssertionError: expected true to be false // Object.is equality

- Expected
+ Received

- false
+ true

 ❯ src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx:664:27
    662|     // the block below.
    663|     const outcome = await runBeforeLoad(LOOKALIKE)
    664|     expect(outcome.threw).toBe(false)
       |                           ^
    665|     expect(h.attemptedRequests).toEqual([])
    666|   })

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯


 Test Files  1 failed (1)
      Tests  1 failed | 4 passed (5)
   Start at  23:39:13
   Duration  4.08s (transform 857ms, setup 116ms, import 2.65s, tests 817ms, environment 267ms)

vitest exit code: 1
=== RUN 4/4: file 2, patched (apps/portal) ===

 RUN  v4.1.5 <archive-root>/apps/portal

 ✓ src/utils/token/get_label_folds_what_the_route_does_not.poc.test.ts > getLabel folds what the transfer route never checks > folds every lookalike onto the victim real label, which the route signs against 209ms
stdout | src/utils/token/get_label_folds_what_the_route_does_not.poc.test.ts > getLabel folds what the transfer route never checks > each lookalike is a different labelhash and a different node than the canonical name
[poc] ASCII case | labelhash(as typed): 0x81376b9868b292a46a1c486d344e427a3088657fda629b5f4a647822d329cd6a | labelhash(getLabel): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
[poc] ASCII upper | labelhash(as typed): 0xab1c679b95f02899dcb97b3404e8ac388504a2e78a30d99d5502d12353e2f8f7 | labelhash(getLabel): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
[poc] U+FF41 fullwidth a | labelhash(as typed): 0x3832fd7bc0b177e8d63364dd2fd82ec44f7902bc326cdff5449e7b131e342bc0 | labelhash(getLabel): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
[poc] U+1D41A math bold a | labelhash(as typed): 0x671898cb0bf313984300f4a13442a855b33abef12a868a45331f01b793a55b9e | labelhash(getLabel): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
[poc] trailing U+FE0F | labelhash(as typed): 0xb9c6c55703a54275789b6ecd2ccd0509c9b686f3cabc8d425d10dd606205542e | labelhash(getLabel): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
[poc] labelhash(alice): 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501

 ✓ src/utils/token/get_label_folds_what_the_route_does_not.poc.test.ts > getLabel folds what the transfer route never checks > the app own validator rejects every lookalike, and accepts the canonical name 1ms
 ✓ src/utils/token/get_label_folds_what_the_route_does_not.poc.test.ts > getLabel folds what the transfer route never checks > each lookalike is a different labelhash and a different node than the canonical name 13ms

 Test Files  1 passed (1)
      Tests  3 passed (3)
   Start at  23:39:17
   Duration  1.11s (transform 65ms, setup 107ms, import 235ms, tests 225ms, environment 283ms)

vitest exit code: 0
=== DONE ===
=== POST-RUN: frozen snapshot untouched? ===
clean: no PoC file and no patch hunk present in the snapshot
script exit: 0
```

### What is measured, and what is argued

| Claim                                                                                                                                                                                                                                              | How it is established                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| For `/Alice.eth/ownership/transfer`, the wallet is handed `setResolver`, `setSubregistry` and `safeTransferFrom` keyed on `labelhash('alice')` and the token id of `alice`, and nothing keyed on `Alice`, while every rendered name is `Alice.eth` | Executed green: file 1, run 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| The owner, registry and resolver reads are asked for `Alice.eth`; the role and token reads for `alice`                                                                                                                                             | Executed green: file 1, run 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| The same run for `alice.eth` produces the same three writes on the same canonical label (control)                                                                                                                                                  | Executed green: file 1, run 1, second test                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| All three options render checked when the name carries a resolver, a subregistry and an address record                                                                                                                                             | Executed green: file 1, run 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| After the third confirmation the flow navigates to `/$name/ownership` with the lookalike as the parameter                                                                                                                                          | Executed green: file 1, run 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| That ownership page then shows the victim still owning the lookalike                                                                                                                                                                               | Argued from source: `routes/$name/ownership/index.tsx:22-31` (the owner read on the segment as typed), `:45-53` (`isConnectedOwner`). The ownership route is not mounted by this proof of concept                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `/Alice.eth/ownership` renders a Transfer link to `/$name/ownership/transfer` for the connected owner who holds the transfer role on `getLabel(name)`                                                                                              | Argued from source: `routes/$name/ownership/index.tsx:49-53`, `:58-63`, `:102`, `:114-119`. Not mounted here; file 1 enters the transfer route directly, which is the other of the two links the walkthrough offers                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Each step's description handed to the transaction manager names `Alice.eth`: `Detach resolver for Alice.eth`, `Detach registry for Alice.eth`, `Transfer Alice.eth`                                                                                | Executed green: file 1, run 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| The patch refuses the non-normalised segment at the layout `beforeLoad` and admits the canonical one                                                                                                                                               | Executed green: file 1, runs 1 and 3. The FLIP assertion passes in run 1 and fails in run 3; the `with the fix applied` block is skipped in run 1 and passes in run 3                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| `getLabel(<lookalike>)` is `alice`; `labelhash(<lookalike label>)` differs from `labelhash('alice')`; `namehash(<lookalike>)` differs from `namehash('alice.eth')`; `isValidEnsName(<lookalike>)` is false                                         | Executed green: file 2, runs 2 and 4                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `setResolver(labelhash('alice'), ...)`, `setSubregistry(labelhash('alice'), ...)` and `getTokenId(labelhash('alice'))` address the `alice` entry in the registry                                                                                   | Argued from the registry's verified source, with the barrier named. What is not run: the three recorded requests are not mined against a deployed registry. Why: the registry is not part of the audited asset, its build tree is not in this repository, and nothing in this proof of concept may contact a chain. Why the unrun step cannot change the answer: it is a hash-keyed storage lookup with no branch. `LibLabel.sol:7-9` makes the key `uint256(keccak256(bytes(label)))`; `PermissionedRegistry.sol:593-595` zeroes the version bits and indexes the entry by that value and nothing else; `:306-308`, `:142-147` and `:150-155` each resolve their `anyId` through it. No path routes `uint256(keccak256('alice'))` to any entry but `alice`, and reaching the `Alice` entry needs `uint256(keccak256('Alice'))`, which run 1 asserts appears in no recorded request |
| The victim holds every role the three writes need on `alice`                                                                                                                                                                                       | Argued from source: `ETHRegistrar.sol:17-23`; the role reads in file 1 answer from a table carrying that bitmap                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| The victim cannot undo a completed transfer of a token they no longer hold                                                                                                                                                                         | Argued from source, with the same barrier and the same reason it cannot change: `PermissionedRegistry.sol:480-498` (`_update` reverts unless `from` holds `ROLE_CAN_TRANSFER_ADMIN` on the token, at `:491`, and afterwards the victim holds no token for `alice`; ERC-1155 admits a transfer only from the holder or from an address the holder has approved), `:435-437` (`register` reverts `LabelAlreadyRegistered` while the entry is unexpired and owned), and the contract declares no reclaim, restore, recover or rescue function. Each is a check on the caller or on the entry's own state; none consults the string the page displayed                                                                                                                                                                                                                                  |
| The attacker can send the lookalike to an externally owned account, and the oracle accepts the label `Alice`                                                                                                                                       | Argued from source: `ETHRegistrar.sol:17-23`, `PermissionedRegistry.sol:491`, `StandardRentPriceOracle.sol:272-277,365-368`, and the ERC-1155 standard for the receiver hook. Not exercised: no token is sent and no registration is made                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |

### What this PoC does not establish

None of what follows sits under the impact entry this report files on.

1. **The chain from the rendered route to a mined transaction, in one execution.** File 1 stops at the app's own transaction boundary, where the request is complete and decoded; what the registry does with the `anyId` is cited from its verified source rather than executed. The step that would close it is a registry deployed on a local node, these three requests mined against it, and `balanceOf` read back before and after; it is not run here, for the reasons in that row of the table above, and nothing in this proof of concept contacts a chain.
2. **That any wallet on Sepolia currently holds a lookalike.** The PoC creates the condition in a table; it does not survey for it, and no registration or transfer was made.
3. **The behaviour of a contract wallet as the victim.** File 1's victim is an externally owned account.

### Placement

```bash
cp send_name_signs_for_the_normalised_twin.poc.test.tsx 'apps/portal/src/routes/$name/ownership/'
cp get_label_folds_what_the_route_does_not.poc.test.ts   apps/portal/src/utils/token/
```

```bash
cd apps/portal
NO_COLOR=1 ./node_modules/.bin/vitest run \
   'src/routes/$name/ownership/send_name_signs_for_the_normalised_twin.poc.test.tsx' \
   --reporter=verbose
NO_COLOR=1 ./node_modules/.bin/vitest run \
   src/utils/token/get_label_folds_what_the_route_does_not.poc.test.ts \
   --reporter=verbose
```

The complete capture of the four runs ships in the Gist as `CAPTURE_20260912.log`, 137 lines, with the same `<archive-root>` substitution in its four banner lines and no other change; the blocks above are its lines 1 to 19, 20 to 41, 42 to 63 and 64 to 137.
