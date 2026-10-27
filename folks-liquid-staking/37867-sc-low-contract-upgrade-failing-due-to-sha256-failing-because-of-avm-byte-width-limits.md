# #37867 \[SC-Low] Contract upgrade failing due to SHA256 failing because of AVM byte width limits

**Submitted on Dec 17th 2024 at 20:12:20 UTC by @uhudo for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37867
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The code is not being transparently upgradable through scheduling an upgrade, as well as the upgrade potentially failing - without blocking the protocol.

## Vulnerability Details

The code can be upgraded by admin via a two-step process, i.e. by calling `schedule_update_sc` method to schedule an update and then calling the method `update_sc` to deploy it. The scheduling makes commitments in the form of hashes of the smart contract code, i.e. the approval and clear programs, which are to be uploaded later. If the method `update_sc` is called with programs that do not meet the hash commitments, the update will be rejected. The update can succeed only if the call is made after a predefined amount of time, which is set to 1 day.

The purpose of this two-step process is to give users the option to review the new contract changes and act by burning their xALGO before the new update goes live e.g. if they do not agree with the changes or find them malicious. The downside of the current implementation is that it still requires the admin to publish somewhere the code, which corresponds to the commitments made, for the users to review.

Moreover, the implementation includes an error. The maximum size of programs on Algorand is 8192 bytes. However, the maximum byte width of the Algorand Virtual Machine is 4096 bytes. This means that when verifying whether the correct programs are to be deployed after they have been scheduled, the calculation of hashes of programs larger than 4096 bytes will fail, i.e. at <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L353>. Because a pending upgrade can be overwritten by the admin calling again `schedule_update_sc` method, the error will not result in a blocked contract.

## Impact Details

The centralization fear is one of the main downsides of liquid staking protocols and a big reason why many users do not want to use them. Any effort that can be made to improve decentralization will drive additional users to the protocol.

To resolve both of these limitations simultaneously, the contract could take advantage of box storage to store the code for the new upgrade already in the update scheduling step.

## References

This is one of the insights found during the Audit Competition. The full report on all bugs and insights found is accessible until 2025/01/16 at <https://www.swisstransfer.com/d/4c5dff62-e56b-4c13-bc07-0bbba1e00e84>. The download is password-protected. The password is NT4SCGJ7NTJENGSDWKKLZLZ2J (the first 25 letters of authors' Algorand address: NT4SCGJ7NTJENGSDWKKLZLZ2JNXFXM5Y6HLU224TPUJXNA2IU3DBBHDTMQ). The shared folder includes the full report (PDF file) and a .zip of the full test suite project (using AlgoKit), demonstrating all found issues.

## Proof of Concept

## Proof of Concept

The test showing issues with upgrades is implemented in `schedule_and_update_sc_test.py`, found in <https://www.swisstransfer.com/d/4c5dff62-e56b-4c13-bc07-0bbba1e00e84> (password is NT4SCGJ7NTJENGSDWKKLZLZ2J):

from hashlib import sha256

import pytest from algokit\_utils import ( TransactionParameters, ) from algokit\_utils.beta.account\_manager import AddressAndSigner from algokit\_utils.beta.algorand\_client import AlgorandClient from algokit\_utils.beta.composer import PayParams from algosdk.abi import ArrayStaticType, ByteType, TupleType, UintType from algosdk.abi.method import Method, Returns from algosdk.atomic\_transaction\_composer import ( AtomicTransactionComposer, TransactionWithSigner, ) from algosdk.error import AlgodHTTPError from algosdk.transaction import ApplicationCallTxn, OnComplete

from tests.consensus.conftest import BOX\_UPDATE\_PREFIX, MBR\_UPDATE\_BOX, Setup from tests.utils import ( advance\_time, get\_approval\_and\_clear\_bytes, get\_box, get\_latest\_timestamp, get\_sp, )

def test\_fails\_when\_update\_too\_large( algorand\_client: AlgorandClient, dispenser: AddressAndSigner, setup: Setup, ) -> None:

```
ap, cp = get_approval_and_clear_bytes(setup.client.algod_client, "consensus_v_two/ConsensusV2")
approval_program = ap + ap  # Double the size of program
clear_program = cp
approval_sha256 = sha256(approval_program).digest()
clear_sha256 = sha256(clear_program).digest()

# Schedule update
atc = AtomicTransactionComposer()
send_algo = TransactionWithSigner(
    algorand_client.transactions.payment(
        PayParams(
            sender=dispenser.address,
            signer=dispenser.signer,
            receiver=setup.client.app_address,
            amount=MBR_UPDATE_BOX,
        )
    ),
    signer=dispenser.signer,
)
atc.add_transaction(send_algo)

atc = setup.client.compose(atc).schedule_update_sc(
    approval_sha256=approval_sha256,
    clear_sha256=clear_sha256,
    transaction_parameters=TransactionParameters(
        sender=dispenser.address,
        signer=dispenser.signer,
        suggested_params=get_sp(algorand_client),
        boxes=[(0, BOX_UPDATE_PREFIX)],
    ),
).build().execute(setup.client.algod_client, 1)

# Wait for update
time_cur = get_latest_timestamp(algorand_client)

box_raw = get_box(algorand_client,BOX_UPDATE_PREFIX, setup.client.app_id)
data_type = TupleType(
    [
        UintType(64),
        ArrayStaticType(ByteType(), 32),
        ArrayStaticType(ByteType(), 32),
    ]
)
decoded_tuple = data_type.decode(box_raw[0])
time_target = decoded_tuple[0]

time_delta = time_target - time_cur + 1  # One more than the min difference to wait
advance_time(algorand_client, time_delta, dispenser)

# Try to update
with pytest.raises(AlgodHTTPError) as e:
    atc = AtomicTransactionComposer()
    txn_unsigned = ApplicationCallTxn(
        sender=dispenser.address,
        sp=get_sp(algorand_client, 2),
        index=setup.client.app_id,
        on_complete=OnComplete.UpdateApplicationOC,
        approval_program = approval_program,
        clear_program = clear_program,
        app_args=[
            Method(
                name="update_sc",
                args=[],
                returns=Returns(arg_type="void"),
            ).get_selector()
        ],
        boxes=[(0, BOX_UPDATE_PREFIX)],
    )

    txn = TransactionWithSigner(
        txn=txn_unsigned,
        signer=dispenser.signer,
    )
    atc.add_transaction(txn)
    atc.execute(setup.client.algod_client, 1)
assert "txn produced a too big" in str(e.value) and "byte-array" in str(e.value)

return
```
