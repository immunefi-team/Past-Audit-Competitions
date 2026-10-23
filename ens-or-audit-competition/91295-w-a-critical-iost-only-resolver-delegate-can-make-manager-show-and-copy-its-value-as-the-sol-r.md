# 91295 w a critical iost only resolver delegate can make manager show and copy its value as the sol receiving address

**Submitted on Aug 31st 2026 at 11:46:17 UTC by @crypt0n1337 for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91295
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Executive summary

Manager treats address records for different chains as the same receiving address whenever their rendered strings are equal after lowercasing. That is a valid shortcut for case-insensitive hex representations, but it is invalid for case-sensitive Base58.

IOST (coin type 291) and SOL (coin type 501) are both supported as 32-byte Base58-without-checksum records. The following two canonical addresses are different 32-byte Ed25519 points but collapse under Manager's comparison:

```
Attacker-chosen IOST 291 A:
9c6hybhQ6Aycep9jaUnP6uL9ZYvDjUp1aSkFWPUFjtpj
raw = 7fdb46ac3f42d5b3f6617444d7bf34eb786e0ff1ccfae44188eaf10bad4f044c

Legitimate SOL 501 B:
9C6hybhQ6Aycep9jaUnP6uL9ZYvDjUp1aSkFWPUFJtpj
raw = 79b5562e8fe654f94078b112e8a98ba7901f853ae695bed7e0e3910bad049664
```

The strings differ only at two case positions, so `A.toLowerCase() === B.toLowerCase()`, while `decodeIOST(A) !== decodeSOL(B)`. Independent `solders` and libsodium/PyNaCl checks confirm that both are valid on-curve, prime-subgroup Ed25519 points. `B` is derived in the PoC from deterministic test seed bytes `01 02 ... 20`, proving that it is a normal, victim-controlled SOL public key rather than arbitrary text. No real secret is used.

On a profile with no ETH address record, Manager's production address order places IOST 291 before SOL 501. Manager therefore selects `A` as the main address, groups both the IOST and SOL icons onto that row, removes `B` from the chain-specific list, and makes the row's Copy button write `A` to the clipboard. A payer who relies on the SOL icon and sends native SOL to the copied address sends to valid but unintended destination `A`, while the legitimate SOL record `B` remains unchanged.

The attacker does not need permission to change the SOL record. The companion PermissionedResolver test proves that an account with only the documented IOST-291 `ROLE_SET_ADDR` part permission:

1. cannot write coin type 501;
2. has no name-wide address role;
3. can write coin type 291; and
4. leaves the legitimate SOL bytes unchanged.

This report does **not** claim that the attacker knows `A`'s private key or can steal the transferred SOL. The demonstrated impact is the exact scoped wrong-recipient consequence: Manager labels an attacker-chosen value with a SOL icon and copies that value instead of the legitimate SOL destination. Under the standard Ed25519 discrete-log assumption, the victim's key for `B` gives no signing authority for unrelated point `A`.

## Affected code and production path

The tested Manager revision is:

```
cda79acaad59711b943fc68207ebb3f1d0ff8596
```

### 1. Address request order selects IOST first when ETH is absent

`apps/manager/src/features/profile/data/records/address.tsx:42-48` sorts all non-preferred records numerically by coin type. IOST 291 and SOL 501 are not in the preferred list, so IOST precedes SOL.

`apps/manager/src/features/profile/service/profileRecords.ts:116-120` passes `addressRecords` coin types to ENSjs in that order. ENSjs constructs and processes coin calls in the supplied order; empty results are omitted. `apps/manager/src/features/profile/utils/transformRecords.ts:170-179` assigns the resulting `profile.coins` array directly to `records.addresses` without reordering it.

`apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:229-236` selects ETH if it exists and otherwise selects `addresses[0]`. In the demonstrated two-record state, ETH is explicitly absent and IOST 291 is the first record.

### 2. Cross-codec case folding merges two distinct destinations

`apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:251-260` computes `mainAddress.value.toLowerCase()` and includes every record with the same lowercase string in the main row's chain-icon list.

`apps/manager/src/features/profile/components/view/ProfileView.helpers.ts:268-276` uses the same lowercase projection to remove matching records from the chain-specific address list. It therefore removes the legitimate SOL record `B`.

### 3. The production component copies A while showing a SOL icon

`apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:90-123` gives the main row's `CopyableButton` exactly `address.value`, while rendering the chain icons supplied by the grouping helper.

`apps/manager/src/features/profile/components/view/ProfileAddressesSection.tsx:159-178` wires the three affected helpers into the real profile view. The component PoC asserts the rendered IOST and SOL icon asset URLs, absence of `B` as a copy target, and clipboard call with `A`.

## Documented permission model and attacker boundary

This is an application bug, not a claim that the contracts incorrectly authorize Mallory.

ENS's official v2 overview describes per-account Permissioned Resolvers with fine-grained per-record permissions and identifies per-record roles as a standard resolver feature:

* <https://docs.ens.domains/ensv2/overview/>

The official indexing guide documents `NamedAddrResource` as the permission resource for a specific address coin type and says resolver role changes occur through `authorize(Name|Text|Addr)Roles()`:

