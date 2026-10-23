# #41890 \[SC-Insight] MoneyBrinter vault does not consider Farm's staking cap

**Submitted on Mar 19th 2025 at 07:49:59 UTC by @trtrth for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41890
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**

## Description

## Brief/Intro

The MoneyBrinter vault does not take into account the maximum deposit amount. This makes the vault incompliant with ERC4626 and it can cause user deposits unexpectedly to fail

## Vulnerability Details

According to ERC-4626 about `maxDeposit()`

> Maximum amount of the underlying asset that can be deposited into the Vault for the receiver, through a deposit call.

> MUST return the maximum amount of assets deposit would allow to be deposited for receiver and not cause a revert, which MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary). This assumes that the user has infinite assets, i.e. MUST NOT rely on balanceOf of asset.

> MUST factor in both global and user-specific limits, like if deposits are entirely disabled (even temporarily) it MUST return 0.

> MUST return 2 \*\* 256 - 1 if there is no limit on the maximum amount of assets that may be deposited.

> MUST NOT revert.

The current MoneyBrinter implementation always returns `2 ** 256 - 1` for maxDeposit().

However, the farm contract has the staking cap configuration. By this, the value returned from `maxDeposit()` can be higher than the farm's remaining staking cap and it can cause user deposits to be failed.

Below is the staking logic in the farm contract

```solidity
    function _stakeLocked(
        address staker_address,
        address source_address,
        uint256 liquidity,
        uint256 secs,
        uint256 start_timestamp
    ) internal updateRewardAndBalance(staker_address, true) {
        require(!stakingPaused, "Staking paused");
        require(liquidity > 0, "Must stake more than zero");
@> staking cap here >>        require(_total_liquidity_locked.add(liquidity) <= stakingTokenCap, "Farm cap exceeded");
        require(greylist[staker_address] == false, "Address has been greylisted");
        require(secs >= lock_time_min, "Minimum stake time not met");
        require(secs <= lock_time_for_max_multiplier,"Trying to lock for too long");

        uint256 lock_multiplier = lockMultiplier(secs);
        bytes32 kek_id = keccak256(abi.encodePacked(staker_address, start_timestamp, liquidity, _locked_liquidity[staker_address]));
        lockedStakes[staker_address].push(LockedStake(
            kek_id,
            start_timestamp,
            liquidity,
            start_timestamp.add(secs),
            lock_multiplier
        ));

        // Pull the tokens from the source_address
        TransferHelper.safeTransferFrom(address(stakingToken), source_address, address(this), liquidity);

        // Update liquidities
        _total_liquidity_locked = _total_liquidity_locked.add(liquidity);
        _locked_liquidity[staker_address] = _locked_liquidity[staker_address].add(liquidity);

        // Need to call to update the combined weights
        _updateRewardAndBalance(staker_address, false);

        // Needed for edge case if the staker only claims once, and after the lock expired
        if (lastRewardClaimTime[staker_address] == 0) lastRewardClaimTime[staker_address] = block.timestamp;

        emit StakeLocked(staker_address, liquidity, secs, kek_id, source_address);
    }
```

## Impact Details

* Not comply ERC4626

## References

Farm contract implementation: <https://bartio.beratrail.io/token/0xbdEE3F788a5efDdA1FcFe6bfe7DbbDa5690179e6/contract/code>

## Proof of Concept

## Proof of Concept

* Update the test `test_Valid_Deposit_Into_Beradrome` in file `test/vault/Vault_IntegrationTest_ZeroFee.t.sol` as below:

```solidity
    function test_Valid_Deposit_Into_Beradrome() public {
        uint256 maxYeet = 80 * token;
        uint256 maxWBera = 1 * token;

        fundYeet(alice, maxYeet);
        fundWbera(alice, maxWBera);

        (, bytes memory ret) = kodiakFarm.staticcall(abi.encodeWithSignature("owner()"));
        address owner = abi.decode(ret, (address));

        vm.prank(owner);
        kodiakFarm.call(abi.encodeWithSignature("setStakingTokenCap(uint256)", 1 ether));

      
        (uint256 yeetUsed, uint256 wBeraUsed, uint256 islandsMinted) =
            depositIntoYeetIsland(alice, maxYeet, maxWBera);

        vm.startPrank(alice);
        IERC20(yeetIsland).approve(moneyBrinter, islandsMinted);

        uint max = IERC4626(moneyBrinter).maxDeposit(alice);

        console.log('max deposit: %s', max);

        vm.expectRevert('Farm cap exceeded');
        uint256 sharesMinted = IERC4626(moneyBrinter).deposit(islandsMinted, alice);
    }
```

* Run the test and console shows.

```bash
Ran 1 test for test/vault/Vault_IntegrationTest_ZeroFee.t.sol:Vault_IntegrationTest_ZeroFee
[PASS] test_Valid_Deposit_Into_Beradrome() (gas: 1110991)
Logs:
  max deposit: 115792089237316195423570985008687907853269984665640564039457584007913129639935
```

It means that the max deposit amount is still 2\*\*256 - 1 and the deposit fails due to farm's staking cap reached
