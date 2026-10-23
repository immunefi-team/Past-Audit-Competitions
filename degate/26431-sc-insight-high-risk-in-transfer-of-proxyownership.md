# 26431 - \[SC - Insight] High Risk in transfer of proxyOwnership

Submitted on Dec 2nd 2023 at 22:05:37 UTC by @SentientX for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26431

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x54D7aE423Edb07282645e740C046B9373970a168#code>

Impacts:

* Permanent freezing of funds from the Default Deposit Contract that requires malicious actions from the DeGate Operator.
* Theft of funds from the Default Deposit Contract that requires malicious actions from the DeGate Operator.

## Description

## Bug Description

The function:

```
  function transferProxyOwnership(address newOwner) public onlyProxyOwner {
    require(newOwner != address(0));
    emit ProxyOwnershipTransferred(proxyOwner(), newOwner);
    setUpgradabilityOwner(newOwner);
  }
```

handles the transfer of ownership to new proxy owner, however the function fails to recognize two fundamental risks:

1. Transfer of ownership to compromised current owner address
2. Transfer of ownership to invalid address.

both of which can be caused by operational error or malicious actions of Degate Operator.

Risk 1 Scenario:

1. Current owner address is compromised. (eg. private key leaked)
2. In state of panic ownership is transferred to same owner
3. Risk exposure has not changed

Risk 2 Scenario

1. Current owner address is compromised
2. In state of Panic transfer is made to in valid address

## Impact

1. Funds can be stolen if malicious Degate Operator transfers proxy to an address he controls
2. Funds can be permanently locked up if an error in transfer is made to in valid address

## Risk Breakdown

Difficulty to Exploit: Easy Weakness: CVSS2 Score:

## Recommendation

1. Consider use of 2-step transferOwnersip Proxy
2. Ensure proxy address is not the same address as current owner
3. transferProxyOwnership by changing pending owner to newOwner

example:

```
   * @dev Allows the current owner to transfer control of the contract to a newOwner.
   *changes the pending owner to newOwner. But doesn't actually transfer
   * @param newOwner The address to transfer ownership to.
   */
   function transferProxyOwnership(address newOwner) external onlyProxyOwner {
       require(newOwner != address(0));
       _setPendingUpgradeabilityOwner(newOwner);
       emit NewPendingOwner(proxyOwner(), newOwner);
   }
```

2. Let pending Owner claim ownership:

```
    /**
    * @dev Allows the pendingOwner to claim ownership of the proxy
    */
    function claimProxyOwnership() external onlyPendingProxyOwner {
        emit ProxyOwnershipTransferred(proxyOwner(), pendingProxyOwner());
        _setUpgradeabilityOwner(pendingProxyOwner());
        _setPendingUpgradeabilityOwner(address(0));
    }
```

Finally internal function,

```
    function _setUpgradeabilityOwner(address newProxyOwner) internal {
        bytes32 position = proxyOwnerPosition;
        assembly {
            sstore(position, newProxyOwner)
        }
    }
```

## References

Reference implementation:

<https://www.codeslaw.app/contracts/ethereum/0x0000852600ceb001e08e00bc008be620d60031f2?file=TrueHKD.sol\\&start=98>

## Proof of concept

1. Current Proxy owner calls `transferProxyOwnership(address newOwner) public onlyProxyOwner` setting new owner to compromised address accidently or maliciously.
2. Protocol Default Deposit contract is compromised
3. Funds Stolen or Funds Frozen
