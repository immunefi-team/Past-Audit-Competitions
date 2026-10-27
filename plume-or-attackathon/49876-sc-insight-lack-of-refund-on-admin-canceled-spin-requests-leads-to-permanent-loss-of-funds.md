# 49876 sc insight lack of refund on admin canceled spin requests leads to permanent loss of funds

**Submitted on Jul 20th 2025 at 05:23:48 UTC by @vargalove for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49876
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

The Spin contract allows users to call startSpin() by sending ETH as a fee. If the designated oracle (SupraRouter) never invokes the expected callback (handleRandomness()), there is no user-facing recovery mechanism. An administrator can call cancelPendingSpin(), which clears pending state but does not refund the user's ETH — resulting in permanent loss of funds. The issue was validated using a mock router that implements the interface but does not call back handleRandomness().

### Vulnerability Details

The contract registers a pending spin when startSpin() is called and relies on an external oracle (SupraRouter) to complete the flow via handleRandomness(...). If the oracle does not callback (due to downtime, network failure, or malicious behavior), the contract exposes only an admin-driven emergency exit: cancelPendingSpin(). That function clears state but does not return the ETH to the user.

Vulnerable excerpts:

```solidity
function startSpin() external payable {
    require(msg.value == spinPrice, "Incorrect spin price sent");
    ...
    isSpinPending[msg.sender] = true;

    uint256 nonce = supraRouter.generateRequest(...);
    userNonce[nonce] = msg.sender;
    pendingNonce[msg.sender] = nonce;
}
```

```solidity
function cancelPendingSpin(address user) external onlyRole(ADMIN_ROLE) {
    require(isSpinPending[user], "No spin pending for this user");

    uint256 nonce = pendingNonce[user];
    if (nonce != 0) {
        delete userNonce[nonce];
    }

    delete pendingNonce[user];
    isSpinPending[user] = false;
    
    // There is no function that returns the ether to the user.
}
```

A minimal SupraRouterMock was implemented that returns a nonce but never calls handleRandomness(), simulating a realistic scenario where the oracle silently fails to callback.

### Impact Details

Direct loss of funds by the user without any fraudulent action. The loss occurs purely if the oracle fails to callback. There is no user-available method to recover the paid ETH once the admin cancels the pending spin.

### Mitigation

A suggested mitigation is to include a timeout/expiration mechanism allowing users to cancel and reclaim funds themselves after a given time:

```solidity
function cancelMyExpiredSpin() external {
    require(isSpinPending[msg.sender], "No pending spin");
    require(block.timestamp > spinRequestTimestamp[msg.sender] + 5 minutes, "Spin not expired");

    uint256 nonce = pendingNonce[msg.sender];
    if (nonce != 0) {
        delete userNonce[nonce];
    }

    delete pendingNonce[msg.sender];
    isSpinPending[msg.sender] = false;

    _safeTransferPlume(payable(msg.sender), spinPrice);
}
```

This protects users from stuck spins if the oracle becomes unresponsive.

{% hint style="warning" %}
Admin-driven cancellation should not be able to permanently confiscate user funds. Either ensure admin cancellation refunds users, or restrict admin cancellation to only administrative housekeeping combined with user refunds, or allow users to self-cancel after expiration.
{% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Overview and steps to reproduce

* A user calls startSpin() with msg.value = 2 ether.
* startSpin() calls supraRouter.generateRequest(...), which returns a nonce, but the oracle never calls back handleRandomness(...).
* An admin calls cancelPendingSpin(), which clears the pending state but does not refund the user — resulting in permanent loss of ETH.
  {% endstep %}

{% step %}

### Minimal SupraRouterMock

This mock fulfills the ISupraRouterContract interface but never invokes the callback:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import "../../src/interfaces/ISupraRouterContract.sol";

contract SupraRouterMock is ISupraRouterContract {
    function generateRequest(
        string memory,
        uint8,
        uint256,
        uint256,
        address
    ) external pure override returns (uint256) {
        return 1234;
    }

    function generateRequest(
        string memory,
        uint8,
        uint256,
        address
    ) external pure override returns (uint256) {
        return 1234;
    }

    function rngCallback(
        uint256,
        uint256[] memory,
        address,
        string memory
    ) external pure override returns (bool, bytes memory) {
        return (true, "");
    }
}
```

{% endstep %}

{% step %}

### Test simulation (Spin.t.sol)

The following Forge test demonstrates the scenario where the oracle fails to callback and the admin cancels the pending spin, leading to user fund loss:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import "forge-std/Test.sol";
import { console2 } from "forge-std/console2.sol";

import "../src/spin/Spin.sol";
import "./mocks/SupraRouterMock.sol";
import "./mocks/DateTimeMock.sol";

contract SpinMinimalTest is Test {
    Spin public spin;
    address public ADMIN = address(1);
    address public USER = address(2);
    uint256 public constant INITIAL_SPIN_PRICE = 2 ether;

    function setUp() public {
        vm.warp(1752974400);
        vm.startPrank(ADMIN);
        spin = new Spin();
        spin.initialize(address(new SupraRouterMock()), address(new DateTimeMock()));
        vm.deal(address(spin), 100 ether);
        spin.setEnableSpin(true);
        vm.stopPrank();
    }

    function testStuckSpinFundsLostAfterCancel() public {
        assertEq(DateTimeMock(address(spin.dateTime())).getDay(block.timestamp), 20);
        vm.warp(1753060800); // 21 de julio 2025 12:00 UTC

        vm.deal(USER, INITIAL_SPIN_PRICE);
        uint256 userInitialBalance = USER.balance;
        uint256 contractInitialBalance = address(spin).balance;

        vm.prank(USER);
        spin.startSpin{ value: INITIAL_SPIN_PRICE }();

        assertTrue(spin.isSpinPending(USER), "Spin is not left pending");

        vm.prank(ADMIN);
        spin.cancelPendingSpin(USER);

        assertFalse(spin.isSpinPending(USER), "Spin was not cancelled correctly");

        uint256 userFinalBalance = USER.balance;
        uint256 contractFinalBalance = address(spin).balance;

        assertEq(userInitialBalance - userFinalBalance, INITIAL_SPIN_PRICE);
        assertEq(contractFinalBalance - contractInitialBalance, INITIAL_SPIN_PRICE);

        console2.log("Stuck spin canceled: user lost funds without reward");
        console2.log("Initial user balance:", userInitialBalance);
        console2.log("Final user balance:", userFinalBalance);
        console2.log("Final contract balance:", contractFinalBalance);
    }
}
```

{% endstep %}

{% step %}

### Observed test logs

In the author's run, the test passed and logs showed:

* Stuck spin canceled: user lost funds without reward
* Initial user balance: 2000000000000000000
* Final user balance: 0
* Final contract balance: 102000000000000000000

This demonstrates the user-paid ETH (2 ether) remained in the contract after admin cancellation and the user balance decreased, confirming permanent loss.
{% endstep %}
{% endstepper %}

## Recommendation summary

* Add a user-timeout cancellation that refunds the user after a reasonable expiration.
* Ensure cancelPendingSpin (admin) refunds the user as part of cancellation, or restrict admin cancellation so it cannot permanently siphon user funds.
* Record timestamps when spin requests are created to enable time-based expirations and checks.

If you want, I can propose a specific code patch (diff) to add timestamp tracking and a safe refund path for expired spins.
