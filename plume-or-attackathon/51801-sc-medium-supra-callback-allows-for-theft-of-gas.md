# 51801 sc medium supra callback allows for theft of gas

**Submitted on Aug 5th 2025 at 21:27:28 UTC by @arnie for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51801
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:** Theft of gas

## Description

### Brief / Intro

The Spin contract integrates with Supra dVRF (a decentralized VRF service). The protocol must deposit native tokens with Supra to cover the gas fees for the dVRF callback. The callback in the Spin contract performs a call to the user's address which may be a contract — this can be used to steal gas.

### Vulnerability Details

The Spin contract lets users spin for a random reward; randomness is provided via Supra dVRF. Supra requires the protocol to send funds to cover the callback gas (see their docs: <https://docs.supra.com/dvrf/v2-guide>).

In the callback, when the user wins certain reward categories, the contract transfers native funds using call:

```solidity
        if (
            keccak256(bytes(rewardCategory)) == keccak256("Jackpot")
                || keccak256(bytes(rewardCategory)) == keccak256("Plume Token")
        ) {
            _safeTransferPlume(user, rewardAmount * 1 ether);
        }
```

\_safeTransferPlume uses an unbounded call:

```solidity
    function _safeTransferPlume(address payable _to, uint256 _amount) internal {
        require(address(this).balance >= _amount, "insufficient Plume in the Spin contract");
        (bool success,) = _to.call{ value: _amount }("");
        require(success, "Plume transfer failed");
    }
```

There is no gas limit imposed on the call, so a recipient contract can consume excessive gas (e.g., by executing expensive loops or other operations). The gas for the callback is paid from the protocol's Supra top-up. Thus a malicious user can cause the protocol to spend more of the deposited native funds than intended — effectively "stealing" gas.

## Impact Details

Malicious user contracts can cause the Supra-funded callback to execute expensive operations during the transfer call, draining the protocol's deposited native funds for callbacks faster than expected.

## Proof of Concept

{% stepper %}
{% step %}

1. Protocol tops up their Supra account with native token.
   {% endstep %}

{% step %}
2\. Attacker deploys a contract and calls spin, which triggers a randomness request from Supra.
{% endstep %}

{% step %}
3\. Supra calls back the Spin contract at `handleRandomness`.
{% endstep %}

{% step %}
4\. If the attacker is chosen as winner for a category that triggers `_safeTransferPlume`, the Spin contract calls the attacker's contract with a value transfer via `call`.
{% endstep %}

{% step %}
5\. The attacker contract runs expensive operations in its fallback/receive or in the execution path triggered by the call, consuming excessive gas paid by the protocol's Supra top-up.
{% endstep %}

{% step %}
6\. The protocol's deposited native funds are spent faster, forcing the protocol to top up Supra sooner and suffering financial loss due to gas abuse.
{% endstep %}
{% endstepper %}

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L207>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L261>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L579>
* <https://docs.supra.com/dvrf/v2-guide>
