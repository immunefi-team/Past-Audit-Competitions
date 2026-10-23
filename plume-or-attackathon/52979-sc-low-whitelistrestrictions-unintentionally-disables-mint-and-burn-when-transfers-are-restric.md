# 52979 sc low whitelistrestrictions unintentionally disables mint and burn when transfers are restricted

**Submitted on Aug 14th 2025 at 14:45:31 UTC by @RevertLord for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52979
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/restrictions/WhitelistRestrictions.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief / Intro

I found a critical logic conflict in WhitelistRestrictions: when transfers are disabled (`transfersAllowed == false`), the module requires both `from` and `to` to be whitelisted for any transfer to pass. Because `address(0)` cannot be whitelisted, standard ERC20 mint (`from == address(0)`) and burn (`to == address(0)`) operations become impossible while restrictions are active.

This locks the issuer out of supply management and can permanently disrupt core token operations.

### Vulnerability Details

Common pattern in the module:

* `addToWhitelist(address)` rejects `address(0)` (or batch-add silently skips it).
* `isTransferAllowed(from, to, amount)` returns true only if `transfersAllowed == true` OR both endpoints are whitelisted.
* Mint is implemented as a transfer from `address(0)` -> user.
* Burn is implemented as a transfer from user -> `address(0)`.

Since `address(0)` can never be whitelisted, both mint and burn fail whenever transfers are restricted, even for holders and admins who are otherwise authorized.

### Impact Details

* Severity: Critical
* In-scope impact: Permanent freezing of core supply mechanics (mint/burn). The protocol cannot expand or reduce supply while restrictions are active, which at minimum impedes redemptions and at worst stalls product operation entirely.

### Suggested Mitigation

Special-case mint and burn in the transfer restriction check:

```solidity
if (!ws.transfersAllowed) {
    // Always allow mint or burn under restrictions
    if (from == address(0) || to == address(0)) {
        return true;
    }
    return ws.isWhitelisted[from] && ws.isWhitelisted[to];
}
return true;
```

Alternatively:

* Introduce distinct allowlists for mint and burn endpoints.
* Document and enforce that the restriction module must not gate mint/burn paths.

## Proof of Concept

A Foundry test (`WhitelistBlocksMintBurn.t.sol`) sets up `ArcToken` via the factory and links `WhitelistRestrictions`. With transfers allowed, mint and burn succeed. After setting `transfersAllowed(false)`, attempts to mint and burn revert with a transfer restriction error, confirming that supply operations are blocked under active restrictions.

## PoC Execution

{% stepper %}
{% step %}

### Clone repository

```bash
git clone https://github.com/immunefi-team/attackathon-plume-network.git
cd attackathon-plume-network
```

{% endstep %}

{% step %}

### Initialize as a Foundry project

```bash
forge init --force
```

{% endstep %}

{% step %}

### Clean up default files created by forge init

```bash
rm src/Counter.sol test/Counter.t.sol
```

{% endstep %}

{% step %}

### Install the correct versions of the required dependencies

```bash
forge install solidstate-network/solidstate-solidity
forge install OpenZeppelin/openzeppelin-contracts
forge install OpenZeppelin/openzeppelin-contracts-upgradeable
```

{% endstep %}

{% step %}

### Create the final foundry.toml file with the correct configuration and remappings

Create a file `foundry.toml` with the provided configuration (keeps solc, evm version, remappings, etc.).

Example content used in PoC:

```toml
[profile.default]
solc = "0.8.25"
evm_version = "cancun"
src = "src"
out = "out"
libs = ["lib"]
ffi = true
ast = true
build_info = true
extra_output = ["storageLayout"]
optimizer = true
optimizer_runs = 200
remappings = [
    "@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/",
    "@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/",
    "@solidstate/=lib/solidstate-solidity/contracts/",
    "forge-std/=lib/forge-std/src/"
]

[fmt]
single_line_statement_blocks = "multi"
multiline_func_header = "params_first"
sort_imports = true
contract_new_lines = true
bracket_spacing = true
int_types = "long"
quote_style = "double"
number_underscore = "thousands"
wrap_comments = true
```

{% endstep %}

{% step %}

### Create PoC test file

Create the PoC file at `test/WhitelistBlocksMintBurn.t.sol` (code below).
{% endstep %}

{% step %}

### Run the test

```bash
forge test test/WhitelistBlocksMintBurn.t.sol -vv --via-ir
```

{% endstep %}
{% endstepper %}

<details>

