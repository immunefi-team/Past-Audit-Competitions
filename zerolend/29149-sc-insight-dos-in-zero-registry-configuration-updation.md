# 29149 - \[SC - Insight] DoS in Zero Registry configuration updation

Submitted on Mar 8th 2024 at 17:44:51 UTC by @oxumarkhatab for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29149

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/zerolend/governance>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The zero registry will not update the configuration in it's storage due to incomplete implementation.

## Vulnerability Details

The ZeroRegistry contains many onlyOwner functions among which `setPoolConfiguratorImpl` is the one. It takes newPool configuration and supposed to update the implementation logic.

```
  function setPoolConfiguratorImpl(
        address newPoolConfiguratorImpl
    ) external override onlyOwner {
        address oldPoolConfiguratorImpl = _getProxyImplementation(
            POOL_CONFIGURATOR
        );
        _updateImpl(POOL_CONFIGURATOR, newPoolConfiguratorImpl);
        emit PoolConfiguratorUpdated(
            oldPoolConfiguratorImpl,
            newPoolConfiguratorImpl
        );
    }
```

However , due to incomplete implementation of `_getProxyImplementation` & `_updateImpl`,

`_getProxyImplementation will always result 0`

```
 function _getProxyImplementation(bytes32 id) internal returns (address) {
        // address proxyAddress = _addresses[id];
        // if (proxyAddress == address(0)) {
        //     return address(0);
        // } else {
        //     address payable payableProxyAddress = payable(proxyAddress);
        //     return
        //         InitializableImmutableAdminUpgradeabilityProxy(
        //             payableProxyAddress
        //         ).implementation();
        // }
        return address(0);
    }
```

and `_updateImpl` wil always pass and do no storage updates

```
    function _updateImpl(bytes32 id, address newAddress) internal {
        address proxyAddress = _addresses[id];
        // InitializableImmutableAdminUpgradeabilityProxy proxy;
        // bytes memory params = abi.encodeWithSignature(
        //     "initialize(address)",
        //     address(this)
        // );

        // if (proxyAddress == address(0)) {
        //     proxy = new InitializableImmutableAdminUpgradeabilityProxy(
        //         address(this)
        //     );
        //     _addresses[id] = proxyAddress = address(proxy);
        //     proxy.initialize(newAddress, params);
        //     emit ProxyCreated(id, proxyAddress, newAddress);
        // } else {
        //     proxy = InitializableImmutableAdminUpgradeabilityProxy(
        //         payable(proxyAddress)
        //     );
        //     proxy.upgradeToAndCall(newAddress, params);
        // }
    }
```

This function does not store or update anything, leading to denial of service for owner. ( technically it stores the new implementation address always at the zero proxy address which is logical flaw)

In an event where the configuration's implementation has to be updated and otherwise a lot of funds would be at risk, this function will fail to secure the protocol and cause un-intended behaviour causing DoS of updation of configurations.

## Impact Details

Protocol functioning damage - Updates through registry can not be done due to in-complete logic implementation.

## References

see PoC for details

## Proof of Concept

Although it's clear from the above report description , you can visualize the scenario from this PoC

```
function test_failedConfigUpdate()public{
   // suppose we have deployed contract stored in variable zeroRegistry_deployed

address newPoolConfigImpl=0x37A8d3c717ec8fDc8BD859627F18ce89c31E1E8b;

vm.startPrank(Owner);

zeroRegistry_deployed.setPoolConfiguratorImpl(newPoolConfigImpl);

assertEq(zeroRegistry._getProxyImplementation(),address(0));
vm.stopPrank(0)
}
```
