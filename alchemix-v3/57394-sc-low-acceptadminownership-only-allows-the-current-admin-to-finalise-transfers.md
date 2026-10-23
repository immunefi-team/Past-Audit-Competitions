# 57394 sc low acceptadminownership only allows the current admin to finalise transfers

## #57394 \[SC-Low] \`acceptAdminOwnership()\` only allows the current admin to finalise transfers

**Submitted on Oct 25th 2025 at 19:48:50 UTC by @pxng0lin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57394
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**

### Description

## Bug Description

`AlchemistCurator.acceptAdminOwnership()` @src/AlchemistCurator.sol#31-34 is guarded by `onlyAdmin`. After `transferAdminOwnerShip()` sets `pendingAdmin`, the same `admin` must also call `acceptAdminOwnership()` because any other address fails the modifier check. A nominated `pendingAdmin` cannot execute the function, so the contract relies on the outgoing admin to complete both steps of the handover. If that account becomes unavailable, the transition cannot finish.

## Impact

* **Admin handover dependency:** The nominated address cannot finalise control; only the existing admin can call the second step.
* **Operational risk:** If the current admin loses access after scheduling a transfer, the contract remains under the old admin indefinitely.

## Risk Breakdown

* **Privilege required:** Current `admin`.
* **Likelihood:** Medium

## Recommendation

Allow the nominated party to accept by checking `msg.sender == pendingAdmin` (e.g., an `onlyPendingAdmin` modifier) before updating `admin`.

## References

* Handover initiation: `transferAdminOwnerShip()` @src/AlchemistCurator.sol#26-29.
* Acceptance function: `acceptAdminOwnership()` @src/AlchemistCurator.sol#31-34.

### Proof of Concept

### Proof of Concept

1. **Existing ownership:** The contract is deployed with `admin = addrA`.
2. **Planned handover:** `addrA` initiates the transition by calling `transferAdminOwnerShip(addrB)`. Storage now records `pendingAdmin = addrB`.
3. **New admin tries to accept:** Acting as `addrB`, we call `acceptAdminOwnership()`. The call reverts with `Unauthorized("PD")` because `onlyAdmin` accepts only the current `admin`.
4. **Original admin finalises instead:** `addrA` repeats the call and succeeds. The function assigns `admin = addrB` and clears `pendingAdmin`, even though `addrB` never interacted.
5. **Failure mode:** If `addrA` disappears between steps 2 and 4—lost keys, compromised wallet—the takeover cannot complete. Governance stays bound to `addrA`.