<summary>PoC Code (WhitelistBlocksMintBurn.t.sol)</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";
import "../arc/src/ArcToken.sol";
import "../arc/src/ArcTokenFactory.sol";
import "../arc/src/restrictions/RestrictionsRouter.sol";
import "../arc/src/restrictions/WhitelistRestrictions.sol";
import "../arc/src/restrictions/RestrictionTypes.sol";

contract WhitelistBlocksMintBurn_PoC is Test {
    ArcTokenFactory internal factory;
    RestrictionsRouter internal router;
    ArcToken internal arcToken;
    WhitelistRestrictions internal whitelistModule;
    
    address internal owner = address(0x1);
    address internal user = address(0x2);

    function setUp() public {
        vm.prank(owner);
        router = new RestrictionsRouter();
        router.initialize(owner);

        vm.prank(owner);
        factory = new ArcTokenFactory();
        factory.initialize(address(router));

        // When the test contract (as owner) calls createToken, the factory gives
        // all roles on the new ArcToken to the test contract itself (address(this)).
        address tokenAddress = factory.createToken(
            "Test RWA", "TRWA", 0, address(0), "uri", owner, 18
        );
        arcToken = ArcToken(tokenAddress);

        address moduleAddress = arcToken.getRestrictionModule(
            RestrictionTypes.TRANSFER_RESTRICTION_TYPE
        );
        whitelistModule = WhitelistRestrictions(moduleAddress);

        // The test contract grants the necessary roles to our `owner` EOA
        // Grant ArcToken roles to the owner address
        arcToken.grantRole(arcToken.MINTER_ROLE(), owner);
        arcToken.grantRole(arcToken.BURNER_ROLE(), owner);

        // Grant WhitelistRestrictions roles to the owner address
        whitelistModule.grantRole(whitelistModule.ADMIN_ROLE(), owner);
        whitelistModule.grantRole(whitelistModule.MANAGER_ROLE(), owner);
    }

    function test_MintAndBurn_Succeeds_WhenUnrestricted() public {
        assertTrue(whitelistModule.transfersAllowed(), "Transfers should be allowed by default");

        // Minting now works because `owner` has MINTER_ROLE
        vm.prank(owner);
        arcToken.mint(user, 100 ether);
        assertEq(arcToken.balanceOf(user), 100 ether);

        // Burning now works because `owner` has BURNER_ROLE
        vm.prank(owner);
        arcToken.burn(user, 50 ether);
        assertEq(arcToken.balanceOf(user), 50 ether);
    }

    function test_FAIL_MintAndBurn_Fails_WhenRestricted() public {
        console.log("--- Test: Whitelist module blocks mint() and burn() ---");
        
        vm.prank(owner);
        arcToken.mint(user, 100 ether);
        
        vm.prank(owner);
        whitelistModule.addToWhitelist(user);
        
        console.log("Admin is enabling whitelist restrictions...");
        vm.prank(owner);
        whitelistModule.setTransfersAllowed(false);
        assertFalse(whitelistModule.transfersAllowed(), "Transfers should now be restricted");

        console.log("Attempting to mint... This should fail.");
        vm.expectRevert(bytes(abi.encodeWithSignature("TransferRestricted()")));
        vm.prank(owner);
        arcToken.mint(user, 50 ether);
        console.log("Mint was successfully blocked, as expected.");

        console.log("Attempting to burn... This should also fail.");
        vm.expectRevert(bytes(abi.encodeWithSignature("TransferRestricted()")));
        vm.prank(owner);
        arcToken.burn(user, 50 ether);
        console.log("Burn was successfully blocked, as expected.");
        
        console.log("VULNERABILITY CONFIRMED: Core token functions (mint/burn) are permanently disabled while restrictions are active.");
    }
}
```

</details>

<details>

<summary>Execution Logs</summary>

Run `forge test test/WhitelistBlocksMintBurn.t.sol -vv --via-ir`. You'll see the following logs:

```
Ran 2 tests for test/WhitelistBlocksMintBurn.t.sol:WhitelistBlocksMintBurn_PoC
[PASS] test_FAIL_MintAndBurn_Fails_WhenRestricted() (gas: 272032)
Logs:
  --- Test: Whitelist module blocks mint() and burn() ---
  Admin is enabling whitelist restrictions...
  Attempting to mint... This should fail.
  Mint was successfully blocked, as expected.
  Attempting to burn... This should also fail.
  Burn was successfully blocked, as expected.
  VULNERABILITY CONFIRMED: Core token functions (mint/burn) are permanently disabled while restrictions are active.

[PASS] test_MintAndBurn_Succeeds_WhenUnrestricted() (gas: 162942)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 13.72ms (5.03ms CPU time)
```

</details>
