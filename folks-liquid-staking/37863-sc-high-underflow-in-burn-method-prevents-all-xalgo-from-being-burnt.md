# #37863 \[SC-High] Underflow in burn method prevents all xALGO from being burnt

**Submitted on Dec 17th 2024 at 19:47:02 UTC by @uhudo for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37863
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

A portion of minted xALGO cannot be burned, leading to users losing their funds. Currently, about 111,000 ALGO on Mainnet would not be able to be burnt.

## Vulnerability Details

The problem arises in `burn` method (<https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L793>) due to a possible underflow at L824 (<https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L824>). The underflow happens because the ALGO to be returned during the burn is allocated only from the `total_active_stake`, while the xALGO in reality represents also the portion of ALGO that have been received by the protocol as part of the staking rewards `total_reward`. Because the smart contract remains upgradable, a future smart contract upgrade could possibly recover these funds.

## Impact Details

The amount of xALGO that cannot be burnt is increasing with the reward that the protocol is getting, i.e. `xALGO_{lost} = xALGO_{minted} * (R-U)/(A+R-U)`, where `A` is the `total_active_stake`, `R` the `total_reward`, and `U` the `total_unclaimed_fees`. Based on the current state of the Algorand Mainnet (application ID: 1134695678 (<https://lora.algokit.io/mainnet/application/1134695678>)), this would result in about 111,000 ALGO being lost by users that are last to burn their xALGO.

## References

This is the high-level security bug found during the Audit Competition. The full report on all bugs and insights found is accessible until 2025/01/16 at <https://www.swisstransfer.com/d/4c5dff62-e56b-4c13-bc07-0bbba1e00e84>. The download is password-protected. The password is NT4SCGJ7NTJENGSDWKKLZLZ2J (the first 25 letters of authors' Algorand address: NT4SCGJ7NTJENGSDWKKLZLZ2JNXFXM5Y6HLU224TPUJXNA2IU3DBBHDTMQ). The shared folder includes the full report (PDF file) and a .zip of the full test suite project (using AlgoKit), demonstrating all found issues.

## Proof of Concept

## Proof of Concept

The test demonstrating that not all xALGO can be burned is implemented in `burn_test.py`, found in <https://www.swisstransfer.com/d/4c5dff62-e56b-4c13-bc07-0bbba1e00e84> (password is NT4SCGJ7NTJENGSDWKKLZLZ2J):

import pytest from algokit\_utils import ( TransactionParameters, ) from algokit\_utils.beta.account\_manager import AddressAndSigner from algokit\_utils.beta.algorand\_client import AlgorandClient from algokit\_utils.beta.composer import AssetTransferParams from algosdk.atomic\_transaction\_composer import ( AtomicTransactionComposer, TransactionWithSigner, ) from algosdk.error import AlgodHTTPError

from tests.consensus.conftest import BOX\_PROPOSERS\_PREFIX, Setup from tests.utils import ( available\_balance, get\_sp, )

def test\_burn\_all\_fails( algorand\_client: AlgorandClient, dispenser: AddressAndSigner, setup: Setup, ) -> None:

```
with pytest.raises(AlgodHTTPError) as e:
    # Get all xALGO in circulation
    burn_amt = available_balance(algorand_client, dispenser.address, setup.xalgo)
    atc = AtomicTransactionComposer()
    send_xalgo = TransactionWithSigner(
        algorand_client.transactions.payment(
            AssetTransferParams(
                sender=dispenser.address,
                asset_id=setup.xalgo,
                signer=dispenser.signer,
                receiver=setup.client.app_address,
                amount=burn_amt,
            )
        ),
        signer=dispenser.signer,
    )

    setup.client.compose(atc).burn(
        send_xalgo=send_xalgo,
        min_received=1,
        transaction_parameters=TransactionParameters(
            sender=dispenser.address,
            signer=dispenser.signer,
            suggested_params=get_sp(algorand_client, 3),
            accounts=[setup.proposer.address],
            boxes=[(0, BOX_PROPOSERS_PREFIX)],
            foreign_assets=[setup.xalgo],
        ),
    ).build().execute(setup.client.algod_client, 1)
assert "logic eval error: - would result negative" in str(e.value) and "opcodes=app_global_get; load 42" in str(e.value) # noqa: E501
# Incorrectly fails on L824 of `consensus_v2.py` due to underflow

return
```