* <https://docs.ens.domains/ensv2/indexing/>

The official app-developer guide says subname records may be delegated per name or per record key through the resolver's `authorize*Roles` functions:

* <https://docs.ens.domains/ensv2/tutorial-app-developers/>

The contract source at revision `48b3e2d39513b9dd32ef1850877a29009bc807b9` implements exactly that boundary:

* `PermissionedResolver.sol:362-387`: `authorizeAddrRoles(toName, coinType, account, grant)` grants a coin-specific part resource;
* `PermissionedResolver.sol:660-671`: `setAddr` checks `partHash(coinType)` and `ROLE_SET_ADDR`.

The companion Forge PoC deploys the real resolver implementation and demonstrates that Mallory has the IOST resource only. The contract is used only to establish a realistic input state and actor boundary; the reported vulnerable asset remains Manager app Files.

## Attack sequence

{% stepper %}
{% step %}

## Create a profile with no ETH record

A name uses a standard ENSv2 PermissionedResolver and has no ETH address record. The name's legitimate SOL-501 record is `B`.
{% endstep %}

{% step %}

## Grant the limited IOST role

In a documented fine-grained delegation state, Mallory receives only the IOST-291 address-record role. Mallory has neither the SOL-501 part role nor a name-wide address role.
{% endstep %}

{% step %}

## Write the authorized IOST record

Mallory reads public `B`, changes two case characters to obtain canonical IOST value `A`, and writes only the authorized IOST-291 part.
{% endstep %}

{% step %}

## Open the Manager profile

A payer opens the name's normal Manager profile. The production record order is `[291, 501]`; with no ETH, Manager selects IOST `A` as the main receiving value.
{% endstep %}

{% step %}

## Observe the merged records

Case-fold grouping treats SOL `B` as equal to IOST `A`. The main row displays both IOST and SOL icons, and the separate SOL row containing `B` disappears.
{% endstep %}

{% step %}

## Copy the displayed receiving address

The payer clicks the normal Copy button for the main receiving address. Manager copies `A`.
{% endstep %}

{% step %}

## Send through an external Solana wallet

If the payer pastes that value into a Solana wallet and sends, the wallet receives a valid on-curve public key, but it is not the intended SOL destination `B`.
{% endstep %}
{% endstepper %}

There is no exploit-time mutation of the SOL record and no abnormal action inside Manager. The final send is an external wallet action after the normal profile-view/copy workflow.

## Reproduction

### Manager component PoC

Attachment:

```
apps/manager/audit-artifacts/profile-address-casefold-role-boundary-20260831/ProfileAddressCaseFoldRoleBoundary.poc.test.tsx
```

From `apps/manager`:

```powershell
.\node_modules\.bin\vitest.CMD run --config vitest.config.ts audit-artifacts/profile-address-casefold-role-boundary-20260831/ProfileAddressCaseFoldRoleBoundary.poc.test.tsx
```

Expected:

```
Test Files  1 passed (1)
Tests       2 passed (2)
```

The positive test proves all of the following at the production component boundary:

* the deterministic victim seed derives exact SOL `B`;
* IOST `A` and SOL `B` decode to distinct 32-byte values;
* there is no ETH record;
* production configuration orders IOST 291 before SOL 501;
* IOST becomes the main address;
* both actual IOST and SOL icon assets render on the main row;
* the legitimate SOL `B` is absent as a copy target; and
* clicking the real CopyableButton writes IOST `A`.

The negative test changes the SOL value so it no longer case-collides. It then proves that the SOL-specific card stays visible and copies its own SOL value.

### Address and curve verification

Attachments:

```
apps/manager/audit-artifacts/profile-address-casefold-role-boundary-20260831/verify-addresses.mjs
apps/manager/audit-artifacts/profile-address-casefold-role-boundary-20260831/verify-addresses.py
```

From the Manager repository root:

```powershell
node apps/manager/audit-artifacts/profile-address-casefold-role-boundary-20260831/verify-addresses.mjs
python apps/manager/audit-artifacts/profile-address-casefold-role-boundary-20260831/verify-addresses.py
```

The JavaScript verifier uses the production address encoder, derives `B` from deterministic seed bytes, and includes a self-contained compressed-Ed25519 point check. The Python verifier cross-checks both primary points with `solders.Pubkey.is_on_curve()` and libsodium's `crypto_core_ed25519_is_valid_point()`. It also builds, entirely offline, a standard one-lamport Solana System Program transfer instruction whose writable recipient is `A`, with no RPC, signing, simulation, or broadcast. A lower-equal off-curve control shows that the validators distinguish the cases.

### Fine-grained role-boundary PoC

Attachment:

```
test/audit-artifacts/ManagerProfileCaseFoldRoleBoundary.t.sol
```

From the `ens-contracts-v2/contracts` repository:

```powershell
forge test --match-contract ManagerProfileCaseFoldRoleBoundaryTest -vv
```

Expected:

