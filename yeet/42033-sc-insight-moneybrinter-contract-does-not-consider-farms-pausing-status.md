# #42033 \[SC-Insight] MoneyBrinter contract does not consider farm's pausing status

**Submitted on Mar 20th 2025 at 07:26:30 UTC by @trtrth for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42033
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**

## Description

## Brief/Intro

The contract `MoneyBrinter` does not consider Kodiak farm's pausing status for staking and withdrawals. This can cause `MoneyBrinter` to be incompliant with ERC-4626

## Vulnerability Details

Both function `depositFor()` and `withdrawTo()` from Beradrome farm plugin contract handle withdraw locked all from Kodiak farm and continue to stake all into the farm. Here `depositFor()` is called by `MoneyBrinter::deposit()` and `MoneyBrinter::mint(). The function` withdrawTo()`is called by`MoneyBrinter::withdraw()`and`MoneyBrinter::redeem()`. So all the core functions of` MoneyBrinter\` contract go through the steps Farm::withdrawLockedAll() -> Farm::stakeLocked().

```solidity
    function depositFor(address account, uint256 amount) 
        public 
        override 
    {
        super.depositFor(account, amount);
        ICommunalFarm(farm).withdrawLockedAll();
        uint256 balance = IERC20(getUnderlyingAddress()).balanceOf(address(this));
        IERC20(getUnderlyingAddress()).safeApprove(farm, 0);
        IERC20(getUnderlyingAddress()).safeApprove(farm, balance);
        ICommunalFarm(farm).stakeLocked(balance, 0);
    }

    function withdrawTo(address account, uint256 amount) 
        public 
        override 
    {
        ICommunalFarm(farm).withdrawLockedAll(); 
        super.withdrawTo(account, amount);
        uint256 balance = IERC20(getUnderlyingAddress()).balanceOf(address(this));
        IERC20(getUnderlyingAddress()).safeApprove(farm, 0);
        IERC20(getUnderlyingAddress()).safeApprove(farm, balance);
        ICommunalFarm(farm).stakeLocked(balance, 0);
    }
```

On the other side, the farm contract has pausing feature for the staking and withdrawals. So in case only one of these two is paused, then the core functions of `MoneyBrinter` contract does not work and all funds can be temporarily locked in farm contract

```solidity
    function toggleStaking() external onlyOwner {
        stakingPaused = !stakingPaused;
    }

    function toggleWithdrawals() external onlyOwner {
        withdrawalsPaused = !withdrawalsPaused;
    }
```

## Impact Details

* ERC-4626 incompliant
* Contract's core functions can be unable to operate

## References

Farm contract: <https://bartio.beratrail.io/token/0xbdEE3F788a5efDdA1FcFe6bfe7DbbDa5690179e6/contract/code>

Beradrome plugin contract: <https://bartio.beratrail.io/token/0x80D7759Fa55f6a1F661D5FCBB3bC5164Dc63eb4D/contract/code>

## Proof of Concept

## Proof of Concept

* Modify the test `test_Valid_Deposit_Into_Beradrome` in test file `test/vault/Vault_IntegrationTest_ZeroFee.t.sol`

```solidity
    function test_Valid_Deposit_Into_Beradrome() public {
        uint256 maxYeet = 80 * token;
        uint256 maxWBera = 1 * token;

        fundYeet(alice, maxYeet);
        fundWbera(alice, maxWBera);

        (, bytes memory ret) = kodiakFarm.staticcall(abi.encodeWithSignature("owner()"));
        address owner = abi.decode(ret, (address));

        // pause withdrawals
        vm.prank(owner);
        kodiakFarm.call(abi.encodeWithSignature("toggleWithdrawals()"));
        
        (, , uint256 islandsMinted) =
            depositIntoYeetIsland(alice, maxYeet, maxWBera);


        uint maxDeposit = IERC4626(moneyBrinter).maxDeposit(alice);
        console.log('max deposit: %s', maxDeposit);

        vm.startPrank(alice);
        IERC20(yeetIsland).approve(moneyBrinter, islandsMinted);

        vm.expectRevert('Withdrawals paused');
        IERC4626(moneyBrinter).deposit(islandsMinted, alice);
    }
```

* Run the test by command `forge t --mt test_Valid_Deposit_Into_Beradrome -vv` and the console shows

```bash
Ran 1 test for test/vault/Vault_IntegrationTest_ZeroFee.t.sol:Vault_IntegrationTest_ZeroFee
[PASS] test_Valid_Deposit_Into_Beradrome() (gas: 810185)
Logs:
  max deposit: 115792089237316195423570985008687907853269984665640564039457584007913129639935
```

It means that the `maxDeposit()` returns wrong value, and it is also impossible to deposit into the vault even when Kodiak farm only pauses withdrawals
