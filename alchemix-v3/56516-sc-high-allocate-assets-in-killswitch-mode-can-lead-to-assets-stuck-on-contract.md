# 56516 sc high allocate assets in killswitch mode can lead to assets stuck on contract

**Submitted on Oct 17th 2025 at 07:24:05 UTC by @ox9527 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56516
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The MYTStrategy.sol::allocate() function is invoked by two functions within the Morpho V2 vault:

* VaultV2.sol::mint() -> enter() -> allocateInternal() -> allocate()
* VaultV2.sol::allocate() -> allocateInternal() -> allocate()

Note that mint() is a public function, while allocate() can only be called by users with the Allocator role.

In MYTStrategy.sol::allocate(), there is a kill switch / emergency mode. When the kill switch is active (killSwitch == true), the function immediately returns without performing any allocation logic:

```solidity
    function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        if (killSwitch) {
            return (ids(), int256(0));  <@
        }
```

However, in both mint() and allocate() flows, before calling allocate(), the vault already transfers assets to the strategy via allocateInternal:

```solidity
    function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

        accrueInterest();

        SafeERC20Lib.safeTransfer(asset, adapter, assets);
        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);
```

As a result, when the kill switch is active, the assets are transferred from the vault to MYTStrategy.sol, but since allocate() returns (ids(), 0), the assets are never utilized or recorded, leading to them becoming stuck in the MYTStrategy.sol contract.

## Vulnerability Details

Above

## Impact Details

assests stuck in contract when allocate() is called by public users/Allocator

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/MYTStrategy.sol?utm_source=immunefi#L107-L109>

## Proof of Concept

## Proof of Concept

add test to file AlchemistAllocator.t.sol

```solidity
    function test_POC1() public {
        address alice = address(0x1001);

        _magicDepositToVault(address(vault), alice, 150 ether);

        vm.startPrank(admin);
        bytes32 allocationId = mytStrategy.adapterId();
        allocator.allocate(address(mytStrategy), 100 ether);

        //set Emergency.
        mytStrategy.setKillSwitch(true);

        //allocare another 50 ether
        allocator.allocate(address(mytStrategy), 50 ether);
        vm.stopPrank();

        //check cap.
        uint256 allocation = vault.allocation(allocationId);

        assertEq(allocation, 100 ether);
    }

```
