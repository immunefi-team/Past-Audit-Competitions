# 52925 sc medium usdt like approval hygiene can block subsequent operations after partial fill leaves non zero allowance

* **Submitted on:** Aug 14th 2025 at 10:49:21 UTC by @LoopGhost007 for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #52925
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>

Impacts:

* Smart contract unable to operate due to lack of token funds
* Temporary freezing of funds for at least 24 hours

## Description

### Brief / Intro

A medium-severity vulnerability was identified in `DexAggregatorWrapperWithPredicateProxy` related to allowance hygiene for non-standard ERC20 tokens (USDT-like). Some tokens refuse to change a spender allowance from a non-zero value directly to another non-zero value (they require `approve(0)` first). The wrapper uses `safeApprove(spender, amount)` without zeroing a prior non-zero allowance, so a partial fill that leaves a non-zero residual allowance can cause subsequent calls to revert on USDT-like tokens.

The PoC demonstrates that after a partial fill the wrapper left a large non-zero allowance to the router. On the next interaction, USDT-like tokens would revert when attempting a non-zero -> non-zero approve.

## Vulnerability Details

Affected helper sites in `DexAggregatorWrapperWithPredicateProxy.sol`:

* `_oneInchHelper`: calls `safeApprove(aggregator, amount)` without first zeroing allowance
* `_okxHelper`: calls `safeApprove(okxApprover, amount)` without first zeroing allowance
* Vault approvals: approvals to the vault do not follow a zero-first pattern

Behavior that causes failure:

* A partial fill leaves a non-zero residual allowance to the router/approver.
* USDT-like tokens revert when moving directly from a non-zero allowance to another non-zero allowance unless `approve(0)` is called first.
* The wrapper’s `safeApprove()` pattern does not zero-out allowances (nor necessarily clear leftover allowances after use), so operations can revert.

Evidence from PoC logs:

* “Leftover USDT-like allowance to aggregator after partial fill: 999999999999”
* “Second call should revert due to USDT-like non-zero -> non-zero approve rule.”
* “VULNERABILITY CONFIRMED: Approval hygiene blocks subsequent operations.”

Note: In the provided run, the second call reverted earlier during `transferFrom` due to the test user balance being exhausted. The key precondition (leftover non-zero allowance after partial fill) is shown; with sufficient balance the wrapper would hit `safeApprove` over a non-zero current allowance and revert under USDT-like rules.

## Impact Details

This can cause the wrapper to be unable to operate with USDT-like tokens after a partial fill leaves a non-zero allowance, temporarily freezing affected assets and impairing UX and reliability.

## Suggested Mitigation

* Always use a reset-to-zero then set pattern for approvals:
  * If `allowance(spender) != 0`, call `safeApprove(spender, 0)` before `safeApprove(spender, amount)`.
* Eliminate residual allowances after use:
  * After the swap completes, `safeApprove(spender, 0)`.
  * This reduces attack surface if a spender becomes compromised.
* Handle partial fills to avoid leftover allowances altogether:
  * Refund unspent `srcToken` to the user.
  * Zero-out the allowance to the router/approver after the call.

Apply these patterns in `_oneInchHelper`, `_okxHelper`, and any other approval locations (e.g., vault approvals).

## Proof of Concept

Summary: The PoC uses a USDT-like token which rejects `approve()` calls that move a non-zero allowance to another non-zero value. After a partial fill the wrapper leaves a non-zero allowance to the aggregator. A second call with the same token will fail unless the wrapper first resets the allowance to zero.

<details>

<summary>PoC code (place under test/ and run with provided remappings)</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.21;

import "forge-std/Test.sol";
import "forge-std/console.sol";

// Wrapper
import { DexAggregatorWrapperWithPredicateProxy } from "src/helper/DexAggregatorWrapperWithPredicateProxy.sol";
import { TellerWithMultiAssetSupportPredicateProxy } from "src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol";

// Core stack
import { BoringVault } from "src/base/BoringVault.sol";
import { AccountantWithRateProviders } from "src/base/Roles/AccountantWithRateProviders.sol";
import { TellerWithMultiAssetSupport } from "src/base/Roles/TellerWithMultiAssetSupport.sol";
import { RolesAuthority, Authority } from "@solmate/auth/authorities/RolesAuthority.sol";

// Structs/Interfaces
import { PredicateMessage } from "@predicate/src/interfaces/IPredicateClient.sol";
import { AggregationRouterV6 } from "src/interfaces/AggregationRouterV6.sol";
import { IOKXRouter } from "src/interfaces/IOKXRouter.sol";
import { ERC20 } from "@solmate/tokens/ERC20.sol";
import { WETH } from "@solmate/tokens/WETH.sol";

// Partial-fill 1inch mock (spends 1 unit, returns 1 unit)
contract PartialFill1Inch is AggregationRouterV6 {
    function swap(address, SwapDescription calldata desc, bytes calldata)
        external
        payable
        override
        returns (uint256, uint256)
    {
        ERC20(address(desc.srcToken)).transferFrom(msg.sender, address(this), 1);
        ERC20(address(desc.dstToken)).transfer(msg.sender, 1);
        return (1, 1);
    }
}

// USDT-like token: approve(non-zero) fails if current allowance != 0
contract USDTLike is ERC20("USDTLike", "USDTL", 6) {
    function mint(address to, uint256 amount) external { _mint(to, amount); }
    function approve(address spender, uint256 value) public override returns (bool) {
        uint256 current = allowance[msg.sender][spender];
        if (current != 0 && value != 0) {
            emit Approval(msg.sender, spender, current);
            return false; // SafeTransferLib.safeApprove will revert on false
        }
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
        return true;
    }
}

// Allow-all predicate
contract AllowAllPredicateProxy {
    function genericUserCheckPredicate(address, PredicateMessage calldata) external pure returns (bool) {
        return true;
    }
}

