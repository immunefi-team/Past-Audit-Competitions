# 50040 sc low missing pause controls eth refund flaws and miscalculated shares enable fund loss and protocol inconsistency in depositandbridge

* **Submitted on:** Jul 21st 2025 at 09:11:24 UTC by @Sharky for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #50040
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupportPredicateProxy.sol>

## Impacts

* Permanent freezing of funds

## Description

### Brief/Intro

The TellerWithMultiAssetSupportPredicateProxy contract contains vulnerabilities that compromise fund safety, emergency response capabilities, and protocol integrity. If exploited, these issues can lead to permanent loss of user/contract funds, broken emergency shutdowns, and inconsistent protocol accounting.

### Vulnerability Details

#### Broken Pause Mechanism (Critical)

1. Issue: The contract inherits `Pausable` but lacks `pause()`/`unpause()` functions.
2. Technical Insight:

* The `paused()` checks in `deposit`/`depositAndBridge` are functional, but without external `pause/unpause` calls (enforceable only by `owner`), the contract can never be paused.
* Inheriting `Pausable` alone is insufficient; the OpenZeppelin design requires explicit pause control functions.

3. Code Proof:

```solidity
contract TellerWithMultiAssetSupportPredicateProxy is ... Pausable { 
    // Missing: pause()/unpause() functions
    function deposit(...) {
        if (paused()) revert ...; // Always false
    }
}
```

#### ETH Trapping Vulnerability (Critical)

1. Issue: ETH refunds fail after `depositAndBridge`, causing permanent fund loss.
2. Technical Insight:

* `lastSender` is reset to `address(0)` after `teller.depositAndBridge` completes.
* If the teller (or bridge) refunds excess ETH after this reset (e.g., for overpaid fees), the `receive()` function attempts to forward ETH to `address(0)`, which reverts and traps funds.

3. Code Proof:

```solidity
function depositAndBridge(...) payable {
    lastSender = msg.sender; // Set temporarily
    teller.depositAndBridge{value: msg.value}(...); // Refunds might occur AFTER this
    lastSender = address(0); // Reset too early
}
receive() external payable {
    (bool success,) = lastSender.call{value: msg.value}(""); // Fails if lastSender=0
}
```

#### Share Calculation Mismatch (Major)

1. Issue: `depositAndBridge` miscalculates shares in `Deposit` events, breaking fee-aware accounting.
2. Technical Insight:

* `deposit` uses actual shares minted (`shares = teller.deposit(...)`), which deducts fees.
* `depositAndBridge` uses a raw quote (`shares = depositAmount.mulDivDown(...)`) that ignores fees.
* This creates inconsistent event data, misleading off-chain systems about user balances.

3. Code Proof:

```solidity
// deposit (correct):
shares = teller.deposit(...); // Fee-aware
emit Deposit(..., shares); // Accurate

// depositAndBridge (incorrect):
teller.depositAndBridge(...); // Shares minted internally (with fees)
uint256 shares = depositAmount.mulDivDown(...); // Fee-ignorant calculation
emit Deposit(..., shares); // Inaccurate
```

## Impact Details

1. Fund Loss Scenarios:

* Stuck ETH: Users overpaying bridge fees lose refunds permanently (trapped in contract).
* Token Lockup: No rescue mechanism for accidentally sent ERC20 tokens.

2. Operational Risks:

* No Emergency Pause: Attackers exploit vulnerabilities unhindered during crises.
* Accounting Corruption: Fee discrepancies in events break integrations (e.g., tax/reporting systems).

3. Quantifiable Losses:

* Direct: 100% of overpaid ETH fees + any non-standard tokens sent to contract.
* Indirect: Protocol insolvency risk due to inconsistent state and loss of user trust.

## References

* Contract Code: Provided in report
* OpenZeppelin Pausable: <https://docs.openzeppelin.com/contracts/5.x/api/utils#Pausable>
* Solmate SafeTransferLib: <https://github.com/transmissions11/solmate/blob/main/src/utils/SafeTransferLib.sol>

## Proof of Concept

{% stepper %}
{% step %}

### Broken Pause Mechanism — Steps to Trigger

1. An attacker discovers a vulnerability in the deposit logic that allows fund theft.
2. Users continue depositing funds through `deposit`/`depositAndBridge` since:
   * `paused()` always returns `false` (unpaused state)
   * Owner cannot activate pause as `pause()` function doesn't exist
3. Attacker exploits vulnerability and drains user funds.

Result:

* All deposits are vulnerable during the attack window with no emergency mitigation.
  {% endstep %}

{% step %}

### ETH Trapping Vulnerability — Steps to Trigger

1. User calls `depositAndBridge` with excess ETH (example: 1 ETH for 0.9 ETH bridge fee):

```solidity
depositAndBridge{value: 1 ether}(...)
```

2. During execution:

* `lastSender = msg.sender` (set to user address)
* `teller.depositAndBridge` consumes 0.9 ETH
* `lastSender = address(0)` (reset after call)

3. Teller contract refunds 0.1 ETH to proxy:

```solidity
payable(proxy).transfer(0.1 ether);
```

4. Proxy's `receive()` triggers and attempts:

```solidity
(bool success,) = address(0).call{value: 0.1 ether}("");
```

5. Call to `address(0)` reverts → 0.1 ETH permanently locked in proxy.

Result:

* 100% of overpaid ETH is irrecoverable.
  {% endstep %}

{% step %}

### Share Calculation Mismatch — Steps to Trigger

1. Teller applies a 1% deposit fee in `depositAndBridge`.
2. User deposits 100 USDC (price: 1 USDC = 1 share):
   * Actual shares minted: 99 (after 1% fee)
3. Proxy calculates shares incorrectly:

```solidity
shares = 100 USDC * (10**18 / 1e6) / 1e18 = 100 shares
```

4. Proxy emits event with 100 shares (vs actual 99 shares).

Result:

* Off-chain monitors see inconsistent share balances (100 vs 99).
* Protocol appears to steal 1 share from user in analytics.
  {% endstep %}

{% step %}

### Unsafe Allowance Handling — Steps to Trigger

1. User deposits 100 USDC via `deposit`:

```solidity
depositAsset.safeApprove(vault, 100 USDC);
```

2. Later, same user deposits 50 USDC:

* Proxy tries: `safeApprove(vault, 50 USDC)`
* Reverts because existing allowance (100 USDC) ≠ 0.

Result:

* Subsequent deposits fail for any token with a prior non-zero approval.
  {% endstep %}

{% step %}

### Missing Rescue Functions — Steps to Trigger

1. User accidentally transfers 1000 USDC to proxy:

```solidity
USDC.transfer(proxyAddress, 1000e6);
```

2. Proxy has no `rescueTokens` function.
3. Owner cannot recover funds → 1000 USDC permanently locked.

Result:

* Any non-standard asset transfer causes permanent loss.
  {% endstep %}
  {% endstepper %}

## Summary of Exploit Paths

| Vulnerability  | Trigger Action                 | Consequence               |
| -------------- | ------------------------------ | ------------------------- |
| Broken Pause   | Exploit during attack          | No emergency shutdown     |
| ETH Trapping   | Overpay bridge fee             | ETH permanently stuck     |
| Share Mismatch | Use `depositAndBridge`         | Incorrect event emissions |
| Allowance Bug  | Second deposit with same asset | Deposit reverts           |
| No Rescue      | Accidental token transfer      | Funds permanently locked  |
