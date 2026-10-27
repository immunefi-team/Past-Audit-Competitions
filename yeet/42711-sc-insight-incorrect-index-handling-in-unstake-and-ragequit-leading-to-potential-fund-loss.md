# #42711 \[SC-Insight] Incorrect Index Handling in \`unstake\` and \`rageQuit\` Leading to Potential Fund Loss

**Submitted on Mar 25th 2025 at 13:13:44 UTC by @x60scs for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42711
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## **Brief/Intro**

In the **StakeV2** contract, when the `rageQuit` and `unstake` functions are used, the order of the user's `vestings` array changes. This can lead to calling functions with incorrect indexes, causing users to perform unintended `unstake` or `rageQuit` operations, which may result in fund loss.

***

## **Vulnerability Details**

This issue arises due to the way the `vestings` array is managed. When a user calls `startUnstake`, the request is stored at a specific index in the array. However, when `unstake(index)` or `rageQuit(index)` is executed, the array is **reordered**, causing indexes to shift. If the user or dApp performs an operation based on previous indices, or if the dApp does not update the page, they may unintentionally unstake the wrong position using `unstake(index)` or `rageQuit(index)`

### **Example Scenario**

1. The user initiates 5 different `startUnstake` requests:
   * Unstake array: `(a, b, c, d, e)`
   * Assigned indexes: `(0, 1, 2, 3, 4)`
2. The user calls `unstake(2)`, removing `c`. The array is updated as follows:
   * New array: `(a, b, d, e)`
   * New indexes: `(0, 1, 2, 3)`
   * **Important:** `d`'s previous index **3** is now **2**, and `e`'s previous index **4** is now **3**.
3. If the user or a dApp, based on outdated information, calls `rageQuit(3)`, expecting to exit `d`, they would actually exit `e` instead.

## **Impact Details**

This vulnerability can cause users to **exit the wrong vesting position**, leading to unintended `rageQuit` operations and **potential fund loss**.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L399-L408>

### **Recommendation**

A `bool isUnstake` variable can be added to the Vesting struct. Additionally, minimum stake and minimum unstake conditions can be introduced to protect against DOS attacks.

```diff
 /// @notice The struct used to store the vesting information
    struct Vesting {
        uint256 amount;
        uint256 start;
        uint256 end;
+       bool isUnstake;
    }
```

```diff
-    function _remove(address addr, uint256 _index) private {
-        Vesting[] storage arr = vestings[addr];
-        require(_index < arr.length, "index out of bound");
-        uint256 length = arr.length;

-        for (uint256 i = _index; i < length - 1; i++) {
-            arr[i] = arr[i + 1];
-        }
-        arr.pop();
-    }
```

```diff
     function unstake(uint256 index) external {
        require(block.timestamp >= vestings[msg.sender][index].end, "Vesting period has not ended");
+       require(!vestings[msg.sender][index].isUnstake, "Unstaked");
        _unstake(index);
    }
    
    function rageQuit(uint256 index) external {
+       require(!vestings[msg.sender][index].isUnstake, "Unstaked");
        _unstake(index);
    }
    
    function _unstake(uint256 index) private {
        Vesting memory vesting = vestings[msg.sender][index];

        (uint256 unlockedAmount, uint256 lockedAmount) = calculateVesting(vesting);
        require(unlockedAmount != 0, "No unlocked amount");

        stakingToken.transfer(msg.sender, unlockedAmount);
        stakingToken.transfer(address(0x000000dead), lockedAmount);
-       _remove(msg.sender, index);
+       vestings[msg.sender][index].isUnstake=true;
        if (lockedAmount > 0) {
            emit RageQuit(msg.sender, unlockedAmount, lockedAmount, index);
        } else {
            emit Unstake(msg.sender, unlockedAmount, index);
        }
        stakedTimes[msg.sender]--;
    }

```

## Proof of Concept

## Proof of Concept

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "forge-std/console2.sol";
import "../src/StakeV2.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {MockWETH} from "./mocks/MockWBERA.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./mocks/SimpleZapperMock.sol";

contract KodiakVaultV1 {
    IERC20 public token0;
    IERC20 public token1;

    constructor(IERC20 _token0, IERC20 _token1) {
        token0 = _token0;
        token1 = _token1;
    }
}

contract StakeV2_POC is Test {
    MockERC20 public token;
    MockWETH public wbera;

    error ERC20InsufficientBalance(
        address sender,
        uint256 balance,
        uint256 needed
    );

    function setUp() public virtual {
        token = new MockERC20("MockERC20", "MockERC20", 18);
        wbera = new MockWETH();
    }
   function test_poc2() public {
        address owner = makeAddr("owner");
        address manager = makeAddr("manager");
        address staker = makeAddr("staker");

        KodiakVaultV1 kodiakVault = new KodiakVaultV1(token, wbera);
        SimpleZapperMock mockZapper = new SimpleZapperMock(
            kodiakVault.token0(),
            kodiakVault.token1()
        );
        StakeV2 stakeV2 = new StakeV2(
            token,
            mockZapper,
            owner,
            manager,
            IWETH(wbera)
        );

        vm.startPrank(staker);
        token.mint(address(staker), 500 ether);
        token.approve(address(stakeV2), 500 ether);
        stakeV2.stake(500 ether);

        vm.expectEmit();
        emit StakeV2.VestingStarted(staker, 40 ether, 0);
        stakeV2.startUnstake(40 ether);

        vm.warp(block.timestamp + 1 days);

        vm.expectEmit();
        emit StakeV2.VestingStarted(staker, 50 ether, 1);
        stakeV2.startUnstake(50 ether);

        vm.warp(block.timestamp + 1 days);

        vm.expectEmit();
        emit StakeV2.VestingStarted(staker, 60 ether, 2);
        stakeV2.startUnstake(60 ether);

        vm.warp(block.timestamp + 1 days);

        vm.expectEmit();
        emit StakeV2.VestingStarted(staker, 70 ether, 3);
        stakeV2.startUnstake(70 ether);

        vm.warp(block.timestamp + 1 days);

        vm.expectEmit();
        emit StakeV2.VestingStarted(staker, 80 ether, 4);
        stakeV2.startUnstake(80 ether);

        assertEq(stakeV2.getVestings(staker)[3].amount, 70 ether);
        stakeV2.rageQuit(2);

        assertEq(stakeV2.getVestings(staker)[3].amount, 80 ether);

        vm.stopPrank();
    }
}

```
