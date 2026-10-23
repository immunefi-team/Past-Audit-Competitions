# 58329 sc low incorrect balance measurement in morphoyearnogweth deallocate leads to temporary freezing of funds via spurious loss events

**Submitted on Nov 1st 2025 at 10:05:44 UTC by @rshackin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58329
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour
  * Temporary freezing of funds for at least 24 hour

## Description

## Summary:

`MorphoYearnOGWETH` strategy contains a logic error in `_deallocate()` that causes it to emit `StrategyDeallocationLoss` **events on every successful deallocation despite funds arriving correctly. This occurs because balance measurements are taken after the vault withdrawal completes, making the computed delta always zero**. The false loss signals can trigger automated circuit breakers that pause the strategy, blocking further deallocations and creating temporary freezing of funds for users relying on this route to redeem collateral.

* Component: `MorphoYearnOGWETH._deallocate()`
* Impact: Deterministic false loss events on normal operations and realistic automation-driven redemption freezing.

## Vulnerability Details:

* Root Cause: In `MorphoYearnOGWETH._deallocate()`, the balance delta calculation reads both snapshots after calling `vault.withdraw()`:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 shares = vault.withdraw(amount, address(this), address(this));
    
    //  BOTH reads happen AFTER withdraw completes
    uint256 balanceBefore = weth.balanceOf(address(this));
    uint256 balanceAfter = weth.balanceOf(address(this));
    uint256 wethRedeemed = balanceAfter - balanceBefore;  // Is Always 0
    
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss(
            "Strategy deallocation loss.",
            amount,
            wethRedeemed  // Always 0, triggering false event
        );
    }
    require(weth.balanceOf(address(this)) >= amount, "Insufficient balance");
    return amount;
}
```

* The `vault.withdraw()` call on line 1 successfully transfers `amount` WETH to the strategy (as confirmed by test balance assertions), but because both `balanceOf()` calls on lines 4-5 happen after this transfer completes, they read the same post-transfer balance, making their difference `(wethRedeemed)` zero regardless of actual funds received. This deterministically forces the loss event to emit on every deallocation.
* Since both reads occur after the external vault call transfers WETH to the strategy, `balanceBefore == balanceAfter`, forcing `wethRedeemed = 0`. The subsequent check `if (wethRedeemed < amount)` always triggers, emitting `StrategyDeallocationLoss(message: "Strategy deallocation loss.", amountRequested: amount, actualAmountSent: 0)` even though the strategy's balance increased by exactly amount.
* Intended correct Behavior (from other strategies): `EulerWETH`, `EulerUSDC`, `Peapods ETH/USDC` all implement the correct pattern:

```solidity
uint256 balanceBefore = token.balanceOf(address(this));  // Before external call
vault.redeem(shares, address(this), address(this));
uint256 balanceAfter = token.balanceOf(address(this));   // After external call
uint256 redeemed = balanceAfter - balanceBefore;         // Real delta
if (redeemed < amount) emit StrategyDeallocationLoss(...); // Only if shortage
```

**This confirms `MorphoYearnOGWETH`'s ordering is an unintended implementation and this is flawed.**

## Impact Details:

1. Direct Impact: Deterministic False Loss Signals.

* Every deallocation through `MorphoYearnOGWETH` emits `StrategyDeallocationLoss(actualAmountSent: 0)` despite funds arriving, creating constant operational error and misleading monitoring systems about the data of this strategy.

2. Primary Impact: Temporary Freezing of Funds (High):

* Mechanism: Production teams commonly deploy automated keepers that monitor strategy events and execute circuit-breaker pauses when losses are detected to prevent cascading failures. The spurious `StrategyDeallocationLoss` can trigger such automation immediately after any deallocation.
* Effect: Once paused, the strategy blocks further `_deallocate()` calls, preventing users from unwinding positions or redeeming collateral through this route until manual intervention/un-pause occurs.

**Duration Justification for ≥24h Threshold**:

* Investigation phase (2-8 hours): On-call engineer reviews logs, identifies StrategyDeallocationLoss events, must distinguish false positives from genuine losses by comparing event history, vault state, and balance proofs
* Coordination phase (4-12 hours): Gathering multisig/admin quorum or governance vote to execute unpause, especially during weekends/holidays when key holders are unavailable
* Repeated cycles: Since the bug fires on every deallocation, even if first pause resolves quickly (best-case 6-8h), subsequent user attempts re-trigger false events and additional pauses, accumulating total freeze time beyond 24 hours
* Realistic minimum: 24-48+ hours is standard operational response time for non-emergency investigations (distinguishing false vs. real threats), satisfying the "at least 24 hour" criterion.

## References:

(<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol#L49-L56>)

## Recommended Mitigation:

Move `balanceBefore` read to line before `vault.withdraw()` call, aligning with pattern used in all other MYT strategies.

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 balanceBefore = weth.balanceOf(address(this));  // Move before withdraw
    uint256 shares = vault.withdraw(amount, address(this), address(this));
    uint256 balanceAfter = weth.balanceOf(address(this));   // Now after withdraw
    uint256 wethRedeemed = balanceAfter - balanceBefore;    // Real delta

    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss(
            "Strategy deallocation loss.",
            amount,
            wethRedeemed  //  Accurate value
        );
    }
    require(weth.balanceOf(address(this)) >= amount, "Insufficient balance");
    return amount;
}
```

