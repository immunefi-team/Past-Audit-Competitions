# 57685 sc critical vulnerabilities in the design of the token s staking mechanism resulted in financial harm to users involved in transfer related operations&#x20;

**Submitted on Oct 28th 2025 at 06:47:47 UTC by @queen for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57685
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:** Vulnerabilities in the design of the token's staking mechanism resulted in financial harm to users involved in transfer-related operations.

## Description

### Brief/Intro

The `Staking.sol` ERC4626 vault mints and allows transfer of ERC4626 shares while lock metadata (per-deposit `stakes`) is tracked separately by address. When a user transfers unlocked shares to another address, the recipient receives ERC4626 shares but no corresponding lock entries. As a result the recipient cannot perform normal withdrawals (which require consuming unlocked stakes) and is forced to use the emergency withdrawal path, incurring a penalty. This leads to direct economic loss and breaks expected unlock semantics.

### Vulnerability Details

* The contract stores lock state per address:

```solidity
mapping(address staker => Stake[] times) public stakes;
```

* On deposit, the contract mints ERC4626 shares and appends a lock entry for the recipient:

```solidity
function _deposit(address by, address to, uint256 assets, uint256 shares) internal override {
    super._deposit(by, to, assets, shares);
    // lock freshly minted shares
    stakes[to].push(Stake({shares: shares, timestamp: block.timestamp}));
}
```

* On normal withdrawal, the contract consumes “unlocked shares” from `stakes[owner]` and reverts if locks aren’t satisfied:

```solidity
function _withdraw(address by, address to, address _owner, uint256 assets, uint256 shares) internal override {
    _consumeUnlockedSharesOrRevert(_owner, shares);
    super._withdraw(by, to, _owner, assets, shares);
}
```

```solidity
function _consumeUnlockedSharesOrRevert(address staker, uint256 need) internal {
    // consumes only stakes whose lock period has elapsed; otherwise reverts MinStakePeriodNotMet()
    ...
    if (remaining != 0) revert MinStakePeriodNotMet();
}
```

* If a user transfers ERC4626 shares (permitted by ERC4626/solady ERC20 implementation), the contract does not migrate the corresponding `stakes[from]` entries to `stakes[to]`. The recipient thus owns shares without any lock entries. When they attempt a standard withdrawal, `_consumeUnlockedSharesOrRevert(to, ...)` finds no unlocked stake entries and reverts. The only available path becomes the emergency withdrawal, which always applies a penalty:

```solidity
function _emergencyWithdraw(...) internal {
    ...
    uint256 penalty = FixedPointMathLib.fullMulDiv(assets, penaltyPercentage, SCALING_FACTOR);
    ...
    LONG.safeTransfer(to, payout);
    LONG.safeTransfer(treasury, penalty);
}
```

* This occurs even if the transferred shares were already fully unlocked at the sender address. Unlock state is tied to `stakes[owner]` and does not follow share transfers.

Root cause:

* ERC4626 shares are transferable.
* No transfer hook overrides to block transfers or migrate `stakes` on `transfer/transferFrom`.
* Withdrawal logic requires lock metadata on the current owner, not on the original depositor.

### Impact Details

* Forced penalty loss: Users who wait out the minimum stake period and then transfer their unlocked shares to another wallet will be unable to perform a standard withdrawal and will incur the emergency withdrawal penalty (default 10%), causing direct economic loss.
* UX/DoS of normal withdrawal: Recipients of transferred shares are effectively denied the standard withdraw path due to missing lock entries, despite holding shares that semantically should be unlocked.
* Griefing / integration risk: Any workflow that moves shares between wallets (custodial moves, account abstractions, migrations) may unintentionally convert “unlocked” assets into “penalized-only” assets.
* Severity: High for affected users (loss proportional to position size × penalty). The issue undermines the advertised semantics of time-locked staking that unlocks after min period.

### Concrete scenario

{% stepper %}
{% step %}

### Depositor stakes and waits

User A deposits LONG and waits past `minStakePeriod` (all shares matured).
{% endstep %}

{% step %}

### Transfer of unlocked shares

User A transfers all shares to User B (e.g., moving to a hardware wallet).
{% endstep %}

{% step %}

### Normal withdrawal fails for recipient

User B attempts to withdraw; `_consumeUnlockedSharesOrRevert(B, …)` finds no unlocked entries (there are none), so it reverts.
{% endstep %}

{% step %}

### Emergency path imposes penalty

User B must use emergency withdrawal and loses `penaltyPercentage` of assets.
{% endstep %}
{% endstepper %}

## References (code pointers)

* Lock state tracking and deposit hook:

```solidity
mapping(address staker => Stake[] times);
```

```solidity
function _deposit(...) internal override {
    ...
    stakes[to].push(...);
}
```

* Normal withdraw requires unlocked stakes; otherwise revert:

