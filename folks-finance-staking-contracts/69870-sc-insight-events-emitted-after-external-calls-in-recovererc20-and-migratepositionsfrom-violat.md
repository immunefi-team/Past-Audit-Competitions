# 69870 sc insight events emitted after external calls in recovererc20 and migratepositionsfrom violate cei pattern

**Submitted on Mar 17th 2026 at 07:43:59 UTC by @hyuunn for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69870
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**

## Description

### Brief/Intro

Two functions (`recoverERC20` and `migratePositionsFrom`) emit their effect events after external `safeTransfer` calls, violating the Checks-Effects-Interactions (CEI) pattern. The same contract correctly follows CEI in `_stake` and `_withdraw`, demonstrating this is an oversight rather than a design choice.

### Vulnerability Details

In `recoverERC20` (`Staking.sol:159-160`), the external call executes before the event:

```solidity
IERC20(tokenAddress).safeTransfer(msg.sender, tokenAmount);  // L159: interaction
emit Recovered(tokenAddress, tokenAmount);                    // L160: effect after
```

In `migratePositionsFrom` (`Staking.sol:206-208`):

```solidity
TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);  // L206: interaction
emit MigrateFrom(msg.sender, user);                                          // L208: effect after
```

The same contract correctly follows CEI elsewhere:

```solidity
// _stake (L294-295): effect before interaction ✓
emit Staked(msg.sender, periodIndex, params.referrer, stakeIndex, amount);
TOKEN.safeTransferFrom(msg.sender, address(this), amount);

// _withdraw (L328-329): effect before interaction ✓
emit Withdrawn(msg.sender, stakeIndex, amountToClaim, rewardToClaim);
TOKEN.safeTransfer(msg.sender, amountToClaim + rewardToClaim);
```

Both `recoverERC20` and `migratePositionsFrom` are protected by `nonReentrant` (migratePositionsFrom) or role-gated (recoverERC20), so this is not directly exploitable. However, the inconsistency violates the security best practice of uniform CEI ordering.

### Impact Details

Event-driven off-chain systems that reconstruct contract state from transaction logs expect effects (events) to be recorded before interactions (external calls). The inconsistent ordering between functions in the same contract means off-chain indexers must handle two different event ordering patterns depending on which function was called.

### References

* `src/Staking.sol:159-160` — `recoverERC20`: safeTransfer before Recovered event
* `src/Staking.sol:206-208` — `migratePositionsFrom`: safeTransfer before MigrateFrom event
* `src/Staking.sol:294-295` — `_stake`: correctly emits before safeTransferFrom
* `src/Staking.sol:328-329` — `_withdraw`: correctly emits before safeTransfer

## Proof of Concept

1. Manager calls `recoverERC20(tokenAddress, amount)` to recover excess tokens
2. `safeTransfer` emits an ERC20 `Transfer` event at log index N
3. `Recovered` event is emitted at log index N+1 (after the transfer)
4. Similarly, migrator calls `migratePositionsFrom(user)`
5. `safeTransfer` emits `Transfer` at log index M
6. `MigrateFrom` is emitted at log index M+1 (after the transfer)
7. By contrast, `_stake` emits `Staked` before `Transfer` — proving the contract's own standard is effects-before-interactions

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {Staking} from "src/Staking.sol";
import {IStakingV1} from "src/interfaces/IStakingV1.sol";
import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract MockFolksTokenF005 is ERC20 {
    constructor() ERC20("Folks", "FOLKS") {}
    function mint(address to, uint256 amount) external { _mint(to, amount); }
}

contract F_U1_005_Test is Test {
    Staking staking;
    MockFolksTokenF005 token;
    address admin = makeAddr("admin");
    address manager = makeAddr("manager");
    address pauser = makeAddr("pauser");
    address migrator = makeAddr("migrator");
    address user = makeAddr("user");

    function setUp() public {
        vm.startPrank(admin);
        token = new MockFolksTokenF005();
        staking = new Staking(admin, manager, pauser, address(token));
        staking.grantRole(staking.MIGRATOR_ROLE(), migrator);
        vm.stopPrank();
        token.mint(address(staking), 100_000e18);
        vm.prank(manager);
        staking.addStakingPeriod(1_000_000e18, 30 days, 7 days, 1000, true);
        token.mint(user, 1_000e18);
        vm.startPrank(user);
        token.approve(address(staking), type(uint256).max);
        staking.stake(0, 1_000e18, IStakingV1.StakeParams(30 days, 7 days, 1000, address(0)));
        staking.setMigrationPermit(migrator, true);
        vm.stopPrank();
    }

    function testRecoverERC20EventAfterTransfer() public {
        token.mint(address(staking), 1e18);
        vm.recordLogs();
        vm.prank(manager);
        staking.recoverERC20(address(token), 1e18);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 transferSig = keccak256("Transfer(address,address,uint256)");
        bytes32 recoveredSig = keccak256("Recovered(address,uint256)");
        int256 tIdx = -1; int256 rIdx = -1;
        for (uint256 i = 0; i < logs.length; i++) {
            if (logs[i].topics[0] == transferSig && tIdx == -1) tIdx = int256(i);
            if (logs[i].topics[0] == recoveredSig) rIdx = int256(i);
        }
        assertLt(tIdx, rIdx, "Transfer precedes Recovered — CEI violated");
    }

    function testMigrateEventAfterTransfer() public {
        vm.recordLogs();
        vm.prank(migrator);
        staking.migratePositionsFrom(user);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 transferSig = keccak256("Transfer(address,address,uint256)");
        bytes32 migrateSig = keccak256("MigrateFrom(address,address)");
        int256 tIdx = -1; int256 mIdx = -1;
        for (uint256 i = 0; i < logs.length; i++) {
            if (logs[i].topics[0] == transferSig && tIdx == -1) tIdx = int256(i);
            if (logs[i].topics[0] == migrateSig) mIdx = int256(i);
        }
        assertLt(tIdx, mIdx, "Transfer precedes MigrateFrom — CEI violated");
    }
}
```

**Run:** `FOUNDRY_PROFILE=poc forge test --match-contract F_U1_005_Test -vvvv`
