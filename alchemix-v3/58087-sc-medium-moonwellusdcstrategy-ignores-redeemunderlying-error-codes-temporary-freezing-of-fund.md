# 58087 sc medium moonwellusdcstrategy ignores redeemunderlying error codes temporary freezing of funds withdrawals revert&#x20;

**Submitted on Oct 30th 2025 at 15:12:21 UTC by @humaira45 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58087
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

In the Alchemix V3 MYT adapters for Moonwell (Optimism), the strategies call Compound-style Moonwell functions that return a uint error code (0 = success), and do not always revert on failure. Specifically, redeemUnderlying(amount) can fail (e.g., “redeem paused”, insufficient cash, per-market caps) by returning a nonzero error code. Both MoonwellUSDCStrategy and MoonwellWETHStrategy ignore the return value and continue execution.

As a result, deallocation/withdrawal proceeds as if redeem succeeded, and the strategy only fails at the end when it requires the underlying balance to be ≥ amount. This deterministically reverts user withdrawals, creating a temporary freeze that persists as long as the Moonwell market remains in a failing state. Our Foundry PoC demonstrates the freeze persisting across +1 hour and +24 hours (via vm.warp), satisfying the “Temporary freezing of funds for at least 24 hour” impact. If the team requires stricter operational evidence for a 24h window on a live market, the report can fall back to “Temporary freezing of funds for at least 1 hour,” which is also demonstrated.

## Vulnerability Details

What the contract does

* Strategy deallocation path:
  * Calls mToken.redeemUnderlying(amount) on Moonwell to receive underlying back to the strategy.
  * Then requires the strategy’s underlying balance to be ≥ amount and approves the vault to pull.
* Caller expectations:
  * Compound/Moonwell-style functions return uint success codes and do not revert on many failures. Callers must check require(err == 0).

Where the bug happens

Primary impacted contract for this report (choose one):

* In-scope contract: MoonwellUSDCStrategy.sol (Smart Contract - MoonwellUSDCStrategy — 14 October 2025)
  * Function: \_deallocate(uint256 amount)
    * mUSDC.redeemUnderlying(amount); // no check on the returned error code
    * require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
    * If redeemUnderlying returns a nonzero error code (no revert), the function still proceeds and ends in a revert, freezing user withdrawals.

Also impacted (same root cause, identical outcome):

* MoonwellWETHStrategy.sol (Smart Contract - MoonwellWETHStrategy — 14 October 2025)
  * Function: \_deallocate(uint256 amount)
    * mWETH.redeemUnderlying(amount); // no check on returned error code
    * require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, ...)
    * Note: MWETH’s redeem flows ETH to the strategy (via MWethDelegate), but the freeze is caused by the unchecked error code regardless of ETH/WETH specifics.

Why this freezes funds

* When redeemUnderlying fails with err != 0 (non-revert), the strategy does not notice, continues, and only at the end checks “do we have amount of underlying?” Since no funds arrived, the require fails and user withdrawals revert.
* The bug is time-independent: the path will keep reverting while redeemUnderlying keeps failing (paused market, low cash, caps, etc.). Our PoC uses vm.warp to show the condition persists over +1 hour and +24 hours when the failure mode does not change.

Root cause (code level)

* Unchecked Compound/Moonwell return codes:
  * MoonwellUSDCStrategy.\_deallocate: mUSDC.redeemUnderlying(amount) is invoked without require(ret == 0).
  * MoonwellWETHStrategy.\_deallocate: mWETH.redeemUnderlying(amount) is invoked without require(ret == 0).
* MYTStrategy enforces that a strategy must have ≥ amount at the end of deallocation. Because the redeem failed silently, the final require trips and reverts, freezing withdrawals.

Impact Details

## Impact Details

High — Temporary freezing of funds for at least 24 hour

* Our PoC demonstrates persistent withdraw reverts after advancing time by +1 hour and +24 hours (vm.warp), while the underlying failure condition remains. If the team requires stricter, real-world evidence for the 24h persistence, this report can fall back to Medium — Temporary freezing of funds for at least 1 hour, which is also explicitly demonstrated.

User impact:

* Users cannot withdraw their funds from the vault while redeemUnderlying keeps returning error codes (no revert). This is a clear availability failure for user funds.

## References

* In-scope strategies:
  * MoonwellUSDCStrategy.sol
    * \_deallocate(uint256) ignores error code of mUSDC.redeemUnderlying(amount)
  * MoonwellWETHStrategy.sol
    * \_deallocate(uint256) ignores error code of mWETH.redeemUnderlying(amount)
* Moonwell behavior (Compound-style):
  * MErc20Delegator.redeemUnderlying(uint): delegates and abi-decodes a uint result, does not force a revert on failures, returning nonzero error codes instead.
  * MToken.redeemUnderlyingInternal / redeemFresh use error-code flow for many failure conditions (paused, insufficient cash, etc.).

## Link to Proof of Concept

<https://gist.github.com/humairar301-droid/f3786fa32965ca5a6721adedbe452df8>

## Proof of Concept

## Proof of Concept

What this PoC proves (end‑to‑end)

* Demonstrates that when redeemUnderlying returns a nonzero error code (no revert), the vault’s withdraw path reverts on both WETH and USDC strategies.
* Shows the freeze persists after advancing time by +1 hour and +24 hours via vm.warp (failure is time-independent and lasts as long as the underlying redeem failures continue).
* Includes sanity tests showing that when redeem succeeds (err = 0 and funds are available), withdraw proceeds without revert.

Full PoC (single file) — Gist Link: <https://gist.github.com/humairar301-droid/f3786fa32965ca5a6721adedbe452df8>

How to run

* All tests in this PoC contract:
  * forge test -vvv --match-contract UncheckedRedeemCodes\_Moonwell\_PoC --evm-version cancun
* Individual tests:
  * forge test -vvv --match-test test\_WETH\_RedeemErrorCode\_FreezesWithdraw --evm-version cancun
  * forge test -vvv --match-test test\_USDC\_RedeemErrorCode\_FreezesWithdraw --evm-version cancun
  * forge test -vvv --match-test test\_WETH\_RedeemError\_Freeze\_Persists\_1h\_and\_24h --evm-version cancun
  * forge test -vvv --match-test test\_USDC\_RedeemError\_Freeze\_Persists\_1h\_and\_24h --evm-version cancun
  * forge test -vvv --match-test test\_WETH\_RedeemOk\_AllowsWithdraw --evm-version cancun
  * forge test -vvv --match-test test\_USDC\_RedeemOk\_AllowsWithdraw --evm-version cancun

Representative results

* WETH and USDC, redeem failure case:
  * Withdraw reverts with: "Strategy balance is less than the amount needed"
  * After vm.warp +1 hour and +24 hours, the same revert persists.
* Sanity (redeem success):
  * When redeemUnderlying returns 0 and pays out, withdraw succeeds and allocation decreases.

Why this is in‑scope and feasible

* In-scope assets: Both MoonwellUSDCStrategy.sol and MoonwellWETHStrategy.sol are explicitly listed in the program scope.
* Impact category: “Temporary freezing of funds for at least 24 hour” (High) is covered. If the project requests different evidence, a fallback to “Temporary freezing of funds for at least 1 hour” (Medium) is also validated by the PoC.
* No exotic assumptions:
  * Compound/Moonwell functions return error codes and do not revert on many types of failures.
  * The adapters must check err == 0; they currently do not.

Suggested remediation

Minimal, ABI-compatible fix: Check and enforce Moonwell return codes in deallocation.

* MoonwellUSDCStrategy:
  * require(mUSDC.redeemUnderlying(amount) == 0, "Moonwell redeemUnderlying failed");
* MoonwellWETHStrategy:
  * require(mWETH.redeemUnderlying(amount) == 0, "Moonwell redeemUnderlying failed");

Optional hardening:

* Consider retry/partial deallocation semantics or graceful handling to avoid hard reverts (e.g., deallocate the actually redeemed amount if nonzero).
* Add circuit-breaker/kill-switch policy for known Moonwell failures to avoid triggering user-facing withdraws while underlying is paused/illiquid (the contract already exposes killSwitch; ensure operational playbooks use it).
* Ensure integration tests cover Compound-style nonreverting error codes, both on mint and redeem paths.
