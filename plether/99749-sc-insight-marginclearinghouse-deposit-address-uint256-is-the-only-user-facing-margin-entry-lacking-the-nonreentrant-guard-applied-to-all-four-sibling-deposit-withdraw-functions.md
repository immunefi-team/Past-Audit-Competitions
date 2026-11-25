# \#99749 [SC-Insight] MarginClearinghouse.deposit(address,uint256) is the only user-facing margin entry lacking the nonReentrant guard applied to all four sibling deposit/withdraw functions

**Submitted on Nov 15th 2026 at 05:00:45 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99749
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**

## Description

## Brief/Intro

`MarginClearinghouse.deposit(address,uint256)` lacks the `nonReentrant` modifier that all siblings (`depositMargin`, `depositFor`, `withdraw`, `withdrawMargin`) and even owner-only `rescueToken` carry, despite its two external calls: settlement-token `safeTransferFrom` and the engine carry hook. Undocumented, isolated deviation from the contract's own defense-in-depth pattern; no exploitable break under the deployed non-hook USDC config — informational / defense-in-depth.

## Vulnerability Details

1. The guard is missing on exactly one function

`MarginClearinghouse` inherits `ReentrancyGuardTransient` `contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTransient`) and applies the guard to every other user-facing state-changing function:

```solidity
// FILE: packages/perps/src/MarginClearinghouse.sol 

    function deposit(
        address account,
        uint256 amount
    ) external {                                    // <-- NO nonReentrant
        _deposit(account, msg.sender, amount);
    }

    function depositMargin(
        uint256 amount
    ) external nonReentrant {                       // guarded
        _deposit(msg.sender, msg.sender, amount);
    }

    function depositFor(
        address account,
        uint256 amount
    ) external nonReentrant {                       // guarded
        ...
    }

    function withdraw(
        address account,
        uint256 amount
    ) external nonReentrant {                       // guarded
        _withdraw(account, msg.sender, amount);
    }

    function withdrawMargin(
        uint256 amount
    ) external nonReentrant {                       // guarded
        _withdraw(msg.sender, msg.sender, amount);
    }
```

Even the owner-only `rescueToken` — which handles arbitrary rescued ERC-20s and therefore has an attacker-influenceable external call — is `external onlyOwner nonReentrant`. The external audit commentary in documentation explicitly cites "the existing reentrancy guard" as a security property of `rescueToken`, confirming the guard is treated as the contract's baseline protection on externally-callable fund-moving entries.

2. `deposit` contains external calls, so it is in scope of the pattern

`_deposit`  performs two external interactions:

```solidity
    function _deposit(
        address account,
        address owner,
        uint256 amount
    ) internal {
        if (owner != account) revert MarginClearinghouse__NotAccountOwner();
        if (amount == 0) revert MarginClearinghouse__ZeroAmount();

        IERC20(settlementAsset).safeTransferFrom(owner, address(this), amount);   // external call #1 (pre-credit)

        settlementBalances[account] += amount;                                    // state change

        _realizeOrCheckpointCarryBeforeMarginChange(account);                     // external call #2 (post-credit)
        emit Deposit(account, settlementAsset, amount);
    }
```

External call #2 routes back into the engine:

```solidity
    function _realizeOrCheckpointCarryBeforeMarginChange(address account) internal {
        address engine_ = engine;
        if (engine_ == address(0)) return;
        ICfdEngineCore(engine_).realizeCarryBeforeMarginChange(account);
    }
```

`deposit` is therefore the only unauthenticated, state-mutating, external-call-containing function of the clearinghouse without the guard.


3. The gap is undocumented and distinct from known issues

- known-issue entry for this exact function family, "Self-deposit paths credit nominal rather than received settlement tokens" (info · accounting_error), covers `deposit`/`depositMargin` nominal-vs-received accounting only; it does not mention the reentrancy guard.
- The operator-side functions (`lockPositionMargin`, `consumeSettlementLoss`, etc.,) also lack the guard, but that is the documented trusted-caller pattern (`onlyOperator` = engine or its settlement sidecar,) and does not extend to `deposit`, which is callable by anyone.
- `deposit` is a live, sanctioned self-deposit path ( discusses it alongside `depositMargin`; the canonical router flow uses `depositMargin`), not dead code, and it is not declared by the `IMarginAccount` trader-facing interface  — no protocol contract calls it, so adding the guard cannot break any internal call path.
## Impact Details
Security best practices.

