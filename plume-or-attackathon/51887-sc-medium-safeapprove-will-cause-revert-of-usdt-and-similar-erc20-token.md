# 51887 sc medium safeapprove will cause revert of usdt and similar erc20 token

**Submitted on Aug 6th 2025 at 13:16:42 UTC by @SAAJ for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51887
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

{% hint style="info" %}
Summary: Checking the boolean return of ERC20 `approve` breaks protocol compatibility with USDT and similar tokens because they do not return `true` on success. Repeated calls to `safeApprove` will revert for such tokens unless allowance is explicitly first set to zero.
{% endhint %}

## Vulnerability Details

The `TellerWithMultiAssetSupportPredicateProxy` contract calls a solmate-based `safeApprove` in its `deposit` and `depositAndBridge` methods:

```solidity
depositAsset.safeApprove(address(vault), depositAmount);
```

`depositAsset` is the ERC20 token that will be approved to the vault.

USDT (and similar non-EIP-20-compliant tokens) enforce that changing a non-zero allowance to another non-zero allowance requires first setting it to zero. The USDT code causing this behavior:

```solidity
// To change the approve amount you first have to reduce the addresses`
//  allowance to zero by calling `approve(_spender, 0)` if it is not
//  already 0 to mitigate the race condition described here:
//  https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
require(!((_value != 0) && (allowed[msg.sender][_spender] != 0)));
```

Because the contract calls `safeApprove` without ensuring the existing allowance is zero, a subsequent `safeApprove` for USDT (or similar tokens) will revert. This affects repeated calls to `deposit` and `depositAndBridge` by the same user/token combination, causing those calls to fail.

## Affected locations (code references)

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L89>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/0ee676b5715075c26db6706960fd49ab59b587fc/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol#L146>

## Recommendation

Properly handle approvals by checking whether an existing non-zero allowance exists; if so, first set it to zero before setting it to the new non-zero allowance. Apply this check to both `deposit` and `depositAndBridge`.

Suggested patch (illustrative):

```diff
        ERC20 vault = ERC20(teller.vault());

+       // If existing approval is non-zero -> set it to zero
+       if (depositAsset.allowance(msg.sender, address(vault)) != 0) {
+           depositAsset.safeApprove(address(vault), uint8(0));
+       }

        //approve vault to take assets from proxy
        depositAsset.safeApprove(address(vault), depositAmount);
```

Apply equivalent logic to both `deposit` and `depositAndBridge` methods so repeated deposits with USDT-like tokens don't revert.

## Proof of Concept

The following test reproduces the revert when calling `safeApprove` for USDT more than once. The snippet demonstrates a first successful approve and a second approve that reverts:

```solidity
// forge t -f $fork_rpc_url --etherscan-api-key $key_Api --mt test_USDTApprove -vvv
function test_USDTApprove() external {
    ERC20 USDT = ERC20(0xdAC17F958D2ee523a2206206994597C13D831ec7); // usdt address
    address random_User = 0x46340b20830761efd32832A74d7169B29FEB9758; // random User having USDT
    uint256 amount_Approve = 1000 * 1e6;

    vm.startPrank(random_User); // mock call made by user
    USDT.safeApprove(address(this), amount_Approve); // approving amount 
    uint256 approved_Balance = (vault).allowance(random_User, address(vault)); // caching approved amount
    console.log("Approved Balance: %e", approved_Balance);
    vm.stopPrank();

    vm.startPrank(random_User); // again calling the function when approve exist
    USDT.safeApprove(address(vault), amount_Approve); // this line reverts
}
```

<details>

<summary>Failure log (expanded)</summary>

Ran 1 test for test/PlumeTest.t.sol:implementUpgradeTest \[FAIL: APPROVE\_FAILED] test\_USDTApprove() (gas: 41079) Logs: Approved Balance: 1e9

Traces: \[41079] implementUpgradeTest::test\_USDTApprove() ├─ \[0] VM::startPrank(0x46340b20830761efd32832A74d7169B29FEB9758) │ └─ ← \[Return] ├─ \[26953] TetherToken::approve(implementUpgradeTest: \[0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 1000000000 \[1e9]) │ ├─ emit Approval(owner: 0x46340b20830761efd32832A74d7169B29FEB9758, spender: implementUpgradeTest: \[0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], v alue: 1000000000 \[1e9]) │ └─ ← \[Stop] ├─ \[1356] TetherToken::allowance(0x46340b20830761efd32832A74d7169B29FEB9758, implementUpgradeTest: \[0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496]) \[stat iccall] │ └─ ← \[Return] 1000000000 \[1e9] ├─ \[0] console::log("Approved Balance: %e", 1000000000 \[1e9]) \[staticcall] │ └─ ← \[Stop] ├─ \[0] VM::stopPrank() │ └─ ← \[Return] ├─ \[0] VM::startPrank(0x46340b20830761efd32832A74d7169B29FEB9758) │ └─ ← \[Return] ├─ \[903] TetherToken::approve(implementUpgradeTest: \[0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 1000000000 \[1e9]) │ └─ ← \[Revert] EvmError: Revert └─ ← \[Revert] APPROVE\_FAILED

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 6.49s (2.05s CPU time)

Ran 1 test suite in 7.62s (6.49s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests: Encountered 1 failing test in test/PlumeTest.t.sol:implementUpgradeTest \[FAIL: APPROVE\_FAILED] test\_USDTApprove() (gas: 41079)

Encountered a total of 1 failing tests, 0 tests succeeded

</details>
