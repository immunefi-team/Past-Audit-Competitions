# 31264 - \[SC - Insight] Multiple Reports QALowOOS Medium

## Multiple Reports: QA/Low/OOS Medium

Submitted on May 15th 2024 at 22:57:26 UTC by @The\_Seraphs for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31264

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

### Description

## AS PER RECOMMENDATION FROM PROJECT IN DISCORD:

Ref: <https://discord.com/channels/787092485969150012/1232293258760028180/1238623014044827728>

### QA

#### (1) Use of a modifier to remove repetition and clean-up functions

In multiple contracts the repitition of the following require statement is present

e.g. `AlchemixGovernor.sol`, `VotingEscrow.sol`, `Voting.sol`.

```solidity
    require(msg.sender == admin, "not admin");
```

The protocol could benefit from implementing a `modifier` in place of the `require()` statements that exists across multiple functions and contracts.

```solidity
    modifier onlyAdmin() {
        require(msg.sender == admin, "not admin");
        _;
    }
```

#### (2) Zero address checks

There are several contracts that utilise a zero address check in constructors or functions. However, the use of `require` conditions throughout the contracts may become gas heavy with repeated calls.

**Examples from `RevenueHandler`, `Minter`, `VotingEscrow`, `RewardPoolManager`, `Fluxtoken`**

```solidity
    constructor(address _veALCX, address _treasury, uint256 _treasuryPct) Ownable() {
        veALCX = _veALCX;
        require(_treasury != address(0), "treasury cannot be 0x0");
        treasury = _treasury;
        require(treasuryPct <= BPS, "treasury pct too large");
        treasuryPct = _treasuryPct;
    }

...

    function setTreasury(address _treasury) external {
        require(msg.sender == admin, "not admin");
        require(_treasury != address(0), "treasury cannot be 0x0");
        treasury = _treasury;
        emit TreasuryUpdated(_treasury);
    }

...

    function delegate(address delegatee) public {
        require(delegatee != address(0), "cannot delegate to zero address");
        return _delegate(msg.sender, delegatee);
    }

...

    function swapOutRewardPoolToken(uint256 i, address oldToken, address newToken) external {
        require(msg.sender == admin, "not admin");
        require(rewardPoolTokens[i] == oldToken, "incorrect token");
        require(newToken != address(0));

        isRewardPoolToken[oldToken] = false;
        isRewardPoolToken[newToken] = true;
        rewardPoolTokens[i] = newToken;
    }

...

    function whitelist(address _token) public {
        require(msg.sender == admin, "not admin");
        require(_token != address(0), "cannot be zero address");
        _whitelist(_token);
    }

...

    function setMinter(address _minter) external onlyMinter {
        require(_minter != address(0), "FluxToken: minter cannot be zero address");
        minter = _minter;
    }
```

#### Suggestions:

**The protocols contract's could be improved in several ways, to name a couple:**

1. Gas Efficiency: They reduce the cost of deploying and interacting with your contract by avoiding the storage of error strings in the bytecode. This efficiency is important for frequently called functions and during high gas price periods on the Ethereum network.
2. Code Clarity and Maintenance: Custom errors help in organising and streamlining error handling code. By defining errors in a single location and using them throughout the contract, you make your codebase easier to understand and modify. This structured approach is beneficial in large projects, with complex functions that repeat a lot of the checks - removing the need for duplication of code.

**Suggested custom error:**

**1. Basic Zero Address Error** A basic custom error for zero address checks.

```solidity
// Custom error to indicate an action was given a zero address
error ZeroAddress();
```

**2. Error with Context** You can enhance the custom error by including a parameter that specifies the context or the function where the error occurred.

```solidity
// Custom error with description of where the zero address was used
error ZeroAddress(string action);
```

**Usage example:**

```solidity
    function setTreasury(address _treasury) external override onlyOwner {
        if (_treasury == address(0)) {
            revert ZeroAddress("Updating treasury address");
        treasury = _treasury;
        emit TreasuryUpdated(_treasury);
    }
```

**3. Mixed Errors** In scenarios where different checks might lead to a zero address error under different conditions, defining multiple custom errors might be appropriate.

```solidity
    error InvalidRecipient();
    error NoZeroAddressAllowed(string parameter);
```

**Usage example:**

```solidity
...

    function delegate(address delegatee) public {
        if (delegatee == address(0)) {
            revert InvalidRecipient;
        return _delegate(msg.sender, delegatee);
    }

...
        
    function _mint(address _to, uint256 _tokenId) internal returns (bool) {
        // Throws if `_to` is zero address
        if (_to == address(0)) {
         NoZeroAddressAllowed("_to");
        
        ...
        
    }
```

#### (3) Inefficiency of loops in functions

The current implementation to add revenue tokens can be improved, saving gas, by introducing mapping into the contract, additionally, adjusting the remove revenue token to match the changes

***NB**: Changes to this function would then require adjustments to other existing functions such as, `checkpoint()`, which uses iteration of the `revenueTokens` array. Using an array to store the keys of the mapping and then iterate, would need to be implemented in the function.*

**Suggested changes**

```diff
-    address[] public revenueTokens;
+    mapping(address => bool) public revenueTokens;
```

```diff
    function addRevenueToken(address _token) public {
-       uint256 length = revenueTokens.length;
-       for (uint256 i = 0; i < length; i++) {
-           if (revenueTokens[i] == revenueToken) {
+           if (revenueTokens[_token]) {
                revert("Token already exists");
            }
-       revenueTokens.push(revenueToken);
+       revenueTokens[_token] = true;
        emit RevenueTokenTokenAdded(revenueToken);
    }
}
```

#### **Results: Gas saving**