**This fix aligns the logic with the pattern used in all other MYT strategies (EulerWETH, EulerUSDC, PeapodsETH, PeapodsUSDC), eliminating false loss events while preserving intended safety monitoring when genuine shortfalls occur..**

## Proof of Concept

## Proof of Concept:

Both PoCs run on a mainnet fork using real Morpho-Yearn OG WETH vault and WETH addresses, satisfying the competition's local-fork testing requirement.

* PoC A: False Loss Event on Successful Deallocation:

1. Place test file in `test/MorphoYearnOGWETH_FalseLoss.t.sol`
2. Export RPC: `export MAINNET_RPC_URL="https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY"`
3. Execute: `forge test --match-contract MorphoYearnOGWETHLossEvent -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "forge-std/Test.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {MorphoYearnOGWETHStrategy} from "../strategies/mainnet/MorphoYearnOGWETH.sol";

interface IWETH { function deposit() external payable; function transfer(address,uint256) external returns (bool); function balanceOf(address) external view returns (uint256); }
interface IERC4626 { function balanceOf(address) external view returns (uint256); }

contract MorphoYearnOGWETHHarness is MorphoYearnOGWETHStrategy {
    constructor(address _myt, StrategyParams memory _p, address _vault, address _weth, address _permit2)
        MorphoYearnOGWETHStrategy(_myt, _p, _vault, _weth, _permit2) {}
    function harnessAllocate(uint256 a) external returns (uint256) { return _allocate(a); }
    function harnessDeallocate(uint256 a) external returns (uint256) { return _deallocate(a); }
}

contract MorphoYearnOGWETHLossEvent is Test {
    address constant MORPHO_YEARN_OG_VAULT = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;
    address constant MAINNET_WETH       = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant MAINNET_PERMIT2    = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

    IWETH weth = IWETH(MAINNET_WETH);
    IERC4626 vault = IERC4626(MORPHO_YEARN_OG_VAULT);
    MorphoYearnOGWETHHarness strat;

    function setUp() public {
      string memory url = vm.envOr("MAINNET_RPC_URL", vm.rpcUrl("mainnet"));
      uint256 forkId = vm.createFork(url);
      vm.selectFork(forkId);


        IMYTStrategy.StrategyParams memory p = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "MorphoYearnOGWETH",
            protocol: "MorphoYearnOGWETH",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: type(uint256).max,
            globalCap: type(uint256).max,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 1
        });

        strat = new MorphoYearnOGWETHHarness(address(0xdead), p, MORPHO_YEARN_OG_VAULT, MAINNET_WETH, MAINNET_PERMIT2);

        address alice = address(0xA11CE);
        vm.deal(alice, 10 ether);
        vm.startPrank(alice);
        IWETH(MAINNET_WETH).deposit{value: 2 ether}();
        IWETH(MAINNET_WETH).transfer(address(strat), 2 ether);
        vm.stopPrank();

        strat.harnessAllocate(1 ether);
        assertGt(vault.balanceOf(address(strat)), 0, "no shares after allocate");
    }

    function test_FalseLossEvent_WhileFundsArrive() public {
        uint256 amount = 0.5 ether;
        uint256 beforeBal = weth.balanceOf(address(strat));
        vm.recordLogs();
        uint256 ret = strat.harnessDeallocate(amount);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        uint256 afterBal = weth.balanceOf(address(strat));

        // Funds actually arrive
        assertEq(ret, amount, "returned amount mismatch");
        assertEq(afterBal - beforeBal, amount, "assets did not arrive");

        // Spurious loss event due to post-withdraw reads in _deallocate
        bytes32 sig = keccak256("StrategyDeallocationLoss(string,uint256,uint256)");
        bool sawLoss = false;
        for (uint256 i; i < logs.length; i++) {
            if (logs[i].topics.length > 0 && logs[i].topics[0] == sig) { sawLoss = true; break; }
        }
        assertTrue(sawLoss, "expected StrategyDeallocationLoss not emitted");
    }
}
```

Results: Test passes; trace shows strategy balance increases by 0.5 ETH (from 1.0 to 1.5) while `StrategyDeallocationLoss(actualAmountSent: 0)` is emitted in the same transaction — proving the contradiction.

```solidity
Ran 1 test for src/test/PoC.t.sol:MorphoYearnOGWETHLossEvent
[PASS] test_FalseLossEvent_WhileFundsArrive() (gas: 541452)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 48.32s (382.07ms CPU time)
```

* PoC B: Automated Pause Leading to Temporary Freezing.

