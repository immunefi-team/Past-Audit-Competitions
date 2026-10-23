# 57849 sc high funds gets stuck even when killswitch is enabled

**Submitted on Oct 29th 2025 at 08:03:41 UTC by @shadowHunter for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57849
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

Even if `killSwitch` is enabled by Owner, funds gets transferred from Vault to Adaptor in case of allocation. These funds remain stuck with no way to deallocate until `killSwitch` is turned off

## Vulnerability Details

* If `killSwitch` is true on MYTStrategy and allocation is done then `MYTStrategy` correctly make 0 changes

```
if (killSwitch) {
            return (ids(), int256(0));
        }
```

* But still Morpho vault which ignores if any changes were made to id, will send funds from vault to the MYTStrategy adaptor. This should be unexpected

```
https://github.com/morpho-org/vault-v2/blob/406546763343b9ffa84c2f63742ae55a490b7c42/src/VaultV2.sol#L576
function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

        accrueInterest();

        SafeERC20Lib.safeTransfer(asset, adapter, assets);
        (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);
```

* Also now you cannot deallocate this stuck funds until owner turns kill switch off which may not even happen (since adaptor does not approve funds to vault due to early exit)

```
 function deallocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
        external
        onlyVault
        returns (bytes32[] memory strategyIds, int256 change)
    {
        if (killSwitch) {
            return (ids(), int256(0));
        }
```

## Impact Details

Funds get stuck in adaptor

## Recommendation

Allocator should not call vault if kill switch is enabled

## Proof of Concept

## Proof of Concept

Place this in `MYTStrategy.t.sol` test suit

```
function test_killSwitchPreventsAllocation() public {
        //  Enable kill switch
         vm.prank(admin);
         strategy.setKillSwitch(true);

        //  Vault should fail to allocate
         vm.prank(address(vault));
         vm.expectRevert(bytes("emergency"));
         strategy.allocate(abi.encode(0), 100e18, bytes4(0x00000000), address(allocator));
     }
```

Ouput:

```
Encountered 1 failing test in src/test/MYTStrategy.t.sol:MYTStrategyTest
[FAIL: next call did not revert as expected] test_killSwitchPreventsAllocation() (gas: 50021)

```
