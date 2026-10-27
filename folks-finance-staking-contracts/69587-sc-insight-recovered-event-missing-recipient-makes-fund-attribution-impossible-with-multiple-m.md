# 69587 sc insight recovered event missing recipient makes fund attribution impossible with multiple managers

**Submitted on Mar 15th 2026 at 18:48:07 UTC by @teoslaf1 for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69587
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>

## Description

`recoverERC20` emits:

```solidity
event Recovered(address indexed token, uint256 amount);
// ...
emit Recovered(tokenAddress, tokenAmount);
```

`msg.sender` — the manager who receives the tokens — is never logged. Since `MANAGER_ROLE` is a role-based permission that can be granted to multiple addresses, there is no way to attribute a recovery event to a specific manager from the event log alone. An off-chain observer must trace the raw ERC20 `Transfer` event and cross-reference it with known manager addresses to reconstruct who received the funds.

This matters for:

* Treasury monitoring and accounting systems
* Security incident response (which manager moved funds?)
* Compliance and audit trails

## Fix

Add `recipient` to the event:

```solidity
event Recovered(address indexed token, address indexed recipient, uint256 amount);

// in recoverERC20:
emit Recovered(tokenAddress, msg.sender, tokenAmount);
```

## Proof of Concept

Add the test to `/test/Staking.t.sol`

```solidity
function test_Poc_RecoveredEventMissingRecipient() public {
    address manager2 = address(0xb0b);

    // grant MANAGER_ROLE to a second manager
    vm.prank(admin);
    staking.grantRole(keccak256("MANAGER"), manager2);

    deal(address(token), address(staking), 1000 ether);

    // manager1 recovers 100 ether
    vm.recordLogs();
    vm.prank(manager);
    staking.recoverERC20(address(token), 100 ether);

    Vm.Log[] memory logs = vm.getRecordedLogs();
    // find the Recovered event
    bytes32 recoveredSig = keccak256("Recovered(address,uint256)");
    bool found = false;
    for (uint256 i = 0; i < logs.length; i++) {
        if (logs[i].topics[0] == recoveredSig) {
            found = true;
            // event only has token and amount — no recipient
            // off-chain observer cannot determine if manager or manager2 received funds
            assertEq(logs[i].topics.length, 2); // only `token` is indexed
            break;
        }
    }
    assertTrue(found, "Recovered event not emitted");

    // both managers have MANAGER_ROLE — impossible to attribute from event alone
    assertTrue(staking.hasRole(staking.MANAGER_ROLE(), manager));
    assertTrue(staking.hasRole(staking.MANAGER_ROLE(), manager2));
    assertEq(token.balanceOf(manager), 100 ether);   // actual recipient
    assertEq(token.balanceOf(manager2), 0);          // not the recipient — but event doesn't tell you this
}
```

Run it with:

```bash
forge test --match-test test_Poc_RecoveredEventMissingRecipient -vv
```
