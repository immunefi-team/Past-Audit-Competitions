# 93018 w a critical raw cased profile url renders the real owner over an attacker resolver s address

**Submitted on Sep 13th 2026 at 21:35:09 UTC by @ChaseTheLight for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93018
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

The Manager's profile route takes the URL segment as typed and hands that one string to two reads. The owner read normalizes it to its ENSIP-15 form before touching the chain, so the owner, the expiry and the page's registration state come from the canonical node. The record read does not normalize. The ENS client library selects the resolver by the name as typed and reads the text records under the hash of the name as typed, so the resolver, the description and the social records come from a different node whenever the segment is not already canonical; the address records are requested under the canonical hash, but from that other node's resolver. `alice.eth` and `Alice.eth` are two names on chain (their namehashes differ), two registrations with two owners and two resolvers. A visitor who opens `/Alice.eth` is therefore shown the owner of `alice.eth` and the records served by the resolver of `Alice.eth` on one page, presented as one identity. An attacker who registers the label `Alice` as typed, naming as its resolver a contract of theirs that answers the address query for the canonical node with their own address (a registration and resolver writes argued from contract source, not performed for this report), gets a page at the Manager's `/Alice.eth` route that names the real owner of `alice.eth` in its Owner row and offers the attacker's address under the heading "Main receiving address", behind a one-click copy button. The page presents one identity: the heading is the name as typed and the Owner row answers for the canonical name. The victim reaches it by a shared link, or by typing the name with a capital letter, which the app's own search would fold and the address bar does not; once there, the page's share action emits the same raw URL. No wallet is connected and no transaction is built in the Manager; the loss lands wherever the victim pastes the copied address. The defect is two application services disagreeing about which node one string names, and it is closed by one canonicalisation step before either read.

## Vulnerability Details

### Root cause

One URL segment reaches the chain twice, canonicalised once. The route parameter is accepted as typed (`apps/manager/src/routes/$name/route.tsx:5-13`): the only checks are a minimum length and the presence of a dot.

```ts
    parse: ({ name }) => {
      if (name.length < 3 || !name.includes('.')) {
        throw new Error('Invalid ENS name')
      }

      return {
        name,
      }
    },
```

The page loader hands that string to the record query and the owner query together (`routes/$name/index.tsx:51-56`):

```ts
    const [profileRecords, ownerData] = await Promise.all([
      queryClient.ensureQueryData(profileRecordsQuery(name)),
      // Fetch, not ensure: `ensureQueryData` serves invalidated data, so a name
      // cached as ownerless pre-registration would redirect its owner away.
      queryClient.fetchQuery(profileOwnerQuery(name)),
    ])
```

The owner service canonicalises first. `getOwner` calls `normalizeEthName(params.name)` (`features/profile/service/profileOwner.ts:25`), which runs viem's ENSIP-15 `normalize` and returns the folded string (`profileName.ts:14-21`); every chain read in that service uses the normalized name: the v2 owner at `:56` and the v1 fallback at `:88` use `ethName.name`, and the DNS branch uses the normalized `dnsName` (`:39`). The expiry service does the same through `normalizeEth2LdName` (`profileExpiry.ts:77`, reads at `:94` and `:106`). The record service does not. `getProfileRecords` (`profileRecords.ts:82-154`) queries the record index with `{ id: name }` as given (`:49`, called at `:95`) and then asks the client library for the records of `name` as given (`:122-132`):

```ts
  const records = yield* fromPromise(
    getRecords(client, {
      name,
      texts,
      coins,
      contentHash: true,
      abi: true,
      ignoreInvalidCoinTypes: true,
    }),
    (error) => new GetProfileRecordsError({ cause: error }),
  )
```

The library selects the resolver by the name as typed and hashes the name differently per record type. The universal-resolver call that locates the resolver wraps `toHex(packetToBytes(nameWithSizedLabels))`, the DNS encoding of the name as given (`@ensdomains/ensjs`, `dist/actions/public/resolveNameData.js:40,54`). The text, content-hash and ABI calls are built on `namehash(name)` (`dist/utils/coders/getText.js:7`, `getContentHash.js:8`, `getAbi.js:8`). The address call is the exception: the address coder imports `normalize` from `viem/ens` (`dist/utils/coders/getAddress.js:3`) and builds `args: [namehash(normalize(name))]` for coin 60 (`:33-35`) and `[namehash(normalize(name)), coinType]` for every other coin (`:43-45`), and `getRecords` routes each requested coin through it (`dist/actions/public/getRecords.js:22-27`). `namehash` is keccak over the label bytes, so `namehash('Alice.eth')` is `0x4a46419fb486ae90839e2a549d4be74a30ae9a1525e537f9c2aab4b9f33331b0` and `namehash('alice.eth')` is `0x787192fc5378cc32aa956ddfdedbf26b24e8d78e40109add0eea2c1a012c3dec`. For the URL segment `Alice.eth` the record read therefore goes to the resolver of the node `Alice.eth`, asks it for the text records of `Alice.eth` and for the ETH address of `alice.eth`, and reports that resolver's address as the page's resolver.

| What the page shows                                                     | Read by                                                                    | Name used                                                        | Node                                             |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------------- | ------------------------------------------------ |
| Owner row, registration state, the decision to render a profile at all  | `profileOwner.ts:25,56,88`                                                 | `normalize(name)`                                                | canonical                                        |
| Expiry                                                                  | `profileExpiry.ts:77,94,106`                                               | `normalize(name)`                                                | canonical                                        |
| Resolver address, description, social and contact records, content hash | `profileRecords.ts:95,123-124`; `resolveNameData.js:40,54`; `getText.js:7` | `name` as typed                                                  | raw                                              |
| "Main receiving address", chain-specific addresses                      | `profileRecords.ts:123-124`; `getAddress.js:33-35,43-45`                   | `normalize(name)` for the hash, `name` as typed for the resolver | canonical hash, asked of the raw node's resolver |
| Document title, `og:description`                                        | `index.tsx:122-124,137`                                                    | `name` as typed, raw node's `description`                        | raw                                              |

The two halves meet on one screen. `ProfileView` subscribes to both queries with the same `name` (`components/view/ProfileView.tsx:93-96` and `:99-101`), passes the canonical owner and the records read on the raw name into the same header (`:192-211`, `owner` at `:207`, `records` at `:209`), and those records into the cards (`:213-217`). The header renders the segment as typed as the page heading (`ProfileHeader.tsx:51`, `ProfileDetails.tsx:22-28`) directly above the Owner row, which shows the canonical node's owner, or that owner's primary name when one is set, with its own copy button (`ProfileDetails.tsx:113-123`).

The sink is the address card. `ProfileAddressesSection` picks the main address from the records the raw node's resolver returned (`ProfileAddressesSection.tsx:159`); the picker takes the coin-60 entry first (`ProfileView.helpers.ts:229-237`, the `find` at `:234`). That address is rendered under the heading "Main receiving address" (`:170-172`) inside a copy button whose value is the address itself (`:90-95`):

```tsx
    <CopyableButton
      className={`${cardSurfaceClassName} ${addressCardPaddingClassName} h-auto min-h-13.75 w-full justify-between gap-2 py-4 lg:landscape:max-w-132.75`}
      iconClassName={profileCardCopyIconClassName}
      iconStrokeWidth={profileCardTrailingIconStrokeWidth}
      value={address.value}
    >
```

`CopyableButton` also sets the button's `title` to that value (`components/atoms/CopyableButton/CopyableButton.tsx:47`), so the full address is what the visitor copies and what their browser shows on hover. Coin 60 is in the probe list on every profile load (`features/profile/data/records/utils.ts:42`, spread at `profileRecords.ts:117`), so the ETH address comes from the chain read, the query the library sends to the raw node's resolver under the canonical hash.

The rest of the app treats the two spellings as one name. The header search lowercases its input before it navigates (`features/navigation/Header/search/searchSuggestions.utils.ts:54`), the registration parser lowercases before it validates (`features/register-v2/utils/name-parser.ts:46`), and the owner and expiry services normalize. The project's own test states the intent for the owner path: `normalizeEthName('Sub.FOO.ETH')` must return `sub.foo.eth` (`profileName.test.ts:5-11`).

### Trigger and preconditions

The victim is any visitor to a profile URL. The attacker needs one registration and no interaction with the victim's wallet or the Manager.

