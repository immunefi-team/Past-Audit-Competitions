# 25921 - \[SC - Insight] Flaw in upgradeToAndCall leads to the proxy cal...

Submitted on Nov 21st 2023 at 03:11:43 UTC by @neth for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25921

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x54D7aE423Edb07282645e740C046B9373970a168#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

In [OwnedUpgradeabilityProxy, function upgradeToAndCall](https://github.com/degatedev/protocols/blob/c8961f2cd354a6578bb332337f983ab4c39c1806/packages/loopring_v3/contracts/thirdparty/proxies/OwnedUpgradabilityProxy.sol#L87)

```solidity
  function upgradeToAndCall(address implementation, bytes memory data) payable public onlyProxyOwner {
    upgradeTo(implementation);
    (bool success, ) = address(this).call{value: msg.value}(data);
    require(success);
  }
```

calls himself by doing `address(this).call...`, which redirects to the `fallback` function in `Proxy` with `msg.sender == address(this)`, instead of directly delegatecalling to the implementation with the caller being Timelock, which is bad as proxies are not supposed to "initiate" transactions, but delegate them.

## Impact

In the implementation, the owner is responsible for calling `setCheckBalance`, which is, at the moment, an EOA (see <https://etherscan.io/address/0xacD3A62F3eED1BfE4fF0eC8240d645c1F5477F82>) but it seems it will be Timelock in the future. That means it is NOT, by any means, the proxy, so trying to update the implementation via `upgradeToAndCall` will revert, as the modifier will check for `msg.sender == owner` with `owner != address(proxy)`.

The same applies to [ExchangeProxy](https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B?utm_source=immunefi#code)

## Recommendation

```diff
  function upgradeToAndCall(address implementation, bytes memory data) payable public onlyProxyOwner {
    upgradeTo(implementation);
-   (bool success, ) = address(this).call{value: msg.value}(data);
+   _fallback();
    require(success);
  }
```

## Proof of concept

(Simplified POC, if you need the mainnet fork I can give it to you but it was far more verbose)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console2} from "forge-std/Test.sol";

contract Proxy {
    fallback() payable external {
        console2.log("Caller in Proxy::fallback, which is gonna be passed to impl (shall be Timelock)");
        console2.log(msg.sender);
        console2.log("");
        console2.log("It was the address of the Proxy, see the next log which outputs address(this)");
        console2.log(address(this));
    }

    function updateToAndCall() external {
        console2.log("Caller in Proxy::updateToAndCall, that is, Timelock");
        console2.log(msg.sender);
        console2.log("");
        address(this).call("");
    } 
}

contract POC is Test {

    Proxy internal proxy;

    function setUp() external {
        proxy = new Proxy();
    }

    function testPOC() external {
        console2.log("Address of the initial caller, that is, Timelock");
        console2.log(address(this));
        console2.log("");
        proxy.updateToAndCall();
    } 
}
```