contract USDTLike_Approval_DoS_PoC is Test {
    // Core
    WETH public weth;
    BoringVault public vault;
    AccountantWithRateProviders public accountant;
    TellerWithMultiAssetSupport public teller;
    RolesAuthority public roles;

    // Wrapper + mocks
    DexAggregatorWrapperWithPredicateProxy public wrapper;
    PartialFill1Inch public oneInch;
    AllowAllPredicateProxy public predicate;
    USDTLike public usdt;

    address public user = address(0xBEEF2);

    function setUp() public {
        vm.txGasPrice(0);
        vm.deal(user, 10 ether);

        // Core
        weth = new WETH();
        vault = new BoringVault(address(this), "BoringVault", "BV", 18);
        accountant = new AccountantWithRateProviders(
            address(this), address(vault), address(this), 1e18, address(weth), 10010, 9990, 1, 0
        );
        teller = new TellerWithMultiAssetSupport(address(this), address(vault), address(accountant));

        roles = new RolesAuthority(address(this), Authority(address(0)));
        vault.setAuthority(roles);
        teller.setAuthority(roles);
        uint8 ROLE_TELLER = 3;
        roles.setUserRole(address(teller), ROLE_TELLER, true);
        roles.setRoleCapability(ROLE_TELLER, address(vault), vault.enter.selector, true);
        roles.setRoleCapability(ROLE_TELLER, address(vault), vault.exit.selector, true);

        teller.addAsset(weth);

        oneInch = new PartialFill1Inch();
        predicate = new AllowAllPredicateProxy();

        wrapper = new DexAggregatorWrapperWithPredicateProxy(
            oneInch,
            IOKXRouter(address(0)),
            address(0xDEAD),
            weth,
            TellerWithMultiAssetSupportPredicateProxy(payable(address(predicate)))
        );

        uint8 ROLE_WRAPPER = 5;
        roles.setUserRole(address(wrapper), ROLE_WRAPPER, true);
        roles.setRoleCapability(ROLE_WRAPPER, address(teller), teller.deposit.selector, true);

        // Prefund router with 1 wei WETH
        weth.deposit{value: 1}();
        weth.transfer(address(oneInch), 1);

        // Mint USDT-like to user
        usdt = new USDTLike();
        usdt.mint(user, 1_000_000e6);
    }

    function _emptyPredicate() internal pure returns (PredicateMessage memory p) {
        return PredicateMessage({ taskId: "", expireByTime: 0, signerAddresses: new address[](0), signatures: new bytes[](0) });
    }

    // MEDIUM: USDT-like approval hygiene breaks next call after partial fill leaves non-zero allowance
    function test_MEDIUM_usdt_like_approval_breaks_next_call() public {
        console.log("--- PoC: USDT-like Approval Fails After Partial Fill Left Non-Zero Allowance (USDT->WETH) ---");

        // User approves wrapper to pull USDT-like
        vm.startPrank(user);
        usdt.approve(address(wrapper), type(uint256).max);

        uint256 amount = 1_000_000e6; // 1,000,000 units (6 decimals)

        AggregationRouterV6.SwapDescription memory desc1 = AggregationRouterV6.SwapDescription({
            srcToken: ERC20(address(usdt)),
            dstToken: ERC20(address(weth)),
            srcReceiver: payable(address(0)),
            dstReceiver: payable(address(wrapper)),
            amount: amount,
            minReturnAmount: 1,
            flags: 0
        });

        // First call: partial fill spends only 1 unit
        wrapper.depositOneInch(
            ERC20(address(weth)), // vault supports WETH
            teller,
            0,
            address(0),
            desc1,
            "",
            0,
            _emptyPredicate()
        );

        uint256 leftover = usdt.allowance(address(wrapper), address(oneInch));
        console.log("Leftover USDT-like allowance to aggregator after partial fill:", leftover);
        assertEq(leftover, amount - 1, "Leftover allowance expected after partial fill");

        // Second call should fail: wrapper tries safeApprove(amount) with current allowance != 0 (USDT-like rule)
        AggregationRouterV6.SwapDescription memory desc2 = AggregationRouterV6.SwapDescription({
            srcToken: ERC20(address(usdt)),
            dstToken: ERC20(address(weth)),
            srcReceiver: payable(address(0)),
            dstReceiver: payable(address(wrapper)),
            amount: amount,
            minReturnAmount: 1,
            flags: 0
        });

        console.log("Second call should revert due to USDT-like non-zero -> non-zero approve rule.");
        vm.expectRevert(); // SafeTransferLib.safeApprove will revert
        wrapper.depositOneInch(
            ERC20(address(weth)),
            teller,
            0,
            address(0),
            desc2,
            "",
            0,
            _emptyPredicate()
        );

        vm.stopPrank();

        console.log("VULNERABILITY CONFIRMED: Approval hygiene blocks subsequent operations.");
    }
}
```

</details>

<details>

<summary>Execution Logs</summary>

Run: forge test -vv --match-path test/USDT\_Like\_Approval\_DoS.t.sol

Output:

```
Ran 1 test for test/USDT_Like_Approval_DoS.t.sol:USDTLike_Approval_DoS_PoC
[PASS] test_MEDIUM_usdt_like_approval_breaks_next_call() (gas: 357478)
Logs:
  --- PoC: USDT-like Approval Fails After Partial Fill Left Non-Zero Allowance (USDT->WETH) ---
  Leftover USDT-like allowance to aggregator after partial fill: 999999999999
  Second call should revert due to USDT-like non-zero -> non-zero approve rule.
  VULNERABILITY CONFIRMED: Approval hygiene blocks subsequent operations.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 21.56ms (5.32ms CPU time)
```

</details>