***NB:** I made a new function and left the old one in, so the deployment cost won't have reduced due to this.*

```bash
// OLD FUNCTION IMPLEMENTATION
| src/RevenueHandler.sol:RevenueHandler contract |                 |       |        |       |         |
|------------------------------------------------|-----------------|-------|--------|-------|---------|
| Deployment Cost                                | Deployment Size |       |        |       |         |
| 2021361                                        | 9388            |       |        |       |         |
| Function Name                                  | min             | avg   | median | max   | # calls |
| addRevenueToken                                | 69249           | 69249 | 69249  | 69249 | 1       |
| owner                                          | 2343            | 2343  | 2343   | 2343  | 1       |

// NEW FUNCTION IMPLEMENTATION
| src/RevenueHandler.sol:RevenueHandler contract |                 |       |        |       |         |
|------------------------------------------------|-----------------|-------|--------|-------|---------|
| Deployment Cost                                | Deployment Size |       |        |       |         |
| 2076654                                        | 9644            |       |        |       |         |
| Function Name                                  | min             | avg   | median | max   | # calls |
| addRevenueTokenP                               | 44044           | 44044 | 44044  | 44044 | 1       |
| owner                                          | 2431            | 2431  | 2431   | 2431  | 1       |
```

```solidity
function removeRevenueToken(address revenueToken) public {
    if (!revenueTokens[revenueToken]) {
        revert("revenue token does not exist");
    }
    delete revenueTokens[revenueToken];
    emit RevenueTokenRemoved(revenueToken);
}

```

### Low

#### (1) RenounceOwnership still active from inherited Ownable contract

#### Title

Potential risk in `RevenueHandler` Due to `renounceOwnership()` inherited from Ownable contract

#### Overview

The `renounceOwnership` function inherited from OpenZeppelin's `Ownable` contract allows the contract owner to permanently transfer ownership to the zero address, effectively rendering the contract without an owner. This function could lock out administrative functions and prevent further updates or critical management actions in the `RevenueHandler` contract.

#### Effected Component

**Contract**: `RevenueHandler`

#### POC

* Add the following function to `RevenueHandler.t.sol`

```solidity
    function testRenounceOwnership() external {
        revenueHandler.renounceOwnership();
        assertEq(revenueHandler.owner(), address(0), "owner should be 0x0");
        // attempt to call a function in the contract and expect it not to work
        hevm.expectRevert();
        revenueHandler.addRevenueToken(dai);
        
    }
```

* Run in cli `forge test --mt testRenounceOwnership -vvvv` for full visibility of the trace

**Results**

```shell
Ran 1 test for src/test/RevenueHandler.t.sol:RevenueHandlerTest
[PASS] testRenounceOwnership() (gas: 14668)
Traces:
  [15575] RevenueHandlerTest::testRenounceOwnership()
    ├─ [6981] RevenueHandler::renounceOwnership()
    │   ├─ emit OwnershipTransferred(previousOwner: RevenueHandlerTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], newOwner: 0x0000000000000000000000000000000000000000)
    │   └─ ← [Stop] 
    ├─ [343] RevenueHandler::owner() [staticcall]
    │   └─ ← [Return] 0x0000000000000000000000000000000000000000
    ├─ [0] VM::expectRevert(custom error f4844814:)
    │   └─ ← [Return] 
    ├─ [651] RevenueHandler::addRevenueToken(0x6B175474E89094C44Da98b954EedeAC495271d0F)
    │   └─ ← [Revert] revert: Ownable: caller is not the owner
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.88s (145.67µs CPU time)

Ran 1 test suite in 6.23s (5.88s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

#### Impact

Permanent loss of contract control:

* Loss of ability to call the following functions:
  * `addRevenueToken`
  * `setTreasury`
  * `setTreasuryPct`
  * `enableRevenueToken`
  * `disableRevenueToken`
  * `setPoolAdapter`
  * `setDebtToken`
  * `removeAlchemicToken`
  * `addAlchemicToken`
  * `removeRevenueToken`
  * `addRevenueToken`

#### Recommendation

Override the `renounceOwnership` function in `RevenueHandler` to make it redundant, ensuring that ownership cannot be unintentionally or maliciously renounced. This can be achieved by either removing the function body or reverting any calls to it:

```solidity
function renounceOwnership() public override onlyOwner {
    revert("Operation not permitted");
}
```

### OOS: Medium severity

### **Brief/Intro**

The **AlchemicTokenV2Base** contract is integral to managing upgradeable alchemic tokens, yet it currently lacks a designated storage gap. Such a gap is pivotal for safely introducing new state variables in future contract versions without disturbing the existing storage layout. The omission of this storage gap could result in inadvertent overwriting of state variables in derived contracts, which could lead to significant disruptions or even financial losses.

### **Vulnerability Details**

#### **Components Affected**

* Contract Name: **`AlchemicTokenV2Base`**
* Functionality: Upgradeability and State Variable Management

### **Impact Details**

The **AlchemicTokenV2Base** contract is designed to facilitate the upgradeable framework of the token system. Without a storage gap, there's a substantial risk that any future additions of state variables could overwrite existing variables in contracts that inherit from this base.

**Proposed fix:**

```solidity
contract AlchemicTokenV2Base is ERC20Upgradeable, AccessControlUpgradeable, IERC3156FlashLender, ReentrancyGuardUpgradeable {
    uint256[50] private __gap; // Added storage gap to safeguard future upgrades
}
```

### **References**

* Refer to the OpenZeppelin documentation on this topic: [**OpenZeppelin Upgradeable Contracts, Storage Gaps**](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable#storage-gaps)

### Proof of Concept

**All required POCs are with the respective reports above**