```solidity
function _withdraw(...) internal override {
    _consumeUnlockedSharesOrRevert(_owner, shares);
    ...
}
```

```solidity
function _consumeUnlockedSharesOrRevert(...) internal { ... if (remaining != 0) revert MinStakePeriodNotMet(); }
```

* Emergency withdrawal applies penalty:

```solidity
function _emergencyWithdraw(...) internal { 
    uint256 penalty = ...;
    LONG.safeTransfer(to, payout);
    LONG.safeTransfer(treasury, penalty);
}
```

## Suggested remediations

{% hint style="info" %}
Do not add any behavior or code beyond what's suggested here — these are remediation patterns only.
{% endhint %}

* Option A — Make shares non-transferable: override ERC20 transfer hooks (transfer / transferFrom) to revert, preserving simple per-address lock semantics.
* Option B — Migrate stake metadata on transfers: implement logic in transfer hooks to split/merge/copy stake entries so that lock timestamps follow shares. This must correctly handle partial transfers and gas implications (stake array management, consolidation).
* Option C — Change withdrawal semantics: allow consumption of unlocked shares based on on-chain share ownership and original deposit timestamps (more complex; requires design to prevent circumvention).
* Consider documenting the exact semantics for integrators and users (e.g., "shares are transferable but lock metadata is per-address" is surprising and dangerous).

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.27;

import "forge-std/Test.sol";
import {Staking} from "contracts/v2/periphery/Staking.sol";

// Minimal mintable ERC20 for LONG
contract MockERC20 {
    string public name = "LONG";
    string public symbol = "LONG";
    uint8 public decimals = 18;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 amount);
    event Approval(address indexed owner, address indexed spender, uint256 amount);

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        uint256 a = allowance[from][msg.sender];
        require(a >= amount, "allowance");
        unchecked { allowance[from][msg.sender] = a - amount; }
        _transfer(from, to, amount);
        return true;
    }

    function _transfer(address from, address to, uint256 amount) internal {
        require(balanceOf[from] >= amount, "balance");
        unchecked { balanceOf[from] -= amount; balanceOf[to] += amount; }
        emit Transfer(from, to, amount);
    }
}

contract StakingTransferPoC is Test {
    MockERC20 LONG;
    Staking staking;

    address ALICE = makeAddr("ALICE");
    address BOB   = makeAddr("BOB");
    address TREASURY = makeAddr("TREASURY");

    function setUp() public {
        LONG = new MockERC20();
        staking = new Staking();
        // owner = this, treasury = TREASURY, underlying = LONG
        staking.initialize(address(this), TREASURY, address(LONG));

        // Mint LONG to ALICE and approve staking
        uint256 amount = 100 ether;
        LONG.mint(ALICE, amount);
        vm.prank(ALICE);
        LONG.approve(address(staking), amount);

        // ALICE deposits, creating a stake entry with lock timestamp
        vm.prank(ALICE);
        staking.deposit(amount, ALICE);

        // Wait past minStakePeriod so ALICE's shares are fully unlocked
        vm.warp(block.timestamp + 2 days);
    }

    function test_TransferUnlockedSharesBreaksWithdrawForRecipient() public {
        uint256 aliceShares = staking.balanceOf(ALICE);
        assertGt(aliceShares, 0, "no shares");

        // ALICE transfers all unlocked shares to BOB
        vm.prank(ALICE);
        staking.transfer(BOB, aliceShares);

        // BOB holds shares but has no stakes[] entries → normal withdraw should revert
        vm.prank(BOB);
        vm.expectRevert(Staking.MinStakePeriodNotMet.selector);
        staking.redeem(aliceShares, BOB, BOB); // or withdraw equivalent assets

        // Emergency redeem works but applies penalty (default 10%)
        uint256 bobLongBefore = LONG.balanceOf(BOB);
        uint256 treasuryBefore = LONG.balanceOf(TREASURY);

        vm.prank(BOB);
        uint256 assetsPrePenalty = staking.emergencyRedeem(aliceShares, BOB, BOB);

        uint256 bobLongAfter = LONG.balanceOf(BOB);
        uint256 treasuryAfter = LONG.balanceOf(TREASURY);

        // Expect 10% penalty → BOB receives ~90% of assets
        uint256 payout = bobLongAfter - bobLongBefore;
        uint256 penalty = treasuryAfter - treasuryBefore;

        assertEq(assetsPrePenalty, payout + penalty, "accounting mismatch");
        assertApproxEqRel(payout, (assetsPrePenalty * 90) / 100, 1e14, "payout not ~90%");
        assertApproxEqRel(penalty, (assetsPrePenalty * 10) / 100, 1e14, "penalty not ~10%");
    }
}
```

* Run with Foundry:
  * Place as `test/StakingTransferPoC.t.sol`
  * `forge test -m test_TransferUnlockedSharesBreaksWithdrawForRecipient`
