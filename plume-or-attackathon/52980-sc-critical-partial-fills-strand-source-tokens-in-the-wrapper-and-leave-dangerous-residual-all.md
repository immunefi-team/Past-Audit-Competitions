# 52980 sc critical partial fills strand source tokens in the wrapper and leave dangerous residual allowances

**Submitted on Aug 14th 2025 at 14:48:17 UTC by @RevertLord for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52980
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

A critical flaw exists in `DexAggregatorWrapperWithPredicateProxy` that manifests on partial fills. When a swap spends less than the full `srcToken` amount, the wrapper:

* never returns the unspent remainder to the user, and
* leaves a non-zero allowance outstanding to the router.

This traps user funds inside the wrapper and exposes dangling approvals that can be abused if the external spender (router/aggregator) is compromised.

The behavior is reproducible on the 1inch path and applies analogously to the OKX path.

## Vulnerability Details

{% stepper %}
{% step %}

### 1inch path (non-native)

* The wrapper pulls `desc.amount` of `srcToken` from the user and approves the router for that full amount.
* `aggregator.swap(executor, desc, data)` returns `(returnAmount, spentAmount)`, but the wrapper ignores `spentAmount`.
* If `spentAmount < desc.amount`:
  * The difference (`desc.amount − spentAmount`) remains held by the wrapper.
  * The leftover allowance (same difference) stays granted to the router.
  * There is no refund of unspent `srcToken` to the user.
    {% endstep %}

{% step %}

### Native variant

* The wrapper wraps ETH into WETH and approves the router for the WETH.
* If the router spends only part of the WETH, the remainder stays in the wrapper with an allowance still set.
  {% endstep %}

{% step %}

### OKX path

* The same approval-before-execution pattern exists.
* If the router consumes less than the approved amount, residual tokens and allowances are left behind.
* There is no refund mechanism for unspent balances.
  {% endstep %}
  {% endstepper %}

Root cause: the implementation never reconciles what was actually spent versus what was fronted and approved, and it does not sanitize approvals after execution.

## Impact Details

* Severity: Critical
* In-scope impact: Permanent freezing of user funds (the wrapper has no API to recover the stranded `srcToken`), plus a latent risk of direct theft via dangling approvals if the external spender becomes malicious or is exploited.

## Suggested Mitigation

* Refund unspent balances:
  * Read and use `spentAmount` returned by the 1inch router. Compute `unspent = desc.amount − spentAmount`. If `unspent > 0`, return the unspent `srcToken` to `msg.sender` (or return WETH/unwrapped ETH for the native path).
  * On OKX, derive “spent” via before/after balance deltas and similarly refund any unspent amount.
* Scrub approvals:
  * After the swap completes, set allowance(spender, 0).
  * Prefer the “reset to zero then set” pattern when granting fresh allowances to support USDT-like tokens.
* Consider using `permit` + exact pull semantics or streaming approvals tightly scoped to the execution.

## Proof of Concept

A Foundry test (`PartialFillTrappedResidual.t.sol`) configures a mock 1inch router that spends only 1 wei and returns 1 wei as output. The wrapper still holds almost the entire `srcToken` amount, and the leftover allowance to the router equals that trapped amount.

Key logs from the PoC:

* “Wrapper residual WETH (trapped): 999999999999999999”
* “Residual allowance to aggregator: 999999999999999999”

This confirms both the fund-stranding and the unsafe residual approval.

### PoC Code

Place the following PoC file inside the `test` folder in `attackathon-plume-network-nucleus-boring-vault`. Install all required dependencies and libraries and use the provided `remappings.txt` (unchanged). Once done, create the `PartialFillTrappedResidual.t.sol` file and paste the following code:

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

### Execution Logs (how to run)

Run:

```
forge test -vv --match-path test/PartialFillTrappedResidual.t.sol
```

Expected logs (example):

<details>

<summary>Execution Logs</summary>

Ran 1 test for test/PartialFillTrappedResidual.t.sol:PartialFill\_TrappedResidual\_PoC \[PASS] test\_CRITICAL\_partialFill\_traps\_srcToken\_and\_leaves\_allowance() (gas: 337734) Logs: --- PoC: Partial Fill Freezes Residual srcToken in Wrapper (WETH->WETH) --- Wrapper residual WETH (trapped): 999999999999999999 Residual allowance to aggregator: 999999999999999999 VULNERABILITY CONFIRMED (CRITICAL): Partial fill froze the srcToken residual in the wrapper.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 34.97ms (8.08ms CPU time)

</details>

***

If you want, I can:

* produce a patch suggestion (code diff) that implements the mitigations: reconciling `spentAmount`, refunding unspent tokens, and zeroing approvals safely; or
* generate a minimal test demonstrating a fix.
