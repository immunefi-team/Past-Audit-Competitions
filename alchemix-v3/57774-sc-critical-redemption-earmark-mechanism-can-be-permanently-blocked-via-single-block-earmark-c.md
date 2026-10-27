# 57774 sc critical redemption earmark mechanism can be permanently blocked via single block earmark calls

## #57774 \[SC-Critical] Redemption Earmark Mechanism Can Be Permanently Blocked via Single-Block Earmark Calls

**Submitted on Oct 28th 2025 at 20:13:50 UTC by @pindarev for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57774
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Permanent freezing of unclaimed yield

### Description

### Brief/Intro

The protocol’s debt redemption process relies on block-based accrual via `queryGraph(startBlock, endBlock)`. An attacker can repeatedly trigger `_earmark()` every block to ensure `endBlock == startBlock`, causing `queryGraph()` to always return zero and preventing any debt from being earmarked.

Alchemix v3 utilizes a block-range-dependent staking graph to determine how much debt should be redeemed (“earmarked”) over time. However, the contract updates `lastEarmarkBlock` every time `_earmark()` is called. An attacker can force `_earmark()` to run every block, causing the graph query to always evaluate over an empty range and return zero. This effectively starves the redemption pipeline indefinitely.

### Vulnerability Details

The vulnerable call occurs here (simplified):

```solidity
    function _earmark() internal {
.
.
.
        uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);
.
.
.
        lastEarmarkBlock = block.number;
    }
```

And in the `Transmuter`:

```solidity
    function queryGraph(uint256 startBlock, uint256 endBlock) external view returns (uint256) {
        if (endBlock <= startBlock) return 0;
.
.
.
    }

```

If `_earmark()` is invoked every block, then:

* startBlock = N + 1
* endBlock = N + 1

This condition triggers, so:

queryGraph returns 0 → No earmarking occurs → No redemption progresses

Any address can call `poke(tokenId)` to trigger this behavior. No value or permission is needed, making this trivially griefable.

## Root cause

The protocol attempts to derive continuous redemption progress based on block deltas, but updates to `lastEarmarkBlock` are attacker-influenceable. By controlling when `_earmark()` is called, an attacker controls the block range width, which can be forced to zero every time.

### Impact Details

This vulnerability prevents the protocol from accruing earmarked debt for the Transmuter. As a consequence:

* The redemption pipeline will not progress — no debt will be earmarked for redemption while the attacker maintains the attack.
* Users that create redemptions will not receive yield in future, because the redemption queue will stop advancing.
* The protocol’s redemption logic is effectively broken and the protocol becomes unstable with respect to redemptions (high impact on user experience and protocol guarantees).

### References

[AlchemistV3.sol::\_earmark()](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol?utm_source=immunefi#L1106) (logic that calls queryGraph)

[Transmuter.sol::queryGraph()](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/Transmuter.sol?utm_source=immunefi#L270) (returns zero when endBlock <= startBlock)

### Mitigation

Consider one the options:

* Switch from block-range based accrual to an index-based accumulator so that earmark progression cannot be forced to zero by timing calls.
* Maintain a stored graph cursor (or accumulator checkpoint) instead of relying on `block.number` to ensure each `_earmark()` call always advances state.
* Add a minimum block spacing requirement before updating `lastEarmarkBlock` to prevent zero-range `queryGraph` calls.

### Proof of Concept

### Proof of Concept

Add the following test in `src/test/AlchemistV3.t.sol` file and run it using this command `forge test --mt test_EarmarkGriefingAttack -vv`

PoC:

```solidity
    function test_EarmarkGriefingAttack() external {
        uint256 amount = 100e18;

        // 1) User opens CDP and borrows
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, amount / 2, address(0xbeef)); // create debt
        vm.stopPrank();

        // 2) Second user initiates redemption which should cause earmarking
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();

        // Ensure initial earmark is zero (no prior activity)
        (, , uint256 initialEarmarked) = alchemist.getCDP(tokenId);
        assertEq(initialEarmarked, 0);

        // 3) Attacker calls poke(tokenId) every block for many blocks, starving the graph query
        address attacker = address(0xBADBEEF);
        for (uint256 i = 0; i < 200; i++) {
            vm.roll(block.number + 1);
            vm.prank(attacker);
            alchemist.poke(tokenId); // _earmark() is executed here
        }

        // 4) Check earmark has not increased
        (, , uint256 earmarkedAfterAttack) = alchemist.getCDP(tokenId);
        assertEq(earmarkedAfterAttack, 0, "Earmark should remain zero due to attacker forcing zero-length queries");
    }
```
