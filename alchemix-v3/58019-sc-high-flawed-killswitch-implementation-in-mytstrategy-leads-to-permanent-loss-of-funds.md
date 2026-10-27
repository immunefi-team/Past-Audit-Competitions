# 58019 sc high flawed killswitch implementation in mytstrategy leads to permanent loss of funds

**Submitted on Oct 30th 2025 at 02:24:05 UTC by @fullstop for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58019
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

A critical logic flaw exists in the killSwitch implementation within the MYTStrategy.sol contract. Instead of reverting transactions when activated, the allocate and deallocate functions silently succeed and return a change of 0. When the managing VaultV2 calls allocate, it first transfers assets to the strategy and then calls the strategy's allocate function for accounting. This flaw creates a fatal state mismatch: VaultV2 physically transfers the funds, but its internal accounting ledger (`_caps.allocation`) is never updated. These funds become permanently trapped in the MYTStrategy contract, irrecoverable by either the VaultV2 (which believes its allocation is zero) or the strategy's owner (who has no sweep function).

## Vulnerability Details

The killSwitch is intended to be a safety mechanism to pause all strategy interactions. However, its implementation in allocate and deallocate achieves the opposite, creating a fund-destroying black hole.

The exploit flow is as follows:

1. An administrator, believing the protocol is in danger, calls `setKillSwitch(true)` on a MYTStrategy contract.
2. An allocator (e.g., AlchemistAllocator) calls `VaultV2.allocate(strategy_address, data, amount)` to deposit funds.
3. VaultV2's allocateInternal function executes.
4. The vault first physically transfers the assets: `SafeERC20Lib.safeTransfer(asset, adapter, assets);`. At this moment, amount of the asset is now owned by the MYTStrategy contract.
5. VaultV2 then calls the strategy for accounting: `IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);`.
6. This calls `MYTStrategy.allocate`, which immediately checks the killSwitch:

```
        if (killSwitch) {
            return (ids(), int256(0));
        }
```

7. The Core Flaw: The function does not revert. It returns `(ids(), 0)`, signaling a successful transaction with "zero change."
8. Execution returns to `VaultV2.allocateInternal`.
9. VaultV2 receives the `change = 0` value.
10. VaultV2 updates its internal ledger: `_caps.allocation = (int256(_caps.allocation) + change).toUint256();`.
11. The vault's ledger is updated from `0 + 0`, resulting in 0.
12. The transaction successfully completes.

The system is now in an inconsistent state: MYTStrategy physically holds amount of the assets, but VaultV2's internal ledger (allocation) for that strategy is 0.

The same logic applies in reverse for deallocate, where funds sent from a sub-protocol would get stuck in MYTStrategy instead of being returned to the VaultV2.

## Impact Details

The impact is the permanent and irrecoverable loss of all funds that are allocated or deallocated while the killSwitch is active.

The funds are permanently lost for two reasons:

1. VaultV2 Cannot Recover Funds: The vault is now "blind" to these funds. Any attempt by an allocator to call `vault.deallocate(...)` to rescue the funds will fail. The deallocateInternal function checks the (incorrect) ledger before attempting withdrawal: `require(_caps.allocation > 0, ErrorsLib.ZeroAllocation());`. Since the ledger is 0, this check will always fail, and deallocate will revert, making recovery impossible.
2. MYTStrategy Owner Cannot Recover Funds: The MYTStrategy contract itself acts as the prison for the funds. It inherits Ownable, but the owner has no functions to sweep or withdraw arbitrary ERC20 tokens that are held by the contract. The owner's only powers are to change parameters (e.g., setRiskClass, setKillSwitch).

Because the VaultV2's accounting is corrupted and the MYTStrategy has no emergency sweep function, any funds sent to it during the "emergency" are permanently locked.

## References

Flawed allocate Function: MYTStrategy.sol

## Proof of Concept

## Proof of Concept

This Foundry test can be added to src/test/MYTStrategy.t.sol. The test fully reproduces the vulnerability: it configures the vault, activates the killSwitch, and then shows that an allocate call causes funds to be permanently trapped, failing a subsequent deallocate with ZeroAllocation.

