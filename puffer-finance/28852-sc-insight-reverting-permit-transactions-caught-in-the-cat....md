# 28852 - \[SC - Insight] Reverting permit transactions caught in the cat...

Submitted on Feb 28th 2024 at 22:08:17 UTC by @MrPotatoMagic for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28852

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The functions `depositWstETH()` and `depositStETH()` allows users to obtain pufETH tokens by either providing WstETH or stETH. Before transferring tokens from the PufferDepositor.sol contract to the PufferVault.sol contract, a user needs to approve stETH or wstETH to the PufferDepositor contract.

The issue is that this approval is done through ERC20Permit's permit() function, which is wrapped around by a try-catch block. When a call to permit() reverts due to either `ERC2612ExpiredSignature()` error or `ERC2612InvalidSigner()` error, the catch block catches the issue but it does nothing in its body.

Due to this, the execution continues which causes a revert with an incorrect reason (allowance error) due to the safeTransferFrom() attempting a transfer from the depositor to vault with no allowance from the msg.sender. The real revert reason should've been `ERC2612ExpiredSignature()` error or `ERC2612InvalidSigner()` error.

## References

See how try-catch works in the solidity docs - <https://docs.soliditylang.org/en/v0.8.24/control-structures.html#try-catch>

## Impact Details

The issue has been marked as low-severity since:

1. Execution continues even though `ERC2612ExpiredSignature()` or `ERC2612InvalidSigner()` errors were encountered.
2. The transaction reverts with an allowance error, which is not the real reason for the revert. This could be misleading to users who are trying to debug the issue preventing them from depositing to the vault to obtain pufETH tokens.

## Vulnerability Details

Here is the whole process:

1. User calls function `depositStETH()`

* On Line 183, we call the permit() function on the `_ST_ETH` token with owner = msg.sender, spender = PufferDepositor contract and value = permitData.amount.

```solidity
File: PufferDepositor.sol
182:     function depositStETH(Permit calldata permitData) external restricted returns (uint256 pufETHAmount) {
183:         try ERC20Permit(address(_ST_ETH)).permit({
184:             owner: msg.sender, 
185:             spender: address(this),
186:             value: permitData.amount,
187:             deadline: permitData.deadline,
188:             v: permitData.v,
189:             s: permitData.s,
190:             r: permitData.r
191:         }) { } catch { }
192: 
193:         SafeERC20.safeTransferFrom(IERC20(address(_ST_ETH)), msg.sender, address(this), permitData.amount);
194:         return PUFFER_VAULT.deposit(permitData.amount, msg.sender);
195:     }
```

2. In the permit() function, the following occurs:

* On Line 53, we check if block.timestamp is greater than the deadline specified. If the deadline specified was block.timestamp by the user but the transaction gets included in the next block or if the permit just normally expires , the transaction would revert with the `ERC2612ExpiredSignature` error.
* Line 62 also includes an error but that would only occur if the someone else is trying to use someone else's permit (which is unlikely).
* For now, we'll use the reverting reason of `ERC2612ExpiredSignature`.

```solidity
File: ERC20Permit.sol
44:     function permit(
45:         address owner,
46:         address spender,
47:         uint256 value,
48:         uint256 deadline,
49:         uint8 v,
50:         bytes32 r,
51:         bytes32 s
52:     ) public virtual {
53:         if (block.timestamp > deadline) {
54:             revert ERC2612ExpiredSignature(deadline);
55:         }
56: 
57:         bytes32 structHash = keccak256(abi.encode(PERMIT_TYPEHASH, owner, spender, value, _useNonce(owner), deadline));
58: 
59:         bytes32 hash = _hashTypedDataV4(structHash);
60: 
61:         address signer = ECDSA.recover(hash, v, r, s);
62:         if (signer != owner) {
63:             revert ERC2612InvalidSigner(signer, owner);
64:         }
65: 
66:         _approve(owner, spender, value);
67:     }
```

3. Since there is a try-catch block being used on the permit() call, the revert is captured by the catch block. But since the catch block does not revert with the reason in its body and is empty, we continue execution on Line 193. Since the depositor contract has no allowance from the msg.sender, the call would revert with an allowance error.

