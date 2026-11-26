# \#100060 [SC-Insight] Whitepaper documents a Senior-capacity cancellation escape that cancelPendingDeposit does not implement

**Submitted on Nov 16th 2026 at 07:03:59 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100060
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

# Description

`packages/perps/WHITEPAPER.md:698-700` says that after maturity, deposit
cancellation follows the documented “epoch rejection, terminal-wipe,
impairment, and Senior-capacity escape policy.” The named Senior-capacity
escape does not exist.

`TrancheVault.cancelPendingDeposit` permits an accepted Senior request to be
recovered only when its epoch is rejected, LP epoch settlement is explicitly
held, a terminal wipe is projected, or the Senior tranche is projected to be
impaired. The cancellation condition has no test for current Senior capacity
or whether accepted reservations still fit the active limits.

That is consistent with the implementation's admission-time reservation model.
`HousePool.reserveSeniorDeposit` enforces capacity when the request is accepted,
but later settlement activates the accepted Senior deposit without reapplying
the current admission-cap check. `SECURITY.md:1392-1395` states the intended
rule directly: governance may reduce Senior limits below live exposure, but
accepted Senior reservations are grandfathered against later cap or accounting
changes.

The whitepaper therefore names an unavailable recovery path. An LP, integrator,
or governance team relying on it could expect a cap reduction to unlock an
accepted Senior commitment, while the contract instead keeps the commitment
firm and proceeds to activation.

Correcting this whitepaper sentence aligns it with `SECURITY.md` and the
implemented four-condition recovery model, and prevents LPs, integrators and
governance teams from planning around an unavailable capacity escape.

## References

All source paths are under `packages/perps/` at commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

- `WHITEPAPER.md:698-700` — nonexistent Senior-capacity escape sentence
- `src/TrancheVault.sol:846-870` — complete deposit-cancellation predicate
- `src/HousePool.sol:768-780` — admission-time Senior reservation check
- `src/HousePool.sol:929-939` — accepted Senior deposit settlement
- `SECURITY.md:1392-1395` — prospective cap reduction and reservation
  grandfathering

## Proof of Concept

# Proof of Concept

Conditions under which this Insight is valuable: an accepted Senior deposit is
still pending when governance prospectively lowers a Senior capacity limit, or
accounting changes place accepted reservations outside the active limits. This
is exactly when the whitepaper's named capacity escape would matter to LPs,
frontends, support staff and governance runbooks.

Read-only source demonstration:

1. Read `WHITEPAPER.md:698-700`; the second sentence still names a
   “Senior-capacity escape policy.”
2. Read `TrancheVault.sol:846-870`; the Senior cancellation gate checks epoch
   rejection, settlement hold, projected terminal wipe and projected
   impairment, but not capacity.
3. Read `HousePool.sol:768-780`; capacity is checked when a Senior reservation
   is first accepted.
4. Read `HousePool.sol:929-939`; the accepted Senior deposit phase settles
   without reapplying the admission-cap check.
5. Read `SECURITY.md:1392-1395`; accepted reservations are expressly
   grandfathered against later cap or accounting changes.

Retained local execution confirms the distinction. After accepting a
`10,000.000000`-asset Senior request and finalizing a cap below live exposure,
the Pool reported zero additional capacity and reported that reservations were
outside active limits. Cancellation still reverted
`TrancheVault__SeniorDepositCommitted` before and at maturity. Settlement then
succeeded and made all `10,000.000000` assets claimable, with nothing pending or
refundable.

Together, the source comparison and retained execution establish the
discrepancy: the current whitepaper sentence contradicts both the implemented
lifecycle and its observed behavior.


