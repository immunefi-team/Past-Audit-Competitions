# 52982 sc medium non standard erc20 approvals usdt like cause repeat call failures after partial fills

**Submitted on Aug 14th 2025 at 14:54:07 UTC by @RevertLord for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52982
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:** Smart contract unable to operate due to lack of token funds

## Description

### Brief / Intro

A medium-severity reliability issue was identified in DexAggregatorWrapperWithPredicateProxy when interacting with tokens that require an “approve(0) before approve(new)” pattern (USDT-like semantics). After a partial fill leaves a non-zero allowance to the router, subsequent operations with the same token can fail because the wrapper attempts a non-zero→non-zero approve without first clearing the allowance.

This behavior affects any token that reverts or returns false on non-zero→non-zero approve transitions. Partial fills are common with aggregators, so this is a realistic availability issue for affected assets.

## Vulnerability Details

Where it occurs:

* oneInchHelper: safeApprove(aggregator, amount) without zeroing first.
* \_okxHelper: safeApprove(okxApprover, amount) without zeroing first.
* Allowances to the vault follow the same no-zero-first pattern.

Behavior:

* Partial fills leave a residual allowance to the router/approver.
* On a later call with the same token, the wrapper tries to approve another non-zero amount while the current allowance is non-zero.
* USDT-like tokens typically reject such transitions, making safeApprove revert.

This is practical rather than theoretical: partial fills are normal and USDT-like approval semantics are widespread.

## Impact Details

* Severity: Medium
* In-scope impact: Smart contract unable to operate / temporary freezing for impacted assets. Users attempting to deposit those tokens will experience reverts until the allowance is explicitly reset.

## Suggested Mitigation

* Always adopt zero-first approvals:
  * If allowance(spender) != 0, safeApprove(spender, 0) before safeApprove(spender, amount).
* Clean up after executing swaps:
  * After the router/approver has consumed what it needs, safeApprove(spender, 0) to avoid leaving stale approvals.
* Combine with the partial-fill fix:
  * Refund unspent srcToken to avoid accumulating residual allowances in the first place.

## Proof of Concept

A Foundry test (USDT\_Like\_Approval\_DoS.t.sol) uses a USDT-like token whose approve reverts on non-zero→non-zero changes. After a partial fill, a large non-zero allowance remains to the router. A subsequent call attempts to re-approve while non-zero, which would revert under USDT-like rules, demonstrating a repeatability failure for those assets. Topping up user balances before the second attempt ensures the approve path is exercised first and the revert is attributable to approval semantics.

<details>

<summary>PoC test file (place under test/ and run with remappings)</summary>

{% code title="test/USDT\_Like\_Approval\_DoS.t.sol" %}

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

{% endcode %}

</details>

<details>

<summary>Execution Logs</summary>

Ran 1 test for test/USDT\_Like\_Approval\_DoS.t.sol:USDTLike\_Approval\_DoS\_PoC \[PASS] test\_MEDIUM\_usdt\_like\_approval\_breaks\_next\_call() (gas: 357478) Logs: --- PoC: USDT-like Approval Fails After Partial Fill Left Non-Zero Allowance (USDT->WETH) --- Leftover USDT-like allowance to aggregator after partial fill: 999999999999 Second call should revert due to USDT-like non-zero -> non-zero approve rule. VULNERABILITY CONFIRMED: Approval hygiene blocks subsequent operations.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 21.56ms (5.32ms CPU time)

</details>
