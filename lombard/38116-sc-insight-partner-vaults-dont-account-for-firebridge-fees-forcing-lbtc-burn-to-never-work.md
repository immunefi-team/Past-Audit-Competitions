# #38116 \[SC-Insight] Partner vaults don't account for FireBridge fees, forcing LBTC burn to never wor

**Submitted on Dec 24th 2024 at 21:30:49 UTC by @OxAlix2 for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38116
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/fbtc/PartnerVault.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency

## Description

## Brief/Intro

Users can use partner vaults to deposit their FBTC tokens and get LBTC in return. The reverse is also possible by burning the LBTC tokens in return for FBTC tokens, this is done by the partner vaults calling LockedFBTC, which in return calls FireBridge to finalize the withdrawal, which charges some fees. On the other hand, the withdrawal operation has a unique key represented as the hash of the following: `recipient`, `amount`, `depositTxId`, and `outputIndex`. However, this doesn't account for the fees that the FireBridge charges.

## Vulnerability Details

To get FBTC back there's a step process that should be made, first is calling `initializeBurn`, while providing the needed amount to withdraw, the key is later created according to:

```solidity
bytes32 key = keccak256(
    abi.encode(recipient, amount, depositTxId, outputIndex)
);
```

At a later stage in the function, `lockedFbtc.redeemFbtcRequest` is called which initiates a redeem operation, <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/fbtc/PartnerVault.sol#L239-L243>. LockedFBTC later calls `IFireBridge(fbtcBridge).addMintRequest` which constructs the redemption request, the issue here is that this doesn't return the exact amount but `amount - fees`:

```solidity
function _splitFeeAndUpdate(Request memory r) internal view {
    uint256 _fee = FeeModel(feeModel).getFee(r);
    r.fee = _fee;
    r.amount = r.amount - _fee;
}

function addMintRequest(
    uint256 _amount,
    bytes32 _depositTxid,
    uint256 _outputIndex
)
    external
    onlyActiveQualifiedUser
    whenNotPaused
    returns (bytes32 _hash, Request memory _r)
{
    // ...

    // Compose request. Main -> Self
    _r = Request({
        nonce: nonce(),
        op: Operation.Mint,
        srcChain: MAIN_CHAIN,
        srcAddress: bytes(userInfo[msg.sender].depositAddress),
        dstChain: chain(),
        dstAddress: abi.encode(msg.sender),
        amount: _amount,
        fee: 0, // To be set in `_splitFeeAndUpdate`
        extra: _depositTxData,
        status: Status.Pending
    });

    // Split fee.
    _splitFeeAndUpdate(_r);

    // ...
}
```

The returned request amount gets saved:

```solidity
// Ensure that this caller can redeem for `amount` later when
// all bookkeeping off-chain is done.
$.pendingWithdrawals[key] = request;
```

**The issue here is that the amount used in the key is not equal to `request.amount`**

Later, when `finalizeBurn` is called it'll always revert:

```solidity
bytes32 key = keccak256(
    abi.encode(recipient, amount, depositTxId, outputIndex)
);
PartnerVaultStorage storage $ = _getPartnerVaultStorage();
if ($.pendingWithdrawals[key].amount != amount)
    revert NoWithdrawalInitiated();
```

## Impact Details

This blocks users from swapping their LBTC back to FBTC, which ultimately forces users to lose their FBTC.

## References

* LockedFBTC: <https://etherscan.io/address/0x8dc0d5e06995d119a9ccdb3472cc9e920389f39c#code>
* FireBridge: <https://etherscan.io/address/0xc5e2f85cb57350d3ae918d8b038f891f8ed6f6e5#code>
* <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/fbtc/PartnerVault.sol#L231-L233>
* <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/fbtc/PartnerVault.sol#L269-L270>

## Mitigation

Used the amount returned from the LockedFBTC as the `amount` in the withdrawal key.

## Proof of Concept

## Proof of Concept

To demonstrate the case where fees are >0, we need to apply the following change to `contracts/mock/LockedFBTCMock.sol`:

```diff
    function redeemFbtcRequest(
        uint256 amount,
        bytes32 depositTxId,
        uint256 outputIndex
    ) external pure returns (bytes32, FBTCPartnerVault.Request memory) {
        FBTCPartnerVault.Request memory request = FBTCPartnerVault.Request({
            op: FBTCPartnerVault.Operation.Nop,
            status: FBTCPartnerVault.Status.Unused,
            nonce: 0,
            srcChain: bytes32("test"),
            srcAddress: bytes("test"),
            dstChain: bytes32("test"),
            dstAddress: bytes("test"),
-           amount: amount,
+           amount: amount - 1,
-           fee: 0,
+           fee: 1,
            extra: bytes("extra")
        });

        return (bytes32("test"), request);
    }
```

Add the following test in `test/PartnerVault.ts` in `FBTCPartnerVault`:

```typescript
describe('FBTC DOS', async () => {
    const mintAmount = 10;

    beforeEach(async function () {
        await partnerVault.setAllowMintLbtc(true);
        await partnerVault.grantRole(operatorRoleHash, deployer.address);
        await fbtc.mint(signer1.address, mintAmount);
    });

    it('FBTC DOS PoC', async function () {
        await fbtc
            .connect(signer1)
            .approve(await partnerVault.getAddress(), mintAmount);
        await partnerVault.connect(signer1).mint(mintAmount);

        await partnerVault
            .connect(deployer)
            .initializeBurn(
                signer1.address,
                mintAmount,
                '0x65d7a28e3265b37a6474929f336521b332c1681b933f6cb9f3376673440d862a',
                0
            );

        await expect(
            partnerVault
                .connect(deployer)
                .finalizeBurn(
                    signer1.address,
                    mintAmount,
                    '0x65d7a28e3265b37a6474929f336521b332c1681b933f6cb9f3376673440d862a',
                    0
                )
        ).to.be.revertedWithCustomError(
            partnerVault,
            'NoWithdrawalInitiated'
        );
        await expect(
            partnerVault
                .connect(deployer)
                .finalizeBurn(
                    signer1.address,
                    mintAmount - 1,
                    '0x65d7a28e3265b37a6474929f336521b332c1681b933f6cb9f3376673440d862a',
                    0
                )
        ).to.be.revertedWithCustomError(
            partnerVault,
            'NoWithdrawalInitiated'
        );
    });
});
```