```solidity
File: PufferDepositor.sol
182:     function depositStETH(Permit calldata permitData) external restricted returns (uint256 pufETHAmount) {
183:         try ERC20Permit(address(_ST_ETH)).permit({
184:             owner: msg.sender, 
185:             spender: address(this),
186:             value: permitData.amount,
187:             deadline: permitData.deadline,
188:             v: permitData.v,
189:             s: permitData.s,
190:             r: permitData.r
191:         }) { } catch { }
192: 
193:         SafeERC20.safeTransferFrom(IERC20(address(_ST_ETH)), msg.sender, address(this), permitData.amount);
194:         return PUFFER_VAULT.deposit(permitData.amount, msg.sender);
195:     }
```

## Mitigation

1. Consider removing the try-catch block since it does not serve any purpose.

OR

2. In the catch block, consider reverting with an error.

## Proof of Concept

How to use this POC:

* Add the POC to PufferDepositMainnet.fork.t.sol
* In the TestHelper.sol file, make sure to add the ETH mainnet rpc url in the setUp() function.
* Run the POC using `forge test --mt testStETHPermitDepositIssue -vvvvv`
* The traces will display how we encounter an expired deadline error but continue execution, following which we revert with an allowance error.

```solidity
function testStETHPermitDepositIssue()
        public
        giveToken(BLAST_DEPOSIT, address(stETH), alice, 200 ether)
        withCaller(alice)
    {
        Permit memory permit = _signPermit(
            _testTemps(
                "alice",
                address(pufferDepositor),
                100 ether,
                block.timestamp,
                hex"260e7e1a220ea89b9454cbcdc1fcc44087325df199a3986e560d75db18b2e253"
            )
        );

        vm.warp(block.timestamp + 12); // Assume new block is created

        vm.expectRevert("ALLOWANCE_EXCEEDED");
        pufferDepositor.depositStETH(permit, alice);
    }
```

### Traces

* In the traces below, we can observe how even though we encounter a `DEADLINE_EXPIRED` revert, the execution still continues and followingly reverts with an `ALLOWANCE_EXCEEDED` error, which is not the real reason for the revert.

```solidity
   │   │   │   ├─ [1755] stETH implementation::permit(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6], PufferDepositorProxy: [0x4aA799C5dfc01ee7d790e3bf1a7C2257CE1DcefF], 100000000000000000000 [1e20], 1708460759 [1.708e9], 28, 0x19f3a835cbe9716cc6ea9fdf6e4ead9721dd50e314f35eecd87ac4f1a19a2e8f, 0x3db8985ea74be3cd3e59d43be2eea641f0b1ccbbb007affc0755b32cbed086ca) [delegatecall]
    │   │   │   │   └─ ← revert: DEADLINE_EXPIRED
    │   │   │   └─ ← revert: DEADLINE_EXPIRED
    │   │   ├─ [6407] stETH::transferFrom(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6], PufferDepositorProxy: [0x4aA799C5dfc01ee7d790e3bf1a7C2257CE1DcefF], 100000000000000000000 [1e20])
    │   │   │   ├─ [1763] 0xb8FFC3Cd6e7Cf5a098A1c92F48009765B24088Dc::getApp(0xf1f3eb40f5bc1ad1344716ced8b8a0431d840b5783aea1fd01786bc26f35ac0f, 0x3ca7c3e38968823ccb4c78ea688df41356f182ae1d159e4ee608d30d68cef320)
    │   │   │   │   ├─ [820] stETH kernel::getApp(0xf1f3eb40f5bc1ad1344716ced8b8a0431d840b5783aea1fd01786bc26f35ac0f, 0x3ca7c3e38968823ccb4c78ea688df41356f182ae1d159e4ee608d30d68cef320) [delegatecall]
    │   │   │   │   │   └─ ← 0x00000000000000000000000017144556fd3424edc8fc8a4c940b2d04936d17eb
    │   │   │   │   └─ ← 0x00000000000000000000000017144556fd3424edc8fc8a4c940b2d04936d17eb
    │   │   │   ├─ [2996] stETH implementation::transferFrom(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6], PufferDepositorProxy: [0x4aA799C5dfc01ee7d790e3bf1a7C2257CE1DcefF], 100000000000000000000 [1e20]) [delegatecall]
    │   │   │   │   └─ ← revert: ALLOWANCE_EXCEEDED
    │   │   │   └─ ← revert: ALLOWANCE_EXCEEDED
    │   │   └─ ← revert: ALLOWANCE_EXCEEDED
    │   └─ ← revert: ALLOWANCE_EXCEEDED
    ├─ [0] VM::stopPrank()
```
