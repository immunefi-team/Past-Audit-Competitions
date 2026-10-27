# 56368 sc insight alchemisttokenvault deposit should use safetransferfrom instead of transferfrom alchemisttokenvault withdraw should use safetransfer instead of transfer&#x20;

**Submitted on Oct 15th 2025 at 07:08:20 UTC by @joicygiore for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56368
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistTokenVault.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The `ERC20.transfer()` and `ERC20.transferFrom()` functions return a boolean value indicating whether the operation succeeded. Callers must check this return value to ensure that the transfer actually succeeded. Some tokens do not revert on failure and instead return `false`.

## Vulnerability Details

As marked with `@>`, some tokens (e.g., USDT, BNB) do not fully implement the EIP20 standard. Their `transfer` or `transferFrom` functions return `void` instead of a boolean. Calling these functions using the standard ERC20 function signature will always result in a revert.

```js
    // AlchemistTokenVault::deposit()
    function deposit(uint256 amount) external {
        _checkNonZeroAmount(amount);
@>        IERC20(token).transferFrom(msg.sender, address(this), amount);
        emit Deposited(msg.sender, amount);
    }

    // AlchemistTokenVault::withdraw()
    function withdraw(address recipient, uint256 amount) external override onlyAuthorized {
        _checkNonZeroAddress(recipient);
        _checkNonZeroAmount(amount);


@>        IERC20(token).transfer(recipient, amount);
        emit Withdrawn(recipient, amount);
    }
```

## Impact Details

For tokens that do not comply with the latest EIP20 standard (e.g., USDT, BNB), even if the transfer returns `false`, the contract may consider it successful, resulting in failed operations. This makes such tokens unusable within the protocol.

## References

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistTokenVault.sol#L26-L43>

## Proof of Concept

## Proof of Concept

Add the following test to `src/test/AlchemistTokenVault.t.sol` and run it:

```js
    import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

    function test_poc_AlchemistTokenVault() public {
        address depositor = makeAddr("depositor");
        uint256 amount = 1000e6;

        vm.createSelectFork(vm.envString("MAINNET_RPC_URL"));
        
        address usdt = 0xdAC17F958D2ee523a2206206994597C13D831ec7;
        
        vm.prank(owner);
        vault = new AlchemistTokenVault(usdt, alchemist, owner);

        // Uniswap: USDT pool
        address usdtWhale = 0x5754284f345afc66a98fbB0a0Afe71e0F007B949;
        vm.prank(usdtWhale);
        IERC20(usdt).safeTransfer(depositor, amount); 
        assertEq(IERC20(usdt).balanceOf(depositor), amount);


        vm.startPrank(depositor);
        IERC20(usdt).forceApprove(address(vault), amount);
        vm.expectRevert();
        vault.deposit(amount);  // ❌ Call reverts
        vm.stopPrank();
        assertEq(IERC20(usdt).balanceOf(address(vault)), 0);
    }
```