```
import {ErrorsLib} from "../../lib/vault-v2/src/libraries/ErrorsLib.sol";

// ... inside contract MYTStrategyTest ...

    //Modify setUp()
    function setUp() public {
        //string memory rpc = vm.envString("MAINNET_RPC_URL");
        //uint256 forkId = vm.createFork(rpc, 23567434);
        //vm.selectFork(forkId);
        deployCoreContracts(18);
    }

    /**
     * @notice This test reproduces the killSwitch fund black hole vulnerability.
     * It proves that activating the killSwitch and calling 'allocate'
     * results in funds being physically transferred but not accounted for,
     * leading to permanent loss, demonstrated by a reverting 'deallocate'.
     */
    function test_VULN_killSwitch_CausesFundBlackHole() public {
        // --- 1. Set up VaultV2 environment ---
        // 'proxyOwner' (vault owner) must appoint a 'curator'
        vm.prank(proxyOwner);
        vault.setCurator(proxyOwner);

        // 'curator' must 'submit' and 'execute' addAdapter
        bytes memory addAdapterData = abi.encodeWithSelector(
            vault.addAdapter.selector,
            address(strategy)
        );
        vm.prank(proxyOwner);
        vault.submit(addAdapterData);
        vm.prank(proxyOwner);
        vault.addAdapter(address(strategy));

        // 'curator' must 'submit' and 'execute' setIsAllocator
        bytes memory setIsAllocatorData = abi.encodeWithSelector(
            vault.setIsAllocator.selector,
            address(this), // 'this' (test contract) will be the allocator
            true
        );
        vm.prank(proxyOwner);
        vault.submit(setIsAllocatorData);
        vm.prank(proxyOwner);
        vault.setIsAllocator(address(this), true);

        // 'curator' must 'submit' and 'execute' increaseAbsoluteCap
        // This is necessary to bypass the ZeroAbsoluteCap revert,
        // which would otherwise hide the killSwitch vulnerability.
        bytes memory idData = strategy.getIdData();
        bytes memory capData = abi.encodeWithSelector(
            vault.increaseAbsoluteCap.selector,
            idData,
            type(uint128).max // Set a large cap
        );
        vm.prank(proxyOwner);
        vault.submit(capData);
        vm.prank(proxyOwner);
        vault.increaseAbsoluteCap(idData, type(uint128).max);
        
        // Deposit funds into the Vault for allocation
        uint256 depositAmount = 1000e18;
        vm.startPrank(user);
        vault.deposit(depositAmount, user);
        vm.stopPrank();

        assertEq(fakeUnderlyingToken.balanceOf(address(vault)), depositAmount);
        
        // --- 2. Prepare (Arrange) ---
        uint256 allocateAmount = 500e18;
        uint256 vaultInitialBalance = fakeUnderlyingToken.balanceOf(address(vault));
        bytes32 strategyId = strategy.adapterId();

        assertEq(fakeUnderlyingToken.balanceOf(address(strategy)), 0);
        assertEq(vault.allocation(strategyId), 0);

        // ** CRITICAL STEP: Activate the Kill Switch **
        vm.prank(admin);
        strategy.setKillSwitch(true);
        assertTrue(strategy.killSwitch(), "Kill switch should be active");

        // --- 3. Execute (Act) ---
        // The allocator calls 'vault.allocate'.
        // 1. VaultV2 transfers 'allocateAmount' to 'strategy'.
        // 2. VaultV2 calls strategy.allocate().
        // 3. strategy.allocate() *silently succeeds*, returning change = 0.
        // 4. The transaction succeeds, but vault.allocation(strategyId) remains 0.
        vm.prank(address(this)); 
        vault.allocate(address(strategy), abi.encode(uint256(0)), allocateAmount);

        // --- 4. Assert (The Black Hole) ---

        // 1. Funds physically left the Vault
        assertEq(
            fakeUnderlyingToken.balanceOf(address(vault)),
            vaultInitialBalance - allocateAmount,
            "Vault balance should decrease"
        );

        // 2. Funds are now physically trapped in the Strategy contract
        assertEq(
            fakeUnderlyingToken.balanceOf(address(strategy)),
            allocateAmount,
            "Strategy balance should increase (funds are stuck)"
        );

        // 3. **THE VULNERABILITY**: Vault's internal accounting is incorrect.
        assertEq(
            vault.allocation(strategyId),
            0,
            "Vault allocation should STILL be 0 (THE BUG!)"
        );

        // --- 5. Proof of Permanent Loss ---
        // Attempt to deallocate the funds.
        // This will fail because VaultV2 checks its (incorrect) ledger
        // and reverts due to ZeroAllocation.
        
        vm.expectRevert(ErrorsLib.ZeroAllocation.selector);
        vm.prank(address(this)); // 'this' is the allocator
        vault.deallocate(address(strategy), abi.encode(uint256(0)), allocateAmount);

        // Final confirmation: funds are still trapped.
        assertEq(
            fakeUnderlyingToken.balanceOf(address(strategy)),
            allocateAmount,
            "Funds are still stuck in strategy after failed deallocate"
        );
    }
```