1. **A registered name exists.** Call it `alice.eth`, registered on Sepolia, which is the only chain the Manager is built against (`lib/wagmi.ts:80`, `chains: [sepoliaWithEns]`) and the programme's stated network ("Ethereum Sepolia is the primary network."). The loader takes the fallback or redirect branch when the canonical node has no owner (`index.tsx:77-93`), so the attack page only renders for a name that is registered, which is the ordinary case.
2. **The attacker holds the node `Alice.eth`, and its resolver answers the canonical node's address query.** `Alice.eth` is a separate registration of the label bytes `Alice`. The registry keys each entry on `LibLabel.id(label)` (`live_verified/PermissionedRegistry.0xBDC85dD5.sol:424`, verified source), which is `uint256(keccak256(bytes(label)))` (`LibLabel.sol:7-9`, `ensdomains/contracts-v2` at commit `48b3e2d3`), so `Alice` and `alice` are separate entries. The registrar that calls them is read from that same commit rather than from verified deployment source. The registrar prices through an `IRentPriceOracle` it holds in storage and its owner can replace (`AbstractETHRegistrar.sol:34,78-81`, from `ensdomains/contracts-v2` at commit `48b3e2d3`). The oracle pinned for this deployment by the client library the applications use is `StandardRentPriceOracle` at `0x8914b66260EB8C4fff795650c3AE8Cd335958987` (`@ensdomains/ensjs`, `src/clients/l1.ts:140` for the Sepolia block, entries at `:174-199`), whose published source is verified on the block explorer (solc 0.8.27, verified 9 August 2026) and byte-identical to `ensdomains/contracts-v2` at branch `main` as of 12 September 2026. There, `isValid(string label)` is `getBasePrice(label, 1) > 0`, and its own docstring reads "Does not check if normalized."; `getBasePrice` applies no normalisation or case rule: it returns 0 when `bytes(label).length` is 0 or above 255, and for a label inside that bound it prices from the deployment's rate entry for the label's codepoint count, so a label whose codepoint count maps to a zero rate would price at 0 and be rejected as invalid. `isValid` is a view helper; a registration reaches the same predicate through `getRegisterPrice` and `_requireBasePrice` (`:310`, `:400-409`), which reverts `NotValid(label)` on a zero price. Neither `isValid` nor `getBasePrice` folds case or normalises (`:275-277`, `:365-374`). the pinned v2 `ETHRegistrar` ABI (`@ensdomains/ensjs-abi`, `v2/ethRegistrar`, a dependency rather than part of the audited code) declares `register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer)`, `isAvailable(string label)` and an error set that includes `NameNotAvailable(string label)` and `NotValid(string label)`, the latter annotated in the ABI as bubbled from `IRentPriceOracle`, so the label-validity rule the registrar applies is the rent price oracle's. The application says the same in its own words: "the ENSv2 contracts impose no character constraints on labels (only a 1–255 byte length)" (`apps/portal/src/utils/token/isNameOrAddress.ts:9-10`). The fold is client-side: the Manager lowercases before it hands a label to the registrar (`name-parser.ts:46`); the Explorer's register page rejects a name that is not already normalized (`apps/portal/src/utils/token/nameValidation.ts:25-27`, rendered at `apps/portal/src/features/register/components/RegisterName.tsx:42-44,84-89`) and folds nothing; and the transaction manager both apps drive strips `.eth` and folds nothing else (`packages/transaction-manager/src/machines/registration/registration.actors.ts:141,244`). A direct caller sees none of these. No registration was made for this report.

   The page's ETH-address call carries `namehash('alice.eth')` and is sent to the resolver of `Alice.eth`, so the attacker's resolver must answer `addr(namehash('alice.eth'))` with the attacker's address. The v2 `register` takes the resolver as an argument (the ABI above; the registration flow passes it at `registration.actors.ts:242-252`), so the attacker names it at registration: a contract of theirs that returns a fixed address for every `addr` query; the registrar's interface takes a plain address for it, and the deployed registry applies no check to it: `PermissionedRegistry._register` stores the argument as `entry.resolver` with no validation of any kind (`:460`), and `getResolver(label)` returns the stored value for any label whose entry has not expired (`:257-259`), both verified source. The resolver contract is not part of the audited code; this route needs no property of it beyond the answer it returns. So the attacker sends one `commit` and one `register` for `Alice` naming their resolver, sets that resolver's answer for `addr(namehash('alice.eth'))` to their own address, and copies the victim's `avatar` and `description` text records under `namehash('Alice.eth')`. One registration fee.
3. **The victim opens `/Alice.eth`.** The route is `/$name/` (`routeTree.gen.ts:73`, `:187`), and `/p/$name/` redirects to it with the parameter unchanged (`routes/p/$name/index.tsx:9`), so the app's own share-URL shape reaches the same loader. The route is client-rendered (`index.tsx:142`), so both reads run in the visitor's browser against the app's RPC transport (`lib/wagmi.ts:68-74`). The page's share action builds `${origin}/p/${name}` from the segment as typed (`ProfileView.tsx:60-65`), so a raw URL, once opened, propagates itself through the app's own control.

Who is immune: a visitor who types `Alice.eth` into the header search, because the search folds case before it navigates (`searchSuggestions.utils.ts:54`) and lands on `/alice.eth`. The address bar, a pasted link and the share URL do not pass through that fold.

### Recommended fix

Canonicalise the segment once, before any read, so one URL can only ever name one node. Two changes, both small, and the helper they need sits beside the existing normalisers (`viem/ens` `normalize` is already imported at `profileName.ts:1`):

1. In the loader (`routes/$name/index.tsx:50`), compute the ENSIP-15 form of `name`; if it has none, return the not-found fallback; if it differs from the segment, `redirect` to `/$name` with the canonical value before either query is issued.
2. In `getProfileRecords` (`profileRecords.ts:82`), canonicalise the input and refuse a name that has no canonical form, so the record read cannot name a different node from the owner read for any other caller of `profileRecordsQuery`.

The structural fix is to canonicalise in the route parameter parser (`route.tsx:5-13`) so every route under `/$name` receives the canonical string and the two services cannot be handed different spellings. The full patch and which assertions it flips are in the Proof of Concept section.

## Impact Details

### The in-scope impact this maps to

The governing entry, quoted verbatim from the programme's Impacts in Scope list:

> Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as: - resolving or rendering an ENS name to an attacker-controlled address - showing the wrong recipient or amount on a transaction the user then approves

This report files at **Critical** on that entry, on its first sub-example. The application renders the ENS name `alice.eth`, as identified by its own Owner row, expiry and registration state, to an address the attacker controls, read from the attacker's resolver, under the heading "Main receiving address", in a copy button.

### Loss mechanism and its bound