```
[PASS] test_iostOnlyDelegateCannotWriteSolButCanPlantCaseFoldedIost()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

## Impact and severity rationale

The competition explicitly classifies a wrong address displayed or prefilled by the application, leading a user's funds or transaction to an unintended recipient, as Critical.

Source: <https://immunefi.com/audit-competition/audit-competition-ens/scope/>

The PoC reaches the precise rendering and copy sink described by that impact. Manager shows a SOL icon for `A`, hides the true SOL destination `B`, and copies `A`. This materially changes the destination of a later native-SOL transfer.

This is not being classified as “direct theft of funds.” The attacker is not shown to control `A`, and the report does not rely on that. The severity claim is instead the program's separately enumerated wrong-address/unintended-recipient Critical impact.

## Strongest rejection cases and residual assumptions

1. **The direct per-coin role grant is a protocol-level setup.** The frozen Manager UI does not currently expose a fine-grained `authorizeAddrRoles` control. Triage may regard the state as uncommon or externally created. The counterpoint is that fine-grained per-record delegation is an explicit, documented ENSv2 feature, and the PoC uses the unmodified production resolver implementation.
2. **A later send happens outside Manager.** The component PoC stops after the exact clipboard write and intentionally sends no funds. A payer must paste and approve a normal Solana transfer in a wallet. The scoped Critical wording nevertheless expressly covers an application displaying or prefilling a wrong address which the user later approves.
3. **ETH must be absent.** If an ETH record exists, Manager prioritizes it as the main row and this exact IOST-first construction does not apply. The PoC explicitly tests the no-ETH state and `[291, 501]` production order.
4. **This is wrong-recipient loss, not demonstrated attacker profit.** Both `A` and `B` are valid prime-subgroup points, but the PoC does not claim Mallory knows the scalar for `A`. The demonstrated consequence is that funds do not reach the intended recipient `B`; control or recoverability of `A` is not claimed.
5. **The contract asset is out of scope.** No contract vulnerability is alleged. The Forge test is supporting evidence that a narrowly authorized actor can create the app input while remaining unable to modify SOL.

## Recommended remediation

Do not compare arbitrary multi-coin address strings through a universal lowercase projection.

The safest minimal rule is exact string equality by default, with case-insensitive comparison restricted to address formats that are explicitly case-insensitive. A more general implementation can decode both values using their coin-specific codecs and group them only when their canonical decoded bytes are equal and the formats are intended to share a destination. In all cases, a SOL icon must never inherit a copy target whose decoded bytes differ from the SOL record.

Add regression coverage for:

* cross-codec values with equal lowercase strings but different decoded bytes;
* IOST 291 before SOL 501 with no ETH;
* actual icon rendering and clipboard value;
* non-colliding SOL visibility; and
* normal shared-address EVM records.

## Proof of Concept

### Steps

{% stepper %}
{% step %}

## Use a PermissionedResolver profile

Use a PermissionedResolver profile with no ETH record and legitimate SOL-501 address `B`.
{% endstep %}

{% step %}

## Grant Mallory the IOST role only

Grant Mallory only `authorizeAddrRoles(name, 291, mallory, true)`.
{% endstep %}

{% step %}

## Write the IOST address

As Mallory, call `setAddr(node, 291, bytes(A))`.
{% endstep %}

{% step %}

## Confirm the permission boundary

Confirm Mallory cannot call `setAddr(node, 501, ...)`, and that SOL `B` remains unchanged.
{% endstep %}

{% step %}

## Open the profile in Manager

Open the profile in Manager.
{% endstep %}

{% step %}

## Observe the displayed icons and SOL row

Observe that the main row contains both IOST and SOL icons, while no separate SOL row exposes `B`.
{% endstep %}

{% step %}

## Copy the main address

Click the main Copy button.
{% endstep %}

{% step %}

## Confirm the clipboard value

Observe that the clipboard contains IOST `A`, not SOL `B`.
{% endstep %}
{% endstepper %}

### Commands and expected results

Manager component, from `apps/manager`:

```powershell
.\node_modules\.bin\vitest.CMD run --config vitest.config.ts audit-artifacts/profile-address-casefold-role-boundary-20260831/ProfileAddressCaseFoldRoleBoundary.poc.test.tsx
```

Address validation, from the Manager repository root:

```powershell
node apps/manager/audit-artifacts/profile-address-casefold-role-boundary-20260831/verify-addresses.mjs
python apps/manager/audit-artifacts/profile-address-casefold-role-boundary-20260831/verify-addresses.py
```

Permission boundary, from `ens-contracts-v2/contracts`:

```powershell
forge test --match-contract ManagerProfileCaseFoldRoleBoundaryTest -vv
```

Results: Manager 2/2 PASS; JavaScript verifier PASS; Python solders/PyNaCl verifier PASS; Forge 1/1 PASS.

### Important boundaries

* Fine-grained per-coin delegation is a documented protocol feature but is not exposed by the frozen Manager role UI.
* The PoC stops at the clipboard and does not execute a live transfer.
* The construction requires no ETH record.
* `A` is an unintended valid destination; attacker control of its private key is not claimed.
* The Forge file proves state reachability only; no contract bug is claimed.

The full component PoC, independent address verifiers, and Forge permission-boundary PoC are reproduced in the report text and visual attachments. No live funds were sent and no production chain was modified.
