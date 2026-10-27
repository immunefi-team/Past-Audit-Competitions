# 89114 sc medium permanent denial of service of firelightvault deposit via multiplicative share inflation once totalassets reaches zero through a legitimate payout&#x20;

**Submitted on Aug 17th 2026 at 09:23:39 UTC by @Adanix for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89114
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

After a full, legitimate incident `payout()` drains the vault, `totalAssets()` becomes 0 while `totalSupply()` stays > 0 (payout burns no shares). Because the ERC4626 virtual-offset conversion degenerates to pure multiplication when `totalAssets()==0`, each subsequent permissionless `deposit()` multiplies `totalSupply` exponentially until `mulDiv` overflows and every future `deposit()` reverts permanently with `Panic(0x11)`. In production this permanently disables the vault's ability to accept new deposits, protocol-wide, with no admin recovery path — reached through normal insurance-vault operation, not an attack.

## Vulnerability Details

`FirelightVault` uses the standard OpenZeppelin ERC4626 virtual-offset conversion:

```solidity
// _convertToSharesTotals
return assets.mulDiv(totSupply + 10 ** _decimalsOffset(), totAssets + 1, rounding);
```

This correctly defends the classic first-depositor / donation inflation attack (the only inflation issue listed as a Known Issue for this competition). It does NOT defend the case where `totalAssets()` is driven to 0 while `totalSupply()` stays > 0.

That state is reachable through normal operation:

{% stepper %}
{% step %}

## A regular incident `payout()` drains the vault

A regular incident `payout()` (gated by `PAYOUT_ROLE`) pays out the vault. `payout()` burns no shares. On a full payout, raw balance and `totalAssets()` go to 0 while `totalSupply()` is unchanged.

Note `totalAssets() = super.totalAssets() - pendingWithdrawAssets`; `pendingWithdrawAssets` only makes reaching 0 easier, and there is no reserve floor keeping `totalAssets()` above 0.
{% endstep %}

{% step %}

## The incident guard expires independently of asset recovery

The only guard against depositing into this state is `_hasActiveIncident()`, which checks only the current and previous period — a 2-period time cooldown, NOT tied to actual recovery of `totalAssets()`.

After two period advances (pure passage of time, no action by anyone) the guard clears while `totalAssets()` is still 0.
{% endstep %}

{% step %}

## A permissionless deposit inflates supply multiplicatively

`deposit()` then only checks `whenNotPaused` / `notBlocklisted` — no role.

With `totalAssets()==0` the formula degenerates to `shares = assets * (totalSupply + offset) / 1`, i.e. each deposit multiplies `totalSupply` by roughly `(assets + 1)`.
{% endstep %}
{% endstepper %}

Repeated regular "incident payout -> later deposit" cycles grow `totalSupply` exponentially until `mulDiv` overflows and every further `deposit()` reverts permanently with `Panic(0x11)`. No in-scope admin function resets `totalSupply` or repairs this state.

## Impact Details

Permanent, protocol-wide denial-of-service of `deposit()`: once triggered, the vault can never accept new capital again. It is reached by normal post-incident operation of an insurance vault (whose entire purpose is to pay out claims) and the damaging `deposit()` requires no privileges.

This matches the in-scope impact "Smart contract unable to operate due to lack of token funds" — the vault becomes unable to operate precisely because its token funds were drained to zero.

Existing positions / `withdraw()` / `redeem()` are likely not directly affected (their conversion runs the other direction and does not overflow); this was not separately verified and is disclosed as such.

## Not a duplicate of the listed Known Issue

The listed Known Issue is the classic first-depositor / donation inflation attack (front-running a fresh vault). This is a different mechanism: no front-running, no attacker donation, no manipulated first deposit — a compounding overflow-DoS triggered by legitimate `payout()` usage, where the built-in defense against the known attack is itself the cause.

It is also absent from the prior OpenZeppelin audit (July 2025), which covered a structurally different mint/burn architecture.

## References

* contracts/core/FirelightVault.sol (`_convertToSharesTotals`, `totalAssets`, `_hasActiveIncident`, `deposit`, `payout`)
* OpenZeppelin ERC4626 inflation-attack / virtual-offset documentation

## Link to Proof of Concept

<https://gist.github.com/milo-technologies-ai/428880c29200005ad921bd436e5dab67>

## Proof of Concept

### Proof of Concept (measured against a real deployed proxy, deterministic)

Setup: deploy `FirelightVault` via the project's standard `upgrades.deployProxy` path (same as production). Vault decimals 0 are used for clarity; larger decimals make the overflow occur in FEWER cycles, so this is the conservative case.

{% stepper %}
{% step %}

## Realistic starting state

Three independent users deposit 200,000 / 200,000 / 100,000.

* `totalSupply() = 500,000`
* `totalAssets() = 500,000`
  {% endstep %}

{% step %}

## Legitimate full payout

The incident team marks the period active (`INCIDENT_ROLE`, as intended), then `PAYOUT_ROLE` calls `payout(receiver, totalAssets, capturePeriodStart)` for the full amount.

Measured on-chain after the call:

* `totalAssets() = 0`
* `totalSupply() = 500,000` (UNCHANGED — payout burns no shares)
* Raw balance = 0

The trigger state (`totalAssets()==0` while `totalSupply()>0`) is now real.
{% endstep %}

{% step %}

## The guard is only a time cooldown

An immediate `deposit()` correctly reverts with `CurrentPeriodHasActiveIncident`. But `_hasActiveIncident()` checks only period and period-1.

After two period advances (pure passage of time, no action by anyone), the guard clears while `totalAssets()` is STILL 0 (measured).
{% endstep %}

{% step %}

## Permissionless trigger

Any non-blocklisted account (no special role) deposits 1000.

* It receives 500,001,000 shares.
* `totalSupply` jumps 500,000 -> 500,501,000 (\~1001x) for a single deposit.

`deposit()` checks only `whenNotPaused` / `notBlocklisted`.
{% endstep %}

{% step %}

## Permanent DoS

Repeating the normal "incident payout empties the vault -> a later deposit" cycle from the realistic 500,000 start, `totalSupply` grows \~exponentially and after 23 cycles the next `deposit()` reverts PERMANENTLY with `Panic(0x11)`.

* `totalSupply ~= 5.1e74`, near the uint256 max.
* Measured, not extrapolated.
* No in-scope admin function resets `totalSupply` or repairs the state.
  {% endstep %}
  {% endstepper %}

Every value above is read directly from the deployed contract after each external call. Full runnable test: see linked secret Gist (place in the competition repo's `test/` folder and run `npx hardhat test`).