The loss path is a misdirected payment. An ENS profile's main receiving address is what the page exists to publish, and the Manager puts it behind a copy button so the visitor pastes it into whatever they are sending from. The address the button holds belongs to the attacker's resolver. Nothing on the page reads as a second name: the heading carries the name the visitor asked for, the Owner row carries the real owner (or the real owner's primary name), the expiry is the real expiry, the description and avatar are copies of the real ones. Where the owner has set a primary name the Owner row renders it, so a visitor comparing that string against the heading would see they differ; where the owner has not, the row renders a truncated address and the page offers no such comparison. The Proof of Concept measures the second case.

Per victim the loss is whatever they send, so it is bounded by the victim's intent, not by anything in the application. The attacker's cost is one registration and the resolver it names. The window is the lifetime of the attacker's registration, and the page needs no further action from the attacker once the resolver is set. A specific figure is not offered because the amount sent to a copied address is a property of the victim, not of the code.

The Manager itself takes no transaction from the victim in this flow, so the sub-example that applies is rendering, not pre-filling. The programme's Critical entry names rendering explicitly.

### Why not lower, and the out-of-scope entries to weigh

No lower entry on the list covers a name rendered to an attacker's address. The High entries cover static-content injection and other users' details, neither of which is this; the head clause of the Critical entry, a user sending funds to an unintended recipient, is the harm.

**"Reflected plain text injection (e.g. url parameters, path, etc.)"** does not apply. The page does echo the segment as typed in its heading and document title (`ProfileDetails.tsx:22-28`, `index.tsx:137`), and this report puts no weight on that echo. The attacker's address is not in the URL and is not reflected from it; it is read from the chain, from a node the attacker registered, because the application asked for the wrong node. The heading matters only as the reason the visitor sees one identity: the name as typed sits above an Owner row that answers for the canonical name.

**"Impacts caused by vulnerabilities requiring un-prompted, in-app user actions that are not part of the normal app workflows"** does not apply. Opening a profile URL is the Manager's primary workflow, and the app's own share action produces the raw-cased URL once one has been opened (`ProfileView.tsx:60-65`).

**"UX and UI impacts that do not materially disrupt use of the platform"** does not apply. The defect site is a React component, so the entry is worth naming, but the outcome is a copied attacker address, not a rendering artefact.

**"Theoretical impacts without any proof or demonstration"**: the Proof of Concept drives the real loader, renders the page's real header and cards from what it cached, on in-process fixtures standing in for the chain and the record index, with the attacker's resolver answering as step 2 requires, and reads what the copy buttons hand to the clipboard.

The precondition a reviewer might weigh is that the attacker's setup is a registration and a resolver that answers for the canonical node. The Critical entry's own sub-example presupposes it: "resolving or rendering an ENS name to an attacker-controlled address" requires the attacker to control what a name resolves to, which is resolver state on chain. The sibling Critical entry on the same list names contract-side state as well: "Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address". The programme's ladder accepts harm whose setup lives in contract state; what this report files on is what the application does with one string.

### Not a duplicate, and scope

The nearest Known Issue is **EXP-INPUT-005**, "ENS name validators accept deceptive characters", verbatim:

> Our name validators accept inputs that ENS normalization later rejects or transforms, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI rather than incorrect resolution.

That entry's own impact sentence excludes this report's consequence. Three differences, each from source:

1. **No validator is involved.** EXP-INPUT-005 is about what the name validators accept. The profile route parameter passes through no validator: `route.tsx:6` checks a length and a dot and returns the string. There is nothing on this path for a validator fix to tighten.
2. **The input is not a deceptive character.** It is an ordinary capital letter, which the Manager's own parsers lowercase (`name-parser.ts:46`, `searchSuggestions.utils.ts:54`) and the Explorer's validator rejects as non-normalized. The root cause is that two read paths fed one string canonicalise differently (`profileOwner.ts:25` versus `profileRecords.ts:95,123-124`), not that a validator admits a confusable glyph.
3. **The consequence is incorrect resolution**, the outcome the entry's text sets aside. The page resolves one name's records and another name's owner and presents them as one profile; the attacker's address comes from an on-chain read of the resolver of a node the attacker owns.

The Known Issues document also states, in its own terms, where the line falls: "Anything not listed here is in scope under the normal competition rules, including: ... new consequences of a listed root cause that materially change its severity". Even on the broadest reading of "transforms", an Owner row vouching for an attacker's receiving address is a new consequence at a different severity from display confusion.

No other entry is close. **WEB-1301** ("Address view shows misleading owner and expiry metadata") is the address page showing the wallet's primary name's metadata, a functional defect with no attacker and no second node. Part 2 deduplicates by behaviour ("a report that describes one of these behaviors is a duplicate of this section"), and no Part 2 item describes a profile rendering another node's records. No entry in Part 1 or Part 2 concerns how the profile route canonicalises its name; `getProfileRecords`, `normalizeEthName` and the address card appear nowhere in the document.

The defect is in `apps/manager`, the listed asset "Manager app Files" dated 6 August 2026, in the app's own route and services. Nothing on the out-of-scope list is at fault. The address the page renders is a resolver's answer, and what this report files on is that the application sends that query to the resolver of a different node from the one whose owner it displays (`index.tsx:50-56` hands the raw segment to both queries; `profileOwner.ts:25` normalizes, `profileRecords.ts:124` does not). The defect and its fix are in the application: both reads that produce the page are RPC calls the app itself makes on every profile load, the record index is queried with the same raw string and contributes no bytes of the rendered address, and the fix is one canonicalisation step in the application. The record query does need the index to answer for the raw id rather than error, and the Proof of Concept records that dependency. No feature flag or branch in the loader or in `ProfileView` avoids the defect (`index.tsx:165-174`, `ProfileView.tsx:80-241`); the surrounding page shell is not mounted by the Proof of Concept and is argued from source not to change which node either read comes from.

## References

Repository: `https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager` (the Manager asset, dated 6 August 2026). Paths below are relative to `apps/manager/src` unless stated.

1. `routes/$name/route.tsx` - `#L5-L13` (the parameter parser: length and a dot, string returned as typed)
2. `routes/$name/index.tsx` - `#L51-L56` (both queries issued on the raw segment), `#L77-L93` (the ownerless branch, which the attack page never takes), `#L122-L124` (the raw node's description becomes the page description), `#L131-L141` (`head`, title from the segment as typed at `#L137`), `#L142` (client-rendered)
3. `features/profile/service/profileOwner.ts` - `#L25` (`normalizeEthName`), `#L56` (v2 owner read on `ethName.name`), `#L88` (v1 fallback on `ethName.name`); `profileName.ts` - `#L1` (`normalize` import), `#L14-L21` (ENSIP-15 fold); `profileExpiry.ts` - `#L77`, `#L94`, `#L106`
4. `features/profile/service/profileRecords.ts` - `#L49` (record index asked with `{ id: name }`), `#L95` (that lookup), `#L117` (coin 60 always probed), `#L122-L132` (the record read on `name` as typed), `#L165-L169` (the query, key carrying `name` unchanged); `features/profile/data/records/utils.ts` - `#L42`
5. `features/profile/components/view/ProfileView.tsx` - `#L60-L65` (share URL from the segment as typed), `#L93-L96` and `#L99-L101` (records and owner subscribed with the same `name`), `#L192-L211` (owner and records into one header), `#L213-L217` (records into the cards); `ProfileHeader.tsx` - `#L51`, `#L63-L66`; `ProfileDetails.tsx` - `#L22-L28` (heading), `#L113-L123` (Owner row)
6. `features/profile/components/view/ProfileAddressesSection.tsx` - `#L159` (main address picked from the records), `#L90-L95` (copy button, `value={address.value}`), `#L170-L172` ("Main receiving address"); `ProfileView.helpers.ts` - `#L229-L237` (coin 60 first); `components/atoms/CopyableButton/CopyableButton.tsx` - `#L47` (`title={title ?? value}`)
7. Other name entries, which fold case: `features/navigation/Header/search/searchSuggestions.utils.ts` - `#L54`; `features/register-v2/utils/name-parser.ts` - `#L46`; `features/profile/service/profileName.test.ts` - `#L5-L11`
8. `lib/wagmi.ts` - `#L68-L74` (the public client), `#L80` (`chains: [sepoliaWithEns]`); `routeTree.gen.ts` - `#L73`, `#L187` (`/$name/`), `#L135`, `#L192` (`/p/$name/`); `routes/p/$name/index.tsx` - `#L9`
9. `apps/portal/src/utils/token/isNameOrAddress.ts` - `#L9-L10` (the application's statement that v2 label validity is length only)
10. Dependencies, pinned in `pnpm-lock.yaml` and outside the audited code: `@ensdomains/ensjs` `dist/utils/coders/getAddress.js:3,33-35,43-45` (`namehash(normalize(name))`), `dist/utils/coders/getText.js:7`, `getContentHash.js:8`, `getAbi.js:8` (`namehash(name)`), `dist/actions/public/getRecords.js:22-27` (each coin through the address coder), `dist/actions/public/resolveNameData.js:40,54` (`packetToBytes`); `@ensdomains/ensjs-abi` `v2/ethRegistrar` (`register(string label, address owner, bytes32 secret, address subregistry, address resolver, ...)`, `isAvailable`, `NotValid(string label)`), `v2/permissionedResolver` (`authorizeNameRoles`, `hasRootRoles`)
11. Resolver deployment and record writes, in the monorepo packages the apps drive (paths from the repository root): `packages/transaction-manager/src/machines/registration/registration.actors.ts` - `#L67-L73`, `#L99-L105` (the proxy initialised with the full role bitmap), `#L141`, `#L244` (`.eth` stripped, no fold), `#L242-L252` (`register` with the resolver argument); `packages/smart-account/src/providers/rhinestone/manifest.ts` - `#L166-L172` (`ROLES_ALL`); `packages/smart-account/src/providers/rhinestone/registration-calls.ts` - `#L321-L330` (`authorizeNameRoles(0x00, ...)`); `apps/manager/src/features/migration/service/buildProfileReplayCalls.ts` - `#L20-L27` (`setAddr(node, coinType, value)`); `apps/manager/src/features/migration/service/migrationInvariants.ts` - `#L284-L286`; `apps/portal/src/utils/token/nameValidation.ts` - `#L25-L27`; `apps/portal/src/features/register/components/RegisterName.tsx` - `#L42-L44`, `#L84-L89`

File fingerprints at the audited snapshot:

| File                                                                            | Bytes | Lines | sha256                                                             |
| ------------------------------------------------------------------------------- | ----- | ----- | ------------------------------------------------------------------ |
| `apps/manager/src/routes/$name/route.tsx`                                       | 441   | 20    | `7c4f3019baf172d9b16202ab508e24460aa2b4b58c59f7f1bb2e22505e7a07ef` |
| `apps/manager/src/routes/$name/index.tsx`                                       | 5,835 | 174   | `d310da0134431ec3fe0b0316d9a9652e2e9697af5eb2be876608707e332428a1` |
| `apps/manager/src/features/profile/service/profileOwner.ts`                     | 3,040 | 106   | `aa8593aaa34ab7d8509d801813eb38aa1eeea87e6d84479df2fa5e1465ef5b2c` |
| `apps/manager/src/features/profile/service/profileName.ts`                      | 1,431 | 73    | `6649ead5955265a6904a72c0c3b32aa05b3c69aead0f741bdf3d89ca964f1838` |
| `apps/manager/src/features/profile/service/profileRecords.ts`                   | 5,086 | 169   | `1e0cda446194adae5624a2b19529456fa04a22a43aec677e0c9c7dea47add44f` |
| `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx` | 6,698 | 199   | `7bf7fe06abb63684d0bd3ce716e02532fb42981a66db5d695243ba5ac5385067` |
| `apps/manager/src/features/profile/components/view/ProfileView.helpers.ts`      | 7,637 | 277   | `49533475023ecb3ec5b1e3eebd8daa4086c977a7f3c1f039d41c097f9b346802` |

## Link to Proof of Concept

<https://gist.github.com/ChaseTheLight01/566c08e6fbd65ca4d6960247802085b2>

## Proof of Concept

## Proof of Concept

Three test files and one fix patch, all against the competition snapshot at `audit-comp-ready`. The tests run offline: nothing opens a socket, resolves a hostname, signs anything or touches a deployed system. The chain and the record index are replaced in-process by fixtures keyed on the name string the application hands them. The fixture standing in for the record read answers, for the raw spelling, what the attacker's resolver is set up to answer (the precondition in the report's step 2); the library's own hashing of that string is established from its source, not by the fixture. The tests print which string the application asked each fixture for and what the page then shows. they were executed on 2026-09-12, on Node 22.18.0 and vitest 4.1.5, inside a sealed network namespace, and the captured runs are in the last section.

| File                                              | Enters at                                                                                                 | Measures                                                                                                                                                                                                                                                                                          |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. `profile_name_split_resolution.poc.test.ts`    | the two real profile services, `getOwner` and `getProfileRecords`                                         | which name string each service asks the chain for, and whose owner and whose ETH address come back for one URL segment                                                                                                                                                                            |
| 2. `profile_route_loader_raw_node.poc.test.tsx`   | the real route loader and `head()`, then the page's header and cards rendered from what the loader cached | that the page renders as a normal profile with every profile query cached under the raw name; that one screen shows the canonical node's owner in the Owner row and the address the raw node's resolver answered under "Main receiving address"; and what each copy button hands to the clipboard |
| 3. `name_subpages_skip_validity_gate.poc.test.ts` | the Explorer's own `isValidEnsName` and its route sources                                                 | that the sibling app rejects the raw spelling on its name index page and applies no gating check on the layout, whose only use of that function is a TLD-redirect condition, or on five of the subpages under the same segment (`address`, `records`, `subnames`, `history`, `resolver`)          |

Files 1 and 2 carry the finding; file 3 is the corroborating variant in the other listed app. Every function and component under test is production code; the boundaries replaced are listed below, and the record read is one of them. A single secret Gist holds runnable copies of the three files, the patch, the captured run `CAPTURE.log`, and a `README.md` for placement and commands; its link and its pinned revision hash: <https://gist.github.com/ChaseTheLight01/566c08e6fbd65ca4d6960247802085b2/3a6ebaef2df97d0041db9262833bf45f85858c86>. Nothing below depends on opening it.

### Where these run, and what they never touch

Files 1 and 2 run in process under vitest with happy-dom, inside `apps/manager`, with the app's own vitest configuration. Five module boundaries are replaced: the two ensjs owner reads (`@ensdomains/ensjs/public/v1` and `/v2`), the ensjs record read (`getRecords` from `@ensdomains/ensjs/public`, with the rest of that module left real), the record-index client module, and the wagmi client factory (`@/lib/wagmi/helpers`). The project's own `profileOwner.test.ts:12-31` replaces four of those five the same way, all but the record read. Every replacement is a lookup into one in-memory map of two registrations, `alice.eth` and `Alice.eth`, keyed on the exact string and holding a distinct owner, resolver, ETH address and description for each. Nothing in the fixtures computes, folds or compares names; they answer for the key they are given, and each records the key. The record-read fixture models the resolver's answer, not the library's calldata: in production the library selects the resolver by the raw name and requests the ETH address under the canonical hash (`getAddress.js:33-35`), and the fixture's answer for the key `Alice.eth` is what that resolver returns under the precondition in step 2 of the report. The fixture does not measure that calldata; the claims table below says which rows this affects.

File 2 additionally replaces two functions of the router the way the project's own route test does (`routes/$name/index.test.tsx:6-18`), so the loader and `head` can be called directly; the rest of that module stays real, because the app's button component calls `createLink` at import (`components/ui/button.tsx:86`) and the Owner row imports `useHydrated` (`ProfileDetails.tsx:2`). The page component and its two fallbacks are inert where the route module imports them, since the page is rendered from the two children it composes (test B below). Four queries the loader issues after the owner check (expiry, registration, the owner's primary name, availability) answer inert data under the app's own key shape; none of them is on the path that decides which node the records come from. Two decorative leaves (`PatternAvatar`, `ImageFallback`) are stubbed so happy-dom renders the avatar and the address card; neither is measured. `@/lib/wagmi`, which builds a WalletConnect connector at import, is replaced the way the app's own tests replace it (`lib/smart-account/rhinestone.test.ts:37-43`); nothing on the file's import graph reaches it. Rendering goes through the app's own test wrapper (`utils/test-utils.tsx:71`, the one `ProfileHeader.test.tsx:4` renders with), and the clipboard is the app's own test double (`lib/clipboard.test.ts:7-16`).

File 3 reads the Explorer's route sources as text through Vite's raw import and calls two pure functions, the Explorer's `isValidEnsName` and viem's `namehash`. No network, no chain, no wallet, in any file.

### File 1: one URL segment, two services, two nodes

The file's first test is the pure fact that the two spellings hash to two nodes. Its second test drives the real `getOwner` and the real `getProfileRecords` with the same string, `Alice.eth`, and reads back which key each fixture was asked for and what it returned. Verbatim from the file, the other tests elided:

```ts
  it('the owner is read on the normalized name, the records on the raw one', async () => {
    const owner = (await getOwner({ name: 'Alice.eth' }))._unsafeUnwrap()
    const records = await getProfileRecords('Alice.eth')

    console.log(
      [
        'URL segment                : Alice.eth',
        `owner service asked for    : ${world.asked.ownerV2.join(', ')}`,
        `records service asked for  : ${world.asked.records.join(', ')}`,
        `record index asked for     : ${JSON.stringify(world.asked.index)}`,
        `owner rendered             : ${owner?.owner}`,
        `ETH address rendered       : ${ethAddressOf(records)}`,
        `resolver rendered          : ${records._unsafeUnwrap().resolverAddress}`,
      ].join('\n'),
    )

    expect(world.asked.ownerV2).toEqual(['alice.eth'])
    expect(world.asked.records).toEqual(['Alice.eth'])
    expect(world.asked.index).toEqual([{ id: 'Alice.eth' }])
    // The owner came from the v2 read on the canonical node; the v1 fallback
    // (profileOwner.ts:87-90) was never consulted. Over every boundary, the
    // chain was asked for exactly two strings, one per service.
    expect(world.asked.ownerV1).toEqual([])
    expect([...world.asked.ownerV2, ...world.asked.records]).toEqual([
      'alice.eth',
      'Alice.eth',
    ])

    expect(owner?.owner).toBe(world.VICTIM)
    expect(ethAddressOf(records)).toBe(world.ATTACKER)
    expect(records._unsafeUnwrap().resolverAddress).toBe(
      world.ATTACKER_RESOLVER,
    )
    expect(
      records._unsafeUnwrap().texts.find((t) => t.key === 'description')
        ?.value,
    ).toBe('send funds here')
  })
```

The two that follow close the collection: the v1 owner fallback was never consulted, and across every boundary the chain was asked for exactly two strings, one per service. The four after that are the consequence: the owner returned is the victim; the resolver returned and the description are the attacker's; and the ETH address returned is the attacker's, which is the fixture answering for the raw key as the attacker's resolver is set up to. The file's third test is the control, the canonical spelling, on which both services ask for `alice.eth` and both return the victim. Its fourth test is the in-app control: `parseSearchInput('Alice.eth')`, the real header-search parser, returns `alice.eth`, so the search entry folds the same input the route entry does not.

The last test in the file is written to fail on the unpatched snapshot and pass with the patch applied:

```ts
describe('with normalize_profile_route_name.patch applied', () => {
  // Fails on the unpatched snapshot (the record read asks for the raw string);
  // passes once getProfileRecords canonicalises its input.
  it('the records service asks for the canonical name', async () => {
    const records = await getProfileRecords('Alice.eth')
    expect(world.asked.records).toEqual(['alice.eth'])
    expect(ethAddressOf(records)).toBe(world.VICTIM)
  })
})
```

Run from `apps/manager`:

```bash
./node_modules/.bin/vitest run \
  src/features/profile/service/profile_name_split_resolution.poc.test.ts --reporter=verbose
```

### File 2: the page, and what the copy button holds

Test A calls the real loader for the URL segment `Alice.eth` with a fresh query client, then the real `head()` with what the loader returned, and lists every profile-scoped query the loader left in the cache by the name it is filed under. The real `profileOwnerQuery` and `profileRecordsQuery` run underneath, on the same fixtures as file 1. Verbatim:

```tsx
  it('A. the loader keeps the page: owner read on one node, records on the other, all cached under the raw name', async () => {
    const queryClient = freshClient()
    const result = await loader({
      params: { name: 'Alice.eth' },
      context: { queryClient },
    })
    const tags = head({ params: { name: 'Alice.eth' }, loaderData: result })
    const title = tags.meta.find((tag) => 'title' in tag)?.title
    const ogDescription = tags.meta.find(
      (tag) => tag.name === 'og:description',
    )?.content
    const cached = cachedProfileNames(queryClient)

    console.log(
      [
        'route param               : Alice.eth',
        `loader fallback           : ${String(result.fallback)}`,
        `owner service asked for   : ${world.asked.ownerV2.join(', ')}`,
        `records service asked for : ${world.asked.records.join(', ')}`,
        `record index asked for    : ${JSON.stringify(world.asked.index)}`,
        `profile queries cached    : ${cached.join(', ')}`,
        `document title            : ${title}`,
        `og:description            : ${ogDescription}`,
      ].join('\n'),
    )

    // The page renders as a normal profile (no fallback card, no redirect).
    expect(result.fallback).toBeUndefined()
    // The owner check that let it through ran on the canonical node.
    expect(world.asked.ownerV2).toEqual(['alice.eth'])
    // The records it will render were requested from the raw node's resolver.
    expect(world.asked.records).toEqual(['Alice.eth'])
    expect(world.asked.index).toContainEqual({ id: 'Alice.eth' })
    // The whole cache: every profile query the page will read is filed under
    // the raw segment, and nothing is filed under the name the owner came from.
    expect(cached).toContain('get_records:Alice.eth')
    expect(cached).toContain('owner:Alice.eth')
    expect(cached.every((entry) => entry.endsWith(':Alice.eth'))).toBe(true)
    expect(cached.some((entry) => entry.endsWith(':alice.eth'))).toBe(false)
    // The raw node's text record is what the page publishes as its description.
    expect(result.description).toBe('send funds here')
    expect(title).toBe('Alice.eth - ENS Profile')
    expect(ogDescription).toBe('send funds here')
  })
```

The page component reads by exactly these keys (`ProfileView.tsx:93-96` and `:99-101`), so this is what it renders from. The description, title and `og:description` the loader publishes are the raw node's.

Test B loads the same URL, reads the loader's cache back the way the page component does (records at `ProfileView.tsx:93-96`, owner at `:99-101`, the owner's primary name at `:116-118`, under the keys `profileRecords.ts:167`, `profileOwner.ts:104` and `profileReverseName.ts:45` write them with), and renders the two children the page hands those values to: `ProfileHeader` (`ProfileView.tsx:192-211`, the heading, the Owner row and the About section) and `ProfileCards` (`:213-217`, the address card among others). It then reads the screen and clicks the two copy buttons. Verbatim:

```tsx
  it('B. the page shows the owner of alice.eth over the receiving address of Alice.eth, and copies the latter', async () => {
    const queryClient = freshClient()
    await loader({ params: { name: 'Alice.eth' }, context: { queryClient } })
    const { container } = mountPage('Alice.eth', queryClient)

    const row = ownerRow()
    // The "Main receiving address" card is the copy button whose `title` is its
    // value (ProfileAddressesSection.tsx:90-95, CopyableButton.tsx:47).
    const card = screen.getByTitle(world.ATTACKER)

    console.log(
      [
        `heading                   : ${screen.getByRole('heading', { level: 1 }).textContent}`,
        `Owner row                 : ${row.textContent}`,
        `description               : ${screen.getByText('send funds here').textContent}`,
        `receiving address card    : ${card.textContent}`,
        `card copy value (title)   : ${card.getAttribute('title')}`,
      ].join('\n'),
    )

    // One page, one identity: the heading is the segment as typed.
    expect(screen.getByRole('heading', { level: 1 }).textContent).toBe(
      'Alice.eth',
    )
    // The Owner row answers for the canonical node.
    expect(within(row).getByText(truncateAddress(world.VICTIM))).toBeTruthy()
    // The description is the raw node's; the receiving address is what the
    // raw node's resolver answered.
    expect(screen.getByText('send funds here')).toBeTruthy()
    expect(screen.getByText('Main receiving address')).toBeTruthy()
    expect(within(card).getByText('Alice.eth')).toBeTruthy()
    expect(within(card).getByText(truncateAddress(world.ATTACKER))).toBeTruthy()
    // Whole document: the victim's address appears once (the Owner row), the
    // attacker's once (the card), and no copy button carries the victim's.
    const text = container.textContent ?? ''
    expect(text.split(truncateAddress(world.VICTIM)).length - 1).toBe(1)
    expect(text.split(truncateAddress(world.ATTACKER)).length - 1).toBe(1)
    expect(screen.queryByTitle(world.VICTIM)).toBeNull()

    // The consumer half: what the visitor copies. The receiving-address card
    // hands the raw node's resolver's answer to the clipboard; the Owner row's own copy
    // button (ProfileDetails.tsx:30-48, :113-123) hands over the canonical
    // node's owner. Two nodes, one page, both copyable.
    fireEvent.click(card)
    await waitFor(() => expect(writeText).toHaveBeenCalledWith(world.ATTACKER))
    fireEvent.click(
      within(row).getByRole('button', { name: 'Copy to clipboard' }),
    )
    await waitFor(() => expect(writeText).toHaveBeenCalledWith(world.VICTIM))
    expect(writeText.mock.calls.map(([value]) => value)).toEqual([
      world.ATTACKER,
      world.VICTIM,
    ])
  })
```

The rendered state test B asserts is the finding. One heading, `Alice.eth`. Under it an Owner row showing `0x1111...1111`, the owner of `alice.eth`, with its own copy button. Under that the description the attacker wrote, and a card headed "Main receiving address" carrying `Alice.eth`, `0x2222...2222` and, as the copy button's value, the attacker's full address. The last block executes what the visitor does: clicking the card hands the attacker's address to `navigator.clipboard.writeText` (`CopyableButton.tsx:34-39`, `lib/clipboard.ts:2`), and clicking the Owner row's button hands over the victim's.

What test B does not mount is the page shell around those two children. `ProfileView` (`ProfileView.tsx:80-241`) wraps them in the wallet-connection, feature-flag and smart-account hooks that decide whether the visitor is the owner, the avatar and banner URLs, the grace and migration banners and the action bar. None of those reads the owner or the records under any other key, and none decides which node either comes from; the only branch that withholds the page is `ownerMissing` (`:120-137`), which is false here because the owner check on the canonical node found an owner. The route component renders `ProfileView` whenever the loader returned no fallback (`index.tsx:165-174`), which test A measures.

Test C is the control on the canonical URL: the loader keeps the page, both services ask for `alice.eth`, every cached profile query is filed under `alice.eth`, the description is the real one, the same two children render the victim's address in both the Owner row and the receiving-address card, the attacker's address appears nowhere on the page, and clicking the card copies the victim's address.

The file's last test is the counterfactual and is written to fail on the unpatched snapshot: with the patch, the loader rejects with a redirect to `/alice.eth` before either service is asked for anything and before anything is cached.

```tsx
describe('with normalize_profile_route_name.patch applied', () => {
  // Fails on the unpatched snapshot; passes once the loader redirects a
  // non-canonical segment to its ENSIP-15 form before any read.
  it('the loader redirects Alice.eth to alice.eth before reading anything', async () => {
    const queryClient = freshClient()
    await expect(
      loader({ params: { name: 'Alice.eth' }, context: { queryClient } }),
    ).rejects.toMatchObject({
      redirectedTo: { to: '/$name', params: { name: 'alice.eth' } },
    })
    expect(world.asked.ownerV2).toEqual([])
    expect(world.asked.records).toEqual([])
    expect(cachedProfileNames(queryClient)).toEqual([])
  })
})
```

Run from `apps/manager`:

```bash
./node_modules/.bin/vitest run \
  'src/routes/$name/profile_route_loader_raw_node.poc.test.tsx' --reporter=verbose
```

### File 3: the same crossing in the Explorer

The Explorer's name index page runs an ENSIP-15 validity gate (`apps/portal/src/routes/$name/index.tsx:442`, `:458-460`); its layout uses that function only inside a TLD-redirect condition (`routes/$name.tsx:29`), and the `address`, `records`, `subnames`, `history` and `resolver` subpages under the same segment read the parameter and never call it, so `/Alice.eth/address` reads owner and records on the raw string (`address.tsx:91-100`). The reverse-match badge on that subpage compares names case-folded (`useReverseMatch.ts:79-81`), so a reverse record spelled `Alice.eth` matches the segment. The file measures where the gate is and is not applied, verbatim:

```ts
  it('the index page gates on it; the layout calls it only to redirect TLDs, and the subpages never call it', () => {
    const index = read('./index.tsx')
    const layout = read('../$name.tsx')
    const subpages = {
      'address.tsx': read('./address.tsx'),
      'records.tsx': read('./records.tsx'),
      'subnames.tsx': read('./subnames.tsx'),
      'history.tsx': read('./history.tsx'),
      'resolver.tsx': read('./resolver.tsx'),
    }

    // index.tsx:442 `const isValidName = isValidEnsName(name)` and :458-459
    // `<InvalidNameMessage title="Invalid name" />`.
    expect(index).toContain('const isValidName = isValidEnsName(name)')
    expect(index).toContain('<InvalidNameMessage title="Invalid name" />')

    // The layout's only use is inside the TLD redirect condition
    // ($name.tsx:29 `if (isValidEnsName(name) && isTLD(name))`); it never
    // gates rendering.
    expect(layout).toContain('if (isValidEnsName(name) && isTLD(name))')
    expect(countGateUses(layout.replace('isValidEnsName(name) && isTLD(name)', ''))).toBe(1) // the import line

    const table = Object.entries(subpages).map(([file, source]) => ({
      file,
      gateUses: countGateUses(source),
      readsParam: /useParams\(/.test(source),
    }))
    console.table(table)
    for (const row of table) {
      expect(row.readsParam).toBe(true)
      expect(row.gateUses).toBe(0)
    }
  })
```

This variant is corroborating, not the headline: the Explorer reads the owner on the raw string and its records through the same library (`apps/portal/src/features/profile/hooks/useRecords.ts:25`), so its resolver and text records are the raw node's and its address records are requested under the canonical hash from that resolver, as on the Manager; the difference is that its Owner row is the raw node's too. Run from `apps/portal`:

```bash
./node_modules/.bin/vitest run \
  'src/routes/$name/name_subpages_skip_validity_gate.poc.test.ts' --reporter=verbose
```

### Attack steps: reproducing this by hand

Files 1 and 2 automate steps 3 and 4. Steps 1 and 2 are the attacker's on-chain setup, which this report does not perform against any deployment.

1. **Pick a registered name.** `alice.eth`, registered on Sepolia by its real owner, with an ETH address and a profile set.
2. **Register the raw twin.** This step is the precondition in step 2 of the report: the oracle's validity rule and the registry's key computation are read from the verified source published for those two addresses, the definition of that key from `LibLabel.sol` at commit `48b3e2d3`, and the registrar that calls them from that same commit, and the resolver's answer is argued from its ABI and the applications' own use of it. No registration was made for this report. As the attacker, send `commit` and then `register` to the ETHRegistrar for the label `Alice`, exactly those bytes, with the attacker as owner and a resolver the attacker controls as the `resolver` argument. On that resolver, set the answer for `addr(namehash('alice.eth'))`, the query the page will send it, to the attacker's address (a fixed-address contract, or the flow's own resolver written under that node, as step 2 of the report describes), and set `avatar` and `description` under `namehash('Alice.eth')` to copies of the real profile's values.
3. **Open the Manager's profile URL `/Alice.eth`**, or the share form `/p/Alice.eth`, which redirects to the same route with the segment unchanged.
4. **Read the page.** The heading is `Alice.eth`. The Owner row is the real owner of `alice.eth` (or their primary name). The expiry is the real expiry. The description and avatar are the copies. Under "Main receiving address" is the attacker's address in a copy button whose hover title is the full attacker address.
5. **Copy and send.** The visitor copies the address and pays it.

A visitor who instead types `Alice.eth` into the header search lands on `/alice.eth` and sees one node throughout; the search folds case before it navigates (`searchSuggestions.utils.ts:54`).

### The counterfactual: the fix patch

Canonicalise the segment once, before any read. Save as `normalize_profile_route_name.patch` and apply from the repository root with `patch -p1`. It applied to the snapshot without rejects on the executed run (see the capture). The block below is the whole file, byte for byte.

```diff
Canonicalise the profile route's name before any read.

Two hunks that matter. The first, in the route loader, redirects a
non-canonical `$name` segment to its ENSIP-15 form before the owner or the
records are read, so one name can only ever be one node on the page. The
second makes the record read itself refuse a non-canonical name, which closes
the same crossing for every other caller of `profileRecordsQuery`. The helper
both use lives beside the existing normalisers in profileName.ts.

--- a/apps/manager/src/features/profile/service/profileName.ts
+++ b/apps/manager/src/features/profile/service/profileName.ts
@@ -71,3 +71,10 @@ export const normalizeEth2LdName = (name: string): Eth2LdName | null => {
     name: ethName.name,
   }
 }
+
+/**
+ * The one spelling of a name the profile surfaces are allowed to read: its
+ * ENSIP-15 form, or null when it has none.
+ */
+export const canonicalProfileName = (name: string): string | null =>
+  normalizeEthName(name)?.name ?? normalizeDnsName(name)
--- a/apps/manager/src/features/profile/service/profileRecords.ts
+++ b/apps/manager/src/features/profile/service/profileRecords.ts
@@ -20,6 +20,7 @@ import {
   textRecords,
 } from '../data/records'
 import { DEBUG_PROFILE } from '../MOCK'
+import { canonicalProfileName } from './profileName'
 
 class GetProfileRecordsError extends TaggedError('GetProfileRecordsError')<{
   cause: unknown
@@ -80,8 +81,8 @@ const safeGetCoderFromCoin = fromThrowable(getCoderFromCoin, () => undefined)
 const isSupportedCoinType = (coinType: number): boolean =>
   safeGetCoderFromCoin(coinType).isOk()
 
-export const getProfileRecords = ResultFn(async function* (name: string) {
-  if (name === 'debug') {
+export const getProfileRecords = ResultFn(async function* (input: string) {
+  if (input === 'debug') {
     return ok({
       ...DEBUG_PROFILE,
       _rawSubgraphRecords: {
@@ -91,6 +92,15 @@ export const getProfileRecords = ResultFn(async function* (name: string) {
     })
   }
 
+  // Read records on the canonical node only. The owner path already
+  // normalises (profileOwner.ts); without this the two paths can name two
+  // different registrations for one string.
+  const name = canonicalProfileName(input)
+  if (!name) {
+    return yield* new GetProfileRecordsError({
+      cause: new Error(`"${input}" is not a normalizable ENS name`),
+    })
+  }
   const client = yield* safeGetClient()
   const indexerDomain = yield* fromPromise(
     fetchIndexerDomain(name),
--- a/apps/manager/src/routes/$name/index.tsx
+++ b/apps/manager/src/routes/$name/index.tsx
@@ -13,6 +13,7 @@ import { ProfileLoading } from '@/features/profile/components/view/ProfileLoadin
 import { ProfileView } from '@/features/profile/components/view/ProfileView'
 import { dnsSecEnabledQuery } from '@/features/profile/service/dnsSecEnabled'
 import { profileExpiryQuery } from '@/features/profile/service/profileExpiry'
+import { canonicalProfileName } from '@/features/profile/service/profileName'
 import { profileOwnerQuery } from '@/features/profile/service/profileOwner'
 import { profileRecordsQuery } from '@/features/profile/service/profileRecords'
 import { profileRegistrationQuery } from '@/features/profile/service/profileRegistration'
@@ -48,6 +49,20 @@ const classifyMissingName = (
 
 export const Route = createFileRoute('/$name/')({
   loader: async ({ params: { name }, context: { queryClient } }) => {
+    // One name, one node. A segment that is not in ENSIP-15 form is sent to
+    // its canonical spelling before anything is read, so the owner and the
+    // records on this page can never come from two different registrations.
+    const canonical = canonicalProfileName(name)
+    if (canonical === null) {
+      return { fallback: 'not-found' as const, description: undefined }
+    }
+    if (canonical !== name) {
+      throw redirect({
+        params: { name: canonical },
+        to: '/$name',
+        replace: true,
+      })
+    }
     const [profileRecords, ownerData] = await Promise.all([
       queryClient.ensureQueryData(profileRecordsQuery(name)),
       // Fetch, not ensure: `ensureQueryData` serves invalidated data, so a name
# end of patch
```

What the patch flips, observed on the patched run (runs 4 and 5 in the capture): with the patch applied, file 1's second test fails at `expect(world.asked.records).toEqual(['Alice.eth'])`, because the record service now asks for `alice.eth`, and its last test passes. File 2's tests A and B fail at the loader call, which now rejects with a redirect before either service is asked for anything, and its last test passes; its test C measures the canonical spelling and passes identically both ways, as does file 1's test 3. File 3 is unaffected: the patch is to the Manager.

### What is measured, and what is argued

| Claim                                                                                                                                                                                                                                                                                                                                                                                                | How it is established                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| For the URL `/Alice.eth`, one rendered page shows the owner of `alice.eth` in its Owner row and the address the resolver of `Alice.eth` answered under "Main receiving address"; the card's copy button hands the attacker's address to the clipboard and the Owner row's button hands over the victim's; each address appears once on the page and no copy button carries the victim's full address | Executed green, file 2 test B (run 2 in the capture). The address is the fixture's answer for the raw key, standing in for the attacker's resolver (the resolver row below)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| The real loader for `/Alice.eth` renders a normal profile (no fallback, no redirect) because its owner check ran on `alice.eth`; every profile query it leaves in the cache is filed under `Alice.eth` and none under `alice.eth`; it publishes the raw node's description as the page description and `og:description`                                                                              | Executed green, file 2 test A (run 2)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| For one input string the real owner service asks the chain for `alice.eth` and the real record service asks for `Alice.eth`, the v1 fallback is never consulted, and those two are the only strings the chain is asked for; the owner returned is the victim's and the resolver, description and ETH address returned are the attacker's                                                             | Executed green, file 1 test 2 (run 1). The ETH address is the fixture's answer for the raw key, standing in for the attacker's resolver (the resolver row below)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| On the canonical spelling both services ask for the same node, the cache is filed under it, both return the victim, the page renders one node and copies the victim's address (control)                                                                                                                                                                                                              | Executed green, file 1 test 3 and file 2 test C (runs 1 and 2)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| The header search folds the same input before navigating, so the route parameter is the entry that reaches the record read unfolded                                                                                                                                                                                                                                                                  | Executed green, file 1 test 4 (run 1), against the real `parseSearchInput`. The other folding entries are cited from source (`name-parser.ts:46`, `profileName.ts:14-21`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `normalize('Alice.eth')` is `alice.eth` and the two namehashes differ, so the two spellings are two nodes                                                                                                                                                                                                                                                                                            | Executed green, file 1 test 1 and file 3 test 1 (runs 1 and 3), against viem; the two hash values in the report were computed with the same library                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| The patch redirects a raw segment before any read or cache write and makes the record service ask for the canonical name                                                                                                                                                                                                                                                                             | Executed: the last test in files 1 and 2 fails unpatched (runs 1 and 2) and passes patched (runs 4 and 5); the patch applied with no rejects (three hunks at offset -1)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| The Explorer gates only its index page and compares the reverse-match badge case-folded                                                                                                                                                                                                                                                                                                              | Executed green, file 3 (run 3), reading the route sources and calling the real `isValidEnsName`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| The page shell around the header and cards renders them whenever the loader returned no fallback, and adds no read that could change which node either comes from                                                                                                                                                                                                                                    | Argued from source: `index.tsx:165-174`, `ProfileView.tsx:80-241` (the `ownerMissing` branch at `:120-137`); test A measures the loader's decision                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| The expiry and registration state the page shows are read on the canonical name, so they answer for `alice.eth` alongside the Owner row                                                                                                                                                                                                                                                              | Argued from source: `profileExpiry.ts:77,94,106`, `profileRegistration.ts:66`; file 2 replaces both queries with inert data and renders neither                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| The client library selects the resolver by the name as typed and reads text, content-hash and ABI records under the hash of the name as typed; it requests address records under `namehash(normalize(name))` from that same resolver                                                                                                                                                                 | Argued from the pinned dependency's source: `resolveNameData.js:40,54`; `getText.js:7`, `getContentHash.js:8`, `getAbi.js:8`; `getAddress.js:3,33-35,43-45`; `getRecords.js:22-27`. Not measured: the fixture replaces `getRecords` above the point where the calldata is built                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| The attacker's resolver answers `addr(namehash('alice.eth'))` with the attacker's address                                                                                                                                                                                                                                                                                                            | Argued from source: the v2 `register` takes the resolver address as an argument (`v2/ethRegistrar` ABI; `registration.actors.ts:242-252`), so a contract returning a fixed address for every `addr` query is what the interface admits; that contract is not part of the audited code, and the registry imposes no requirement on it. The deployed registry stores the resolver argument as `entry.resolver` with no validation and returns it unchanged from `getResolver(label)` for any entry that has not expired (`PermissionedRegistry._register` stores it at `:460`; `getResolver` returns it at `:257-259`; verified source); no resolver is executed by this set                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| The registrar accepts the label `Alice` as a registration distinct from `alice`                                                                                                                                                                                                                                                                                                                      | Read from source. The oracle pinned for this deployment by the client library the applications use, `StandardRentPriceOracle` at `0x8914b66260EB8C4fff795650c3AE8Cd335958987` (`@ensdomains/ensjs`, `src/clients/l1.ts:140` for the Sepolia block, entries at `:174-199`; source verified on the block explorer, solc 0.8.27, 9 August 2026, byte-identical to `ensdomains/contracts-v2` at branch `main` as of 12 September 2026), defines `isValid(string label)` as `getBasePrice(label, 1) > 0` under the docstring "Does not check if normalized.", and `getBasePrice` applies no normalisation or case rule, returning 0 when `bytes(label).length` is 0 or above 255 and otherwise pricing from the deployment's rate entry for the label's codepoint count, so a label whose codepoint count maps to a zero rate would price at 0 and be rejected as invalid; a registration reaches that predicate through `getRegisterPrice` and `_requireBasePrice` (`:310`, `:400-409`), not through the `isValid` view helper. The registrar holds its oracle in storage and its owner can replace it (`AbstractETHRegistrar.sol:34,78-81`, from `ensdomains/contracts-v2` at commit `48b3e2d3`). `PermissionedRegistry._register` keys the entry on `LibLabel.id(label)`, which is `uint256(keccak256(bytes(label)))`, and reverts `LabelAlreadyRegistered` only for an unexpired entry under those exact bytes. No normalisation or case fold appears in either path. The v2 registrar and its rent price oracle are not part of the audited code; no registration was made against any network for this report |
| The copy button's hover title is its value, so the full attacker address is what the visitor sees and copies                                                                                                                                                                                                                                                                                         | Argued from source (`CopyableButton.tsx:47`); file 2 test B locates the card by that title and clicks it                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

### What this PoC does not establish

None of what follows sits under the impact entry this report files on.

1. **A registration of `Alice` on public Sepolia.** The attacker's setup is a commit, a registration and the resolver writes that follow them, every one of them state-changing, and this report sends no transaction to any deployment. The fixtures hold the two registrations the attack needs; the oracle's validity rule and the registry's key computation are read from the verified source published for those two addresses; the definition of that key and the registrar itself are read from `ensdomains/contracts-v2` at commit `48b3e2d3`.
2. **That the deployed Manager serves these exact bytes.** The programme states that code for in-scope assets is strictly frozen while the program is live; the tests run against the snapshot, and the file fingerprints in the report identify it.
3. **The avatar.** The page builds the avatar URL from the segment as typed (`ProfileView.tsx:139`), which would fetch the attacker's copy for `Alice.eth`; file 2 stubs the image leaves and does not measure it. The finding does not rest on it.
4. **The send.** The victim's payment happens wherever they paste the copied address, outside the Manager. What is measured stops at the string handed to the clipboard.
5. **The deployed record index's answer for an unknown id.** The fixtures answer `domain: null`, the shape the project's own e2e mock returns for a name it does not hold (`e2e/helpers/mock-indexer.ts:128-133`). An index that errored on an unknown id instead would fail the whole record query, and the page would show the route's error component rather than a profile. The ETH address never comes from that answer either way.
6. **The page shell.** The wallet, feature-flag and smart-account hooks around the two rendered children, the banners and the action bar are not mounted (see the paragraph under file 2); they are argued from source not to touch which node either read comes from.
7. **The library's calldata for the address.** The fixture replaces `getRecords` above the point where the calldata is built, so the tests do not measure that the ETH-address call carries `namehash('alice.eth')` to the resolver of `Alice.eth`; that is read from the library source cited in the table.
8. **The resolver's answer.** That the attacker's resolver returns the attacker's address for the canonical node is the precondition in step 2 of the report, argued from the registrar's `register` signature, the resolver's ABI and the applications' own use of it. No resolver contract is executed.

### The captures, and placing the files

The five runs (three unpatched, then the patch, then two patched) were executed in one sealed session on 2026-09-12: new user, network, pid, ipc, uts and cgroup namespaces with only `lo` present, the host filesystem read-only, a throwaway overlay over the snapshot so nothing was written to it, and four fail-closed gates before any test (no inherited descriptor, interfaces exactly `lo`, no socket file visible, TCP v4, TCP v6 and DNS probes all blocked). The terminal output is in the Gist as `CAPTURE.log`. The block below is an excerpt of it: the gate lines, and for every run the per-test lines, the console measurements, the failure assertion and the summary. Every line is byte-identical to the capture except one substitution and one removal, both disclosed here: the five vitest banners print the absolute path of the snapshot on the host, replaced with `<archive-root>` in both the Gist copy and this excerpt, and a block listing the SHA-256 of the four files as run was deleted from both. The excerpt is also abridged, as the selection above describes, without altering any line it keeps.

No line of any run mentions an outbound request.

```
=== NAMESPACE IDENTITY ===
node: v22.18.0
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
=== PLACEMENT (overlay only) ===
placed: apps/manager/src/features/profile/service/profile_name_split_resolution.poc.test.ts
placed: apps/manager/src/routes/$name/profile_route_loader_raw_node.poc.test.tsx
placed: apps/portal/src/routes/$name/name_subpages_skip_validity_gate.poc.test.ts
=== RUN 1/5: file 1, unpatched (apps/manager) ===
 RUN  v4.1.5 <archive-root>/apps/manager
 ✓ src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > the same two strings are two different nodes on chain 199ms
stdout | src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > the owner is read on the normalized name, the records on the raw one
URL segment                : Alice.eth
owner service asked for    : alice.eth
records service asked for  : Alice.eth
record index asked for     : [{"id":"Alice.eth"}]
owner rendered             : 0x1111111111111111111111111111111111111111
ETH address rendered       : 0x2222222222222222222222222222222222222222
resolver rendered          : 0x4444444444444444444444444444444444444444
 ✓ src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > the owner is read on the normalized name, the records on the raw one 4ms
 ✓ src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > control: the canonical spelling reads one node for both 1ms
 ✓ src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > control: the app itself folds case everywhere else it takes a name 0ms
 × src/features/profile/service/profile_name_split_resolution.poc.test.ts > with normalize_profile_route_name.patch applied > the records service asks for the canonical name 7ms
 FAIL  src/features/profile/service/profile_name_split_resolution.poc.test.ts > with normalize_profile_route_name.patch applied > the records service asks for the canonical name
AssertionError: expected [ 'Alice.eth' ] to deeply equal [ 'alice.eth' ]
 Test Files  1 failed (1)
      Tests  1 failed | 4 passed (5)
vitest exit code: 1
=== RUN 2/5: file 2, unpatched (apps/manager) ===
 RUN  v4.1.5 <archive-root>/apps/manager
stdout | src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > /$name profile route, URL segment Alice.eth > A. the loader keeps the page: owner read on one node, records on the other, all cached under the raw name
route param               : Alice.eth
loader fallback           : undefined
owner service asked for   : alice.eth
records service asked for : Alice.eth
record index asked for    : [{"id":"Alice.eth"}]
profile queries cached    : expiry:Alice.eth, get_records:Alice.eth, owner:Alice.eth, registration:Alice.eth
document title            : Alice.eth - ENS Profile
og:description            : send funds here
stdout | src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > /$name profile route, URL segment Alice.eth > B. the page shows the owner of alice.eth over the receiving address of Alice.eth, and copies the latter
heading                   : Alice.eth
Owner row                 : key_verticalOwner0x1111...1111
description               : send funds here
receiving address card    : Alice.eth0x2222...2222
card copy value (title)   : 0x2222222222222222222222222222222222222222
 ✓ src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > /$name profile route, URL segment Alice.eth > A. the loader keeps the page: owner read on one node, records on the other, all cached under the raw name 216ms
 ✓ src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > /$name profile route, URL segment Alice.eth > B. the page shows the owner of alice.eth over the receiving address of Alice.eth, and copies the latter 72ms
 ✓ src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > /$name profile route, URL segment Alice.eth > C. control: the canonical URL renders one node throughout and copies its own address 17ms
 × src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > with normalize_profile_route_name.patch applied > the loader redirects Alice.eth to alice.eth before reading anything 13ms
 FAIL  src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > with normalize_profile_route_name.patch applied > the loader redirects Alice.eth to alice.eth before reading anything
AssertionError: promise resolved "{ fallback: undefined, …(1) }" instead of rejecting
 Test Files  1 failed (1)
      Tests  1 failed | 3 passed (4)
vitest exit code: 1
=== RUN 3/5: file 3, unpatched (apps/portal) ===
 RUN  v4.1.5 <archive-root>/apps/portal
 ✓ src/routes/$name/name_subpages_skip_validity_gate.poc.test.ts > Explorer name routes, one URL segment > the validity gate rejects the raw spelling and the two spellings are two nodes 234ms
stdout | src/routes/$name/name_subpages_skip_validity_gate.poc.test.ts > Explorer name routes, one URL segment > the index page gates on it; the layout calls it only to redirect TLDs, and the subpages never call it
┌─────────┬────────────────┬──────────┬────────────┐
│ (index) │ file           │ gateUses │ readsParam │
├─────────┼────────────────┼──────────┼────────────┤
│ 0       │ 'address.tsx'  │ 0        │ true       │
│ 1       │ 'records.tsx'  │ 0        │ true       │
│ 2       │ 'subnames.tsx' │ 0        │ true       │
│ 3       │ 'history.tsx'  │ 0        │ true       │
│ 4       │ 'resolver.tsx' │ 0        │ true       │
└─────────┴────────────────┴──────────┴────────────┘
 ✓ src/routes/$name/name_subpages_skip_validity_gate.poc.test.ts > Explorer name routes, one URL segment > the index page gates on it; the layout calls it only to redirect TLDs, and the subpages never call it 5ms
 ✓ src/routes/$name/name_subpages_skip_validity_gate.poc.test.ts > Explorer name routes, one URL segment > the address subpage badge compares the reverse name case-folded 0ms
 Test Files  1 passed (1)
      Tests  3 passed (3)
vitest exit code: 0
=== APPLY PATCH (overlay only) ===
patching file apps/manager/src/features/profile/service/profileName.ts
patching file apps/manager/src/features/profile/service/profileRecords.ts
Hunk #1 succeeded at 19 (offset -1 lines).
Hunk #2 succeeded at 80 (offset -1 lines).
Hunk #3 succeeded at 91 (offset -1 lines).
patching file 'apps/manager/src/routes/$name/index.tsx'
patch exit code: 0
=== RUN 4/5: file 1, patched (apps/manager) ===
 RUN  v4.1.5 <archive-root>/apps/manager
 ✓ src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > the same two strings are two different nodes on chain 231ms
stdout | src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > the owner is read on the normalized name, the records on the raw one
URL segment                : Alice.eth
owner service asked for    : alice.eth
records service asked for  : alice.eth
record index asked for     : [{"id":"alice.eth"}]
owner rendered             : 0x1111111111111111111111111111111111111111
ETH address rendered       : 0x1111111111111111111111111111111111111111
resolver rendered          : 0x3333333333333333333333333333333333333333
 × src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > the owner is read on the normalized name, the records on the raw one 14ms
 ✓ src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > control: the canonical spelling reads one node for both 1ms
 ✓ src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > control: the app itself folds case everywhere else it takes a name 1ms
 ✓ src/features/profile/service/profile_name_split_resolution.poc.test.ts > with normalize_profile_route_name.patch applied > the records service asks for the canonical name 1ms
 FAIL  src/features/profile/service/profile_name_split_resolution.poc.test.ts > profile route input, one URL segment > the owner is read on the normalized name, the records on the raw one
AssertionError: expected [ 'alice.eth' ] to deeply equal [ 'Alice.eth' ]
 Test Files  1 failed (1)
      Tests  1 failed | 4 passed (5)
vitest exit code: 1
=== RUN 5/5: file 2, patched (apps/manager) ===
 RUN  v4.1.5 <archive-root>/apps/manager
 × src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > /$name profile route, URL segment Alice.eth > A. the loader keeps the page: owner read on one node, records on the other, all cached under the raw name 225ms
 × src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > /$name profile route, URL segment Alice.eth > B. the page shows the owner of alice.eth over the receiving address of Alice.eth, and copies the latter 2ms
 ✓ src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > /$name profile route, URL segment Alice.eth > C. control: the canonical URL renders one node throughout and copies its own address 61ms
 ✓ src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > with normalize_profile_route_name.patch applied > the loader redirects Alice.eth to alice.eth before reading anything 2ms
 FAIL  src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > /$name profile route, URL segment Alice.eth > A. the loader keeps the page: owner read on one node, records on the other, all cached under the raw name
Serialized Error: { redirectedTo: { params: { name: 'alice.eth' }, to: '/$name', replace: true } }
 FAIL  src/routes/$name/profile_route_loader_raw_node.poc.test.tsx > /$name profile route, URL segment Alice.eth > B. the page shows the owner of alice.eth over the receiving address of Alice.eth, and copies the latter
Serialized Error: { redirectedTo: { params: { name: 'alice.eth' }, to: '/$name', replace: true } }
 Test Files  1 failed (1)
      Tests  2 failed | 2 passed (4)
vitest exit code: 1
```

Placing the files:

```bash
cp profile_name_split_resolution.poc.test.ts     apps/manager/src/features/profile/service/
cp profile_route_loader_raw_node.poc.test.tsx    'apps/manager/src/routes/$name/'
cp name_subpages_skip_validity_gate.poc.test.ts  'apps/portal/src/routes/$name/'
```
