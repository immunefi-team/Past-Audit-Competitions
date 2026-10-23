# 49893 sc insight raffle sol implementation logic allows direct plume transfers but has no withdraw locking funds permanently

* Submitted on Jul 20th 2025 at 08:34:29 UTC by @blackgrease for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #49893
* Report Type: Smart Contract
* Severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* Impact: Permanent freezing of funds

## Summary

The implementation of the `Raffle` contract can receive native PLUME via its `receive` function, but it lacks any functionality to withdraw those funds, resulting in permanently locked tokens.

The `RaffleProxy` correctly blocks direct transfers (reverting with `ETHTransferUnsupported()`), but the implementation contract does not, so any PLUME sent directly to the implementation will be unretrievable.

## Details

Raffle implementation snippet:

```solidity
    //---snip----
    // UUPS Authorization
    function _authorizeUpgrade(
        address newImplementation
    ) internal override onlyRole(ADMIN_ROLE) { }

    // Allow contract to receive ETH
    receive() external payable {
     }

}
```

Documentation states that the Raffle system is not expected to handle funds because rewards are handled off-chain:

"Claiming a Prize(Multi-Winner aware): ... The actual delivery of the prize is handled off-chain."\
Source: <https://github.com/plumenetwork/contracts/blob/main/plume/SPIN.md#2-the-raffle-contract-raffle.sol>

Because the implementation contract can accept native PLUME and has no withdrawal or forwarding logic, any PLUME transferred directly to the implementation will be permanently locked.

## Impact

Permanent locking of funds: any PLUME sent to the implementation contract will be inaccessible, resulting in protocol-held value that cannot be used for operations or recovery.

## Mitigation

{% stepper %}
{% step %}

### Revert direct transfers

Modify the `receive` function to revert on direct transfers, for example:

```diff
    receive() external payable {
+        revert("Direct Plume Transfer");
    }
```

{% endstep %}

{% step %}

### Remove the `receive` function

Alternatively, remove the `receive` function entirely so the implementation cannot accept native PLUME.
{% endstep %}
{% endstepper %}

## Proof of Concept

A runnable PoC demonstrates sending native PLUME to the implementation contract and shows that funds become locked with no withdrawal path.

Gist (PoC): <https://gist.github.com/blackgrease/75dcdaf92e9d5482e3e4c4abc30c3d82>

Run with:

```
forge test --mt testCanSendPlumeButNoWithdraw --via-ir -vvv
```

Walk-through:

* Sending PLUME to the `RaffleProxy` reverts as intended.
* Sending PLUME to the `Raffle` implementation succeeds (because of the `receive` function).
* The implementation has no withdrawal functionality; funds remain locked.

<details>

<summary>Additional PoC notes &#x26; artifacts</summary>

* Private Gist Link: <https://gist.github.com/blackgrease/75dcdaf92e9d5482e3e4c4abc30c3d82>
* The original report included a Foundry test and a screenshot of the stack trace from the test run.

</details>
