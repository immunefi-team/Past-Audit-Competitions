# 56936 sc high missing mytsharesdeposited decrements on repay liquidation tvl drift false over collateralization and deposit cap dos

**Submitted on Oct 21st 2025 at 22:44:08 UTC by @s\_a\_l\_e\_m for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56936
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

The AlchemistV3 contract fails to decrement `_mytSharesDeposited` when MYT leaves during force‑repay and liquidation. This creates a persistent, inflated internal balance that (1) misreports TVL and overstates collateralization, (2) under‑applies bad‑debt haircuts in the Transmuter’s denominator (leaking value over time), and (3) blocks new deposits by exhausting `depositCap` despite real capacity (deposit‑side DoS). Governance workarounds like repeatedly raising `depositCap` decouple limits from reality and compound risk.

## Vulnerability Details

At a high level, the contract maintains an internal counter `_mytSharesDeposited` as the book of record for how many MYT shares the Alchemist currently holds. This counter is used in two critical places: (a) to gate new deposits against `depositCap` and (b) to derive “total underlying value” (TVL) that informs collateralization and downstream ratios. When MYT comes in (deposits) the counter increases; when MYT leaves (withdraws/redeems) it should decrease. The implementation follows this rule in some paths (withdraw/redeem and the fee component of repay), but omits it in two others where MYT definitively leaves the Alchemist: force‑repay and liquidation.

In a force‑repay, the Alchemist repays earmarked user debt by sending its own MYT to the Transmuter and pays a protocol fee in MYT to the fee receiver. Both transfers reduce the Alchemist’s actual MYT balance, but the counter is not decremented. In a liquidation, the Alchemist similarly transfers MYT to the Transmuter (net of any fee) and may pay a fee in MYT to the liquidator; again, the counter is not decremented. The result is a persistent “phantom balance” in `_mytSharesDeposited` that is higher than the real on‑chain MYT the contract holds.

Why this matters immediately:

* The deposit gate compares “current counter + incoming deposit” to `depositCap`. Because the counter is inflated after these outflows, the system believes the cap is already consumed and reverts deposits even when there is real capacity. This is a deposit‑side denial‑of‑service that does not self‑heal—an admin must raise `depositCap` to re‑open deposits, and the accounting remains wrong afterward. or a user has to withdraw/reedem as will also create room.

Why this matters systemically (beyond simple DoS):

* TVL is derived from `_mytSharesDeposited`. With an inflated counter, TVL (and any collateralization metrics that use it) are overstated. That produces a false over‑collateralization signal: positions and the system look healthier than they are. In practice this pushes liquidations later/smaller than required.
* The Transmuter also consumes TVL in its denominator when computing bad‑debt haircuts. An overstated denominator reduces haircuts, so claimants can receive slightly more than warranted by true system health. This does not let anyone withdraw more than available balances, but it leaks value faster by under‑charging haircuts and depleting reserves sooner.
* The same accounting rule (“decrement on outflow”) is implemented in other flows (redeem, repay fee). Force‑repay and liquidation are logically identical with respect to the Alchemist’s MYT balance and should update the counter as well.
* There is no compensating mechanism to re‑align `_mytSharesDeposited` with actual balances after these paths. Without a corrective update or a management function to reset the counter, the drift persists indefinitely and compounds across events.

## Impact Details

* Deposit-side denial of service (DoS):
  * After liquidation/force-repay, `_mytSharesDeposited` stays inflated while real MYT decreased.
  * `deposit()` reverts once `_mytSharesDeposited + amount > depositCap` despite real capacity existing.
  * Only admin cap increases or significant outflows can restore liveness; the root accounting remains wrong.
* Misreported TVL/health metrics:
  * Any metric or decision derived from `_mytSharesDeposited` overstates assets.
  * This can skew health assessments, dashboards, and governance automation.
* False over‑collateralization and value leakage over time:
  * Overstated TVL inflates collateralization ratios, delaying or under‑sizing liquidations.
  * The Transmuter’s denominator becomes too large, so bad‑debt haircuts are under‑applied. Claimants can receive slightly more per unit of synthetic than true health warrants, draining reserves faster (bounded by available balances, but still harmful).
* Governance toil and risk drift:
  * Admins may repeatedly increase `depositCap` to unblock deposits, decoupling caps from true capacity.
  * Over time this degrades the effectiveness of deposit limits and masks the real headroom.
* Long-term protocol risk (indirect):
  * Persistently overstated TVL can delay or under-size defensive actions if any off-chain/on-chain processes rely on these values.
  * While not a direct insolvency vector, it increases operational risk and can hinder timely recapitalization.

It does not directly steal funds, but it creates a reproducible deposit DoS and persistent accounting drift with governance ramifications.

## References

* Deposit cap gating uses `_mytSharesDeposited`: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L369>
* Force-repay (no decrement): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L771-L780>
* Liquidation (no decrement): <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L824-L841>

## Proof of Concept

## Proof of Concept

* This proves that `_mytSharesDeposited` is not updated for liquidations, which can lead to deposit DOS and other issues as explained in the report.

```javascript
 function testDepositBlockedAfterLiquidationDueToStaleMytShares() external {
        // 1) Seed Alchemist with shares
        uint256 seedAmount = 150e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), seedAmount + 100e18);
        alchemist.deposit(seedAmount, address(0xbeef), 0);
        // deposit more to create a borrowable position
        SafeERC20.safeApprove(address(vault), address(alchemist), seedAmount);
        alchemist.deposit(seedAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        // Set depositCap AFTER deposits to current shares
        uint256 cap = IERC20(address(vault)).balanceOf(address(alchemist));
        vm.prank(alOwner);
        alchemist.setDepositCap(cap);

        // Start a redemption to earmark debt (enables force-repay component of liquidation)
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();

        // Make account undercollateralized
        uint256 s0 = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(s0);
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(s0 + (s0 * 1000 / 10_000)); // +10%

        // Snapshot shares before
        uint256 sharesBefore = IERC20(address(vault)).balanceOf(address(alchemist));

        // Liquidate
        vm.prank(externalUser);
        alchemist.liquidate(tokenId);

        // Assert MYT actually left
        uint256 sharesAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        assertTrue(sharesAfter < sharesBefore);

        // Attempt a new deposit; cap should reject because _mytSharesDeposited was not decremented
        // Give depositor vault shares to deposit
        uint256 small = 1e18;
        _magicDepositToVault(address(vault), anotherExternalUser, small);
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), small);
        vm.expectRevert(IllegalState.selector);
        alchemist.deposit(small, anotherExternalUser, 0);
        vm.stopPrank();

        // Check that by actual shares, the deposit would fit under the cap (i.e., it is an artificial block)
        assertTrue(sharesAfter + small <= cap);
    }
```
