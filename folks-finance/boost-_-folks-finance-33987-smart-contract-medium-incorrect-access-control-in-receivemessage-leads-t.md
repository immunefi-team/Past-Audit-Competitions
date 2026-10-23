# Boost \_ Folks Finance 33987 - \[Smart Contract - Medium] Incorrect access control in receiveMessage l

Submitted on Sat Aug 03 2024 11:26:33 GMT-0400 (Atlantic Standard Time) by @QuantumKid for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33987

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Due to incorrect adapter access control check in sendMessage function on BridgeRouter contract an attacker can use a malicious contract as adapter to drain the entire funds in the protocol.

## Vulnerability Details

In the receiveMessage to check that whether msg.sender is valid adapter or not first the adapterId is being read from the `adapterToId` mapping as below

```solidity
IBridgeAdapter adapter = IBridgeAdapter(msg.sender);
uint16 adapterId = adapterToId[adapter];
```

Then using that adapterId we check whether that adapterId has a non-zero adapter address in `idToAdapter` mapping as using below function.

```solidity
    function isAdapterInitialized(uint16 adapterId) public view returns (bool) {
        IBridgeAdapter adapter = idToAdapter[adapterId];
        return (address(adapter) != address(0x0));
    }
```

But while adding adapter using below function if zero is used as a valid adapterId any address will be considered as a valid adapter.

```solidity
    function addAdapter(uint16 adapterId, IBridgeAdapter adapter) external onlyRole(MANAGER_ROLE) {
        // check if no existing adapter
        if (isAdapterInitialized(adapterId)) revert AdapterInitialized(adapterId);

        // add adapter
        idToAdapter[adapterId] = adapter;
        adapterToId[adapter] = adapterId;
    }
```

For Example: Let's say 0x1234 is a valid adapter stored at adapterId zero. Then\
`idToAdapter[0] = 0x1234`\
`adapterToId[0x1234] = 0`

Now a malicious contract `0xdead` will also be considered as valid adapter because `adapterToId[0xdead] = 0` and `isAdapterInitialized(0)` will return true as a valid adapter is already initialized with zero as adapterId.

## Impact Details

As any arbitrary contract is considered as a valid adapter. An attacker can use a malicious adapter contract to pass fake messages to drain the entire protocol.

## References

From tests it seems like zero is a valid adapterId

```javascript
  async function addAdapterFixture() {
    const { admin, messager, unusedUsers, bridgeRouter, bridgeRouterAddress } =
      await loadFixture(deployBridgeRouterFixture);

    // deploy and add adapter
    const adapter = await new MockAdapter__factory(admin).deploy(bridgeRouterAddress);
@>  const adapterId = 0;
    const adapterAddress = await adapter.getAddress();
    await bridgeRouter.connect(admin).addAdapter(adapterId, adapterAddress);
  }
```

## Proof of concept

## Proof of Concept

Add this to `add adapter` tests in `/test/bridge/BridgeRouter.test.ts` file.

```javascript
    it("Arbitrary contract will be considered as a valid adapter.", async () => {
      const { bridgeRouter, adapterId, adapterAddress } = await loadFixture(addAdapterFixture);

      // verify adapter was added
      expect(await bridgeRouter.isAdapterInitialized(adapterId)).to.be.true;
      expect(await bridgeRouter.idToAdapter(adapterId)).to.equal(adapterAddress);
      expect(await bridgeRouter.adapterToId(adapterAddress)).to.equal(adapterId);
      // Random address is considered as a valid adapter.
      expect(await bridgeRouter.isAdapterInitialized(await bridgeRouter.adapterToId(getRandomAddress()))).to.be.true;
    });
```
