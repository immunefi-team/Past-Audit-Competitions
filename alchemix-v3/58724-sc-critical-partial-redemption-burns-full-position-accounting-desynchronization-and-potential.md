# 58724 sc critical partial redemption burns full position accounting desynchronization and potential underpayment in transmuter claimredemption&#x20;

**Submitted on Nov 4th 2025 at 09:37:23 UTC by @blackdruiid for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58724
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

When a user redeems a position during periods of liquidity shortage, the Transmuter contract finalizes the position as if it were fully settled, burning the associated NFT and unlocking the entire nominal lock amount, even though only a fraction of the owed collateral can actually be paid out at that time.

This behavior causes an accounting desynchronization between the protocol’s locked liabilities and available collateral and can lead to effective underpayment or loss of claim rights for users. In liquidity, starved scenarios, users receive only a partial payout while their position is marked as fully redeemed, forcing them to recreate a new redemption to recover the remainder once liquidity returns.

## Vulnerability Details

The issue occurs in the Transmuter.claimRedemption() function, which unconditionally finalizes a redemption position as fully settled, regardless of whether the protocol has enough liquidity to complete the payout.

Specifically, the function:

* Burns the NFT representing the redemption position.
* Decreases totalLocked by the full nominal redemption amount (position.amount).
* Marks the position as fully claimed.

When liquidity is limited, the redemption payout is only partial, yet these accounting operations are performed in full, effectively unlocking more than what has actually been paid out.

Root Cause: During liquidity starvation (when MYT collateral is insufficient to satisfy redemptions), claimRedemption() attempts to pay the redeemer what’s available. This behavior originates from AlchemistV3.sol::redeem(uint256 amount), where the logic caps the redeemable amount as follows:

```
if (amount > liveEarmarked) amount = liveEarmarked
```

As a result, the Transmuter only receives and distributes the available portion of collateral, but still executes complete settlement logic for the entire position.

However, even in this case, claimRedemption() still:

* Burns the NFT representing the redemption position.
* Unlocks the full totalLocked amount.
* Marks the position as fully claimed.

Thus, even though only part of the requested amount is actually paid out, the contract treats the redemption as complete.

## Impact Details

1. Accounting Desynchronization: totalLocked and the protocol’s internal debt accounting become inconsistent with the actual asset and liability state, potentially misrepresenting solvency.
2. Loss of Claim Rights: Since the NFT is burned, users permanently lose the right to redeem the unpaid portion once liquidity becomes available again.
3. Potential Underpayment: Users may effectively receive less than their entitled value when claiming during liquidity stress, as the remaining amount is never recoverable.
4. Operational and Monitoring Risk: On-chain metrics (e.g., totalLocked, total debt, synthetic supply) will indicate that the redemption was fully settled, complicating audits, solvency tracking, and automated recovery mechanisms.

## References

* Affected function: Transmuter.sol::claimRedemption(uint256 tokenId)
* Related logic: AlchemistV3.sol::redeem(uint256 amount)

## Proof of Concept

## Proof of Concept

I put the test in src/test/AlchemistV3.t.sol because, with the existing setup, it’s the most complete and convenient place to exercise this function.

You can copy-paste the function into AlchemistV3.t.sol and then run:

```
forge test --fork-url mainnet --match-path src/test/AlchemistV3.t.sol -vvvv --evm-version cancun --mt test_PartialPayoutStillBurnsFullPosition
```

Here the test function:

* Expected: claiming a partial redemption reduces totalLocked only by the amount paid and burns only the claimed portion.
* Actual: a partial payout burns the entire position and unlocks the full totalLocked (over-unlock).

```
function test_PartialPayoutStillBurnsFullPosition() public {
    address buyer = address(0xBEEF);
    address user  = address(0xBAAF);
    Transmuter _transmuter = transmuterLogic;

    // --- Setup user collateral + mint synth ---
    deal(address(vault), user, 2_000e18);
    vm.startPrank(user);
    IERC20(address(vault)).approve(address(alchemist), type(uint256).max);
    alchemist.deposit(1_000e18, user, 0); // positionId = 1
    uint256 tokenId = 1;
    alchemist.mint(tokenId, 200e18, user);
    uint256 synthSupplyBefore = alchemist.totalSyntheticsIssued();
    alToken.approve(address(_transmuter), type(uint256).max);
    vm.stopPrank();

    // --- Buyer creates redemption for 100e18 ---
    deal(address(alToken), buyer, 100e18);
    vm.startPrank(buyer);
    uint256 buyerMytBefore = IERC20(address(vault)).balanceOf(buyer);
    alToken.approve(address(_transmuter), type(uint256).max);
    _transmuter.createRedemption(100e18);
    vm.stopPrank();

    // --- Starve transmuter so claim is underfunded ---
    vm.roll(block.number + 1_000_000);
    deal(address(vault), address(_transmuter), 0);
    deal(address(vault), address(alchemist), 40e18);

    // --- Snapshot before claim ---
    uint256 lockedBefore = _transmuter.totalLocked();
    uint256 buyerMytBeforeClaim = IERC20(address(vault)).balanceOf(buyer);
    uint256 synthSupplyBeforeClaim = alchemist.totalSyntheticsIssued();

    // --- Claim redemption (partial payout happens) ---
    vm.startPrank(buyer);
    _transmuter.claimRedemption(tokenId);
    vm.stopPrank();

    // --- Snapshot after claim ---
    uint256 lockedAfter     = _transmuter.totalLocked();
    uint256 buyerMytAfter   = IERC20(address(vault)).balanceOf(buyer);
    uint256 synthSupplyAfter= alchemist.totalSyntheticsIssued();

    // Second claim must fail (NFT burned)
    vm.startPrank(buyer);
    vm.expectRevert();
    _transmuter.claimRedemption(tokenId);
    vm.stopPrank();

    // --- Assertions ---
    assertGt(buyerMytAfter, buyerMytBeforeClaim, "no MYT payout");
    assertLt(synthSupplyAfter, synthSupplyBeforeClaim, "synth supply not reduced");
    assertLt(lockedAfter, lockedBefore, "totalLocked not updated");

    // Core bug: unlocked accounting (full position) > actually paid out (partial)
    uint256 paidOut            = buyerMytAfter - buyerMytBefore;
    uint256 unlockedAccounting = lockedBefore - lockedAfter;
    assertGt(unlockedAccounting, paidOut, "no over-unlock");

    // Logs
    console.log("synthSupplyBefore =", synthSupplyBefore);
    console.log("synthSupplyAfter  =", synthSupplyAfter);
    console.log("buyerMytBefore    =", buyerMytBefore);
    console.log("buyerMytAfter     =", buyerMytAfter);
    console.log("lockedBefore      =", lockedBefore);
    console.log("lockedAfter       =", lockedAfter);
    console.log("paidOut           =", paidOut);
    console.log("unlockedAccounting=", unlockedAccounting);
}
```

**Quick numbers recap**:

* Deposit: 1,000e18 MYT → position #1
* Mint debt: 200e18 alToken
* Buyer redemption: 100e18 alToken
* Claim transfers:

19.006849315068493151e18 MYT → buyer

0.019025875190258751e18 MYT → test contract

80.812176560121765602e18 alToken → refunded to buyer

19.025875190258751902e18 alToken → burned

Locks: 100e18 → 0

Supply: 200e18 → 180.974124809741248098e18

* Over-unlock: unlocked 100e18 vs paid \~19.0068e18