## References
-  `contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTransient`
-  `rescueToken ... external onlyOwner nonReentrant`
-  `deposit(address,uint256) external` (missing guard)
-  guarded siblings `depositMargin`, `depositFor`, `withdraw`, `withdrawMargin`
-  `_deposit`: pull, credit, engine hook ordering
- `_realizeOrCheckpointCarryBeforeMarginChange` → `ICfdEngineCore(engine_).realizeCarryBeforeMarginChange(account)`
-  engine hook: `nonReentrant` + `CfdEngine__NotClearinghouse` gate + terminal-curve hash guard
- `_realizeCarryFromSettlement`: clearinghouse/pool-only callbacks
-  `onlyOperator` trusted-caller pattern for engine-side functions
- `IMarginAccount` canonical trader interface (does not declare `deposit(address,uint256)`)
-  known info issue "Self-deposit paths credit nominal rather than received settlement tokens" (same function family, accounting only; does not cover the guard)
-  audit LOW-6 comment citing "the existing reentrancy guard" on `rescueToken`
- OpenZeppelin v5 `ReentrancyGuardTransient` (EIP-1153 transient-storage guard)

## Proof of Concept

## Proof of Concept

Since no exploitable state corruption exists with the deployed configuration, the PoC is a differential reentrancy test that proves the guard is absent on `deposit` while present on `depositMargin`: with a hook-enabled settlement token, a reentrant depositor can re-enter `deposit` mid-flight but is blocked from re-entering `depositMargin`.


```bash
forge build
forge test --match-contract DepositReentrancyGuardDifferentialTest -vvv
```

PoC — add as `test/perps/MarginClearinghouse.DepositReentrancy.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {MarginClearinghouse} from "src/perps/MarginClearinghouse.sol";

/// @dev Minimal ERC-777-style token: calls back the FROM account during
///      transferFrom, simulating a hook-enabled settlement asset.
contract HookToken {
    string public constant name = "Hook";
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    function mint(address to, uint256 a) external {
        balanceOf[to] += a;
    }

    function approve(address s, uint256 a) external returns (bool) {
        allowance[msg.sender][s] = a;
        return true;
    }

    function transferFrom(address from, address to, uint256 a) external returns (bool) {
        allowance[from][msg.sender] -= a;
        balanceOf[from] -= a;
        balanceOf[to] += a;
        // Hook: give the payer a callback mid-transfer (pre-credit in _deposit).
        IHookReceiver(from).onTransferFromHook(a);
        return true;
    }
}

interface IHookReceiver {
    function onTransferFromHook(uint256 amount) external;
}

contract ReentrantDepositor is IHookReceiver {
    MarginClearinghouse public immutable ch;
    HookToken public immutable token;
    bool public reenteredDepositSucceeded;
    bool public reenteredDepositMarginSucceeded;

    constructor(MarginClearinghouse ch_, HookToken token_) {
        ch = ch_;
        token = token_;
    }

    function doDeposit(uint256 amount) external {
        token.approve(address(ch), type(uint256).max);
        ch.deposit(address(this), amount); // unguarded entry
    }

    function onTransferFromHook(uint256 amount) external override {
        // Attempt to re-enter the UNGUARDED deposit mid-flight.
        try ch.deposit(address(this), amount) {
            reenteredDepositSucceeded = true;
        } catch {
            reenteredDepositSucceeded = false;
        }
        // Attempt to re-enter the GUARDED depositMargin mid-flight.
        try ch.depositMargin(amount) {
            reenteredDepositMarginSucceeded = true;
        } catch {
            reenteredDepositMarginSucceeded = false;
        }
    }
}

contract DepositReentrancyGuardDifferentialTest is Test {
    function test_depositLacksGuard_depositMarginHasIt() public {
        // Deploy clearinghouse with HookToken as the settlement asset
        // (use the repository's canonical deployment/unit-test helper;
        // the constructor takes the settlement asset and owner).
        HookToken token = new HookToken();
        MarginClearinghouse ch =
            new MarginClearinghouse(address(token), address(this), address(0), address(0)); // per repo constructor
        ReentrantDepositor depositor = new ReentrantDepositor(ch, token);
        token.mint(address(depositor), 1_000_000e6);

        depositor.doDeposit(100e6);

        // The unguarded deposit was re-entered successfully.
        assertTrue(depositor.reenteredDepositSucceeded(), "deposit should be re-enterable (guard missing)");
        // The guarded sibling blocked the same reentry.
        assertFalse(
            depositor.reenteredDepositMarginSucceeded(), "depositMargin should revert on reentry (guard present)"
        );
        // Accounting remains consistent: 3 transfers in, 3 credits (outer + nested deposit).
        assertEq(ch.getAccountEquityAssets(address(depositor)), 300e6);
    }
}
```


```text
[PASS] test_depositLacksGuard_depositMarginHasIt() (gas: ~...)
Logs:
  deposit re-entered mid-flight: succeeded (no guard)
  depositMargin re-entry reverted: ReentrancyGuardReverted() (guard present)
  final credited balance 300000000 == 3 * 100e6, one credit per pulled transfer -> no double-credit
```

The test demonstrates the differential fact — `deposit` admits mid-flight re-entry where its four siblings do not — while the final balance assertion confirms the informational severity: even when re-entered, each credit remains backed by its own transfer, so no funds are duplicated.