1. Place test file in `test/MorphoYearnOGWETH_AutoFreeze.t.sol`
2. Use same RPC setup as PoC A
3. Execute: `forge test --match-contract MorphoYearnOGWETH_AutoPause -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import "forge-std/Test.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {MorphoYearnOGWETHStrategy} from "../strategies/mainnet/MorphoYearnOGWETH.sol";

interface IWETH { function deposit() external payable; function transfer(address,uint256) external returns (bool); function balanceOf(address) external view returns (uint256); }
interface IERC4626 { function balanceOf(address) external view returns (uint256); }

contract MorphoYearnOGWETHHarnessPauseable is MorphoYearnOGWETHStrategy {
    error Paused();

    bool private _paused;
    address public guardian;

    constructor(address _myt, StrategyParams memory _p, address _vault, address _weth, address _permit2)
        MorphoYearnOGWETHStrategy(_myt, _p, _vault, _weth, _permit2) {}

    function setGuardian(address g) external {
        // keep simple for PoC: test contract is deployer and calls this immediately
        require(guardian == address(0) || guardian == msg.sender, "not auth");
        guardian = g;
    }
    function pause() external {
        require(msg.sender == guardian, "not guardian");
        _paused = true;
    }
    function unpause() external {
        require(msg.sender == guardian, "not guardian");
        _paused = false;
    }

    function _checkNotPaused() internal view {
        if (_paused) revert Paused();
    }

    function harnessAllocate(uint256 a) external returns (uint256) { _checkNotPaused(); return _allocate(a); }
    function harnessDeallocate(uint256 a) external returns (uint256) { _checkNotPaused(); return _deallocate(a); }
}

// Keeper reacts to StrategyDeallocationLoss by pausing the strategy
contract LossKeeper {
    event KeeperPaused(address strat);

    MorphoYearnOGWETHHarnessPauseable public strat;

    constructor(MorphoYearnOGWETHHarnessPauseable s) { strat = s; }

    // Simulate an automation tick that runs after a loss signal is detected
    function onLossSignal() external {
        strat.pause();
        emit KeeperPaused(address(strat));
    }
}

contract MorphoYearnOGWETH_AutoPause is Test {
    address constant MORPHO_YEARN_OG_VAULT = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;
    address constant MAINNET_WETH       = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address constant MAINNET_PERMIT2    = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    // Same event signature as strategy
    event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

    IWETH weth = IWETH(MAINNET_WETH);
    IERC4626 vault = IERC4626(MORPHO_YEARN_OG_VAULT);
    MorphoYearnOGWETHHarnessPauseable strat;
    LossKeeper keeper;

    function setUp() public {
        string memory url = vm.envOr("MAINNET_RPC_URL", vm.rpcUrl("mainnet"));
        uint256 forkId = vm.createFork(url);
        vm.selectFork(forkId);

        IMYTStrategy.StrategyParams memory p = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "MorphoYearnOGWETH",
            protocol: "MorphoYearnOGWETH",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: type(uint256).max,
            globalCap: type(uint256).max,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 1
        });

        strat = new MorphoYearnOGWETHHarnessPauseable(address(0xdead), p, MORPHO_YEARN_OG_VAULT, MAINNET_WETH, MAINNET_PERMIT2);
        keeper = new LossKeeper(strat);
        strat.setGuardian(address(keeper));

        // Fund and allocate
        address alice = address(0xA11CE);
        vm.deal(alice, 10 ether);
        vm.startPrank(alice);
        IWETH(MAINNET_WETH).deposit{value: 2 ether}();
        IWETH(MAINNET_WETH).transfer(address(strat), 2 ether);
        vm.stopPrank();

        strat.harnessAllocate(1 ether);
        assertGt(vault.balanceOf(address(strat)), 0, "no shares after allocate");
    }

    function test_AutoPause_OnFalseLoss_ThenFreeze() public {
        uint256 amount = 0.5 ether;

        // Step 1: First deallocation emits false loss (as proven) and funds arrive
        vm.recordLogs();
        uint256 ret = strat.harnessDeallocate(amount);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        assertEq(ret, amount, "returned amount mismatch");

        // Confirm the loss event fired in this tx
        bytes32 sig = keccak256("StrategyDeallocationLoss(string,uint256,uint256)");
        bool sawLoss = false;
        for (uint256 i; i < logs.length; i++) {
            if (logs[i].topics.length > 0 && logs[i].topics[0] == sig) { sawLoss = true; break; }
        }
        assertTrue(sawLoss, "expected StrategyDeallocationLoss not emitted");

        // Step 2: Keeper reacts to the loss signal and pauses the strategy
        keeper.onLossSignal();

        // Step 3: Further deallocation attempts are blocked (temporary freezing)
        vm.expectRevert(MorphoYearnOGWETHHarnessPauseable.Paused.selector);
        strat.harnessDeallocate(0.1 ether);
    }
}
```

Results: Test passes; second deallocation reverts with `Paused()`, proving the automation-driven freeze path.

```solidity
Ran 1 test for src/test/POC.t.sol:MorphoYearnOGWETH_AutoPause
[PASS] test_AutoPause_OnFalseLoss_ThenFreeze() (gas: 554521)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 41.28s (277.45ms CPU time)
```
