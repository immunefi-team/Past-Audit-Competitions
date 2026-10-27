# #37864 \[SC-Insight] Over-charging users on delayed mint

**Submitted on Dec 17th 2024 at 19:55:37 UTC by @uhudo for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37864
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The protocol over-charges users for the necessary storage fees when using the delay minting option of xALGO, effectively reducing the users' returns.

## Vulnerability Details

When a user calls `delayed_mint` method (<https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L695>), it stores the user's mint information in box storage. The user has to pay the protocol ALGO deposit (i.e. increase in the minimum balance requirement of the application's account) for the creation of this box. The deposit is charged according to the number of bytes stored in the box and its name. The box gets deleted upon claiming the minted xALGO with `claim_delayed_mint` (<https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L732>), freeing the storage ALGO deposit. The storage ALGO deposit gets returned to the caller of the `claim_delayed_mint` method, which can be anyone - not necessarily the user who paid for the creation of the box. This is meant to reward anyone who automates the claiming process for the users. However, the protocol is over-charing the users for this feature. The box namely includes unnecessary redundancy, which is costing users more ALGO than necessary. The box stores the address of the minter both as part of the box name (<https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L733>) and its contents (<https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py#L736>). This results in the minter unnecessarily paying for 32 bytes (i.e. length of Algorand address) of storage.

## Impact Details

The costs of of overcharging amount to 0.0128 ALGO on each mint. Assuming Folks Finance reaches the same levels of popularity as Lido - the most prominent liquid staking provider on Ethereum, which processes about 5M withdrawals per year (as per Lido analytics <https://dune.com/queries/2475364/4072036>) and assuming the number of mints is in the same range as the number of withdrawals, this amounts to 64,000 ALGO over-charged to xALGO users per year, effectively (covertly) reducing their return.

The issue could be easily solved by removing the redundancy in the box content information.

## References

This is one of two low-level security bugs found during the Audit Competition. The full report on all bugs and insights found is accessible until 2025/01/16 at <https://www.swisstransfer.com/d/4c5dff62-e56b-4c13-bc07-0bbba1e00e84>. The download is password-protected. The password is NT4SCGJ7NTJENGSDWKKLZLZ2J (the first 25 letters of authors' Algorand address: NT4SCGJ7NTJENGSDWKKLZLZ2JNXFXM5Y6HLU224TPUJXNA2IU3DBBHDTMQ). The shared folder includes the full report (PDF file) and a .zip of the full test suite project (using AlgoKit), demonstrating all found issues.

## Proof of Concept

## Proof of Concept

The test demonstrating that not all xALGO can be burned is implemented in `claim_delayed_test.py`, found in <https://www.swisstransfer.com/d/4c5dff62-e56b-4c13-bc07-0bbba1e00e84> (password is NT4SCGJ7NTJENGSDWKKLZLZ2J):

from algokit\_utils import ( CreateTransactionParameters, TransactionParameters, ) from algokit\_utils.beta.account\_manager import AddressAndSigner from algokit\_utils.beta.algorand\_client import AlgorandClient from algokit\_utils.beta.composer import AssetTransferParams, PayParams from algosdk.abi import AddressType from algosdk.atomic\_transaction\_composer import ( AtomicTransactionComposer, TransactionWithSigner, ) from algosdk.transaction import ApplicationUpdateTxn

import smart\_contracts.artifacts.consensus\_v\_one.consensus\_client as cv1 import smart\_contracts.artifacts.consensus\_v\_two\_one.consensus\_client as cv21 from tests.consensus.conftest import ( BOX\_PROPOSER\_ADMIN\_PREFIX, BOX\_PROPOSERS\_PREFIX, BOX\_USER\_DELAY\_MINT\_PREFIX, CONSENSUS\_DELAY, MBR\_ACCOUNT, MBR\_ASSET, MBR\_PROPOSER\_ADMIN\_EMPTY\_BOX, MBR\_PROPOSERS\_BOX, MBR\_USER\_DELAY\_MINT\_BOX, MBR\_USER\_DELAY\_MINT\_BOX\_NEW, Defaults, Setup, ) from tests.utils import ( create\_and\_fund\_account, get\_approval\_and\_clear\_bytes, get\_sp, wait\_for\_rounds, )

mint\_amt = 10\*\*9

def test\_v2\_0( algorand\_client: AlgorandClient, dispenser: AddressAndSigner, setup: Setup, ) -> None:

```
atc = AtomicTransactionComposer()
# Mint with delay
mbr_txn = TransactionWithSigner(
    algorand_client.transactions.payment(
        PayParams(
            sender=dispenser.address,
            signer=dispenser.signer,
            receiver=setup.client.app_address,
            amount=MBR_USER_DELAY_MINT_BOX,
        )
    ),
    signer=dispenser.signer,
)
atc.add_transaction(mbr_txn)

send_algo = TransactionWithSigner(
    algorand_client.transactions.payment(
        PayParams(
            sender=dispenser.address,
            signer=dispenser.signer,
            receiver=setup.client.app_address,
            amount=mint_amt,
        )
    ),
    signer=dispenser.signer,
)

nonce = 0
nonce_bytes = nonce.to_bytes(2, "big")
boxes = [
    (0, BOX_PROPOSERS_PREFIX),
    (0, BOX_USER_DELAY_MINT_PREFIX + AddressType().encode(dispenser.address) + nonce_bytes),
]

atc = setup.client.compose(atc).delayed_mint(
    send_algo=send_algo,
    nonce=nonce_bytes,
    transaction_parameters=TransactionParameters(
        sender=dispenser.address,
        signer=dispenser.signer,
        suggested_params=get_sp(algorand_client, 2),
        accounts=[setup.proposer.address],
        boxes=boxes,
        foreign_assets=[setup.xalgo],
    ),
).build().execute(setup.client.algod_client, 1)

# Wait for 320 rounds to pass
wait_for_rounds(algorand_client, CONSENSUS_DELAY, dispenser)

# Claim mint
setup.client.claim_delayed_mint(
    receiver=dispenser.address,
    nonce=nonce_bytes,
    transaction_parameters=TransactionParameters(
        sender=dispenser.address,
        signer=dispenser.signer,
        suggested_params=get_sp(algorand_client, 3),
        accounts=[setup.proposer.address],
        boxes=boxes,
        foreign_assets=[setup.xalgo],
    ),
)

return
```

def test\_v2\_1( algorand\_client: AlgorandClient, dispenser: AddressAndSigner, ) -> None:

```
# Setup V2.1
# -------------------------------------

# Create V1 client
client = cv1.ConsensusClient(algorand_client.client.algod)

# ------------------------------
# ----- Create V1 contract -----
# ------------------------------
client.create_create(
    admin=dispenser.address,
    register_admin=dispenser.address,
    min_proposer_balance=0,
    max_proposer_balance=Defaults().max_proposer_balance,
    premium=Defaults().premium,
    fee=Defaults().fee,
    transaction_parameters=CreateTransactionParameters(
        sender=dispenser.address,
        signer=dispenser.signer,
        extra_pages=3,
    ),
)

# Fund 1st proposer with the total stake recorded in SC on mainnet + total rewards recorded in SC + for rekey fee + MBR  # noqa: E501
proposer = create_and_fund_account(algorand_client, dispenser, algo_amount=9505527975627 + 123438790257 + 1000 + MBR_ACCOUNT)  # noqa: E501

# Rekey proposer to app
algorand_client.send.payment(
    PayParams(
        sender=proposer.address,
        signer=proposer.signer,
        receiver=proposer.address,
        amount=0,
        rekey_to=client.app_address,
    )
)

# Fund the application with MBR needed later to create boxes and opt into asset
algorand_client.send.payment(
    PayParams(
        sender=dispenser.address,
        signer=dispenser.signer,
        receiver=client.app_address,
        amount=MBR_ACCOUNT + MBR_ASSET + MBR_PROPOSERS_BOX + MBR_PROPOSER_ADMIN_EMPTY_BOX,
    )
)

# ------------------------------
# --- Initialize V1 contract ---
# ------------------------------
boxes = [
    (0, BOX_PROPOSERS_PREFIX),
    (0, BOX_PROPOSER_ADMIN_PREFIX + AddressType().encode(proposer.address)),
]
res = client.initialise(
    proposer=proposer.address,
    transaction_parameters=TransactionParameters(
        sender=dispenser.address,
        signer=dispenser.signer,
        suggested_params=get_sp(algorand_client, 2),
        boxes=boxes,
    ),
)

xalgo_id = res.tx_info["inner-txns"][0]["asset-index"]

# Opt dispenser into xALGO
algorand_client.send.asset_transfer(
    AssetTransferParams(
        sender=dispenser.address,
        receiver=dispenser.address,
        amount=0,
        asset_id=xalgo_id,
        signer=dispenser.signer,
    )
)

# ------------------------------
# ----- Mint on V1 contract ----
# ------------------------------
send_algo = TransactionWithSigner(
    algorand_client.transactions.payment(
        PayParams(
            sender=dispenser.address,
            signer=dispenser.signer,
            receiver=proposer.address, # Payment for mint goes to the proposer - just in V1!
            amount=0,
        )
    ),
    signer=dispenser.signer,
)

res = client.mint(
    send_algo=send_algo,
    transaction_parameters=TransactionParameters(
        sender=dispenser.address,
        signer=dispenser.signer,
        suggested_params=get_sp(algorand_client, 2),
        accounts=[proposer.address],
        foreign_assets=[xalgo_id],
        boxes=[(0, BOX_PROPOSERS_PREFIX)],
    ),
)

# ------------------------------
# ---- Deploy V2.1 contract ----
# ------------------------------
ap, cp = get_approval_and_clear_bytes(client.algod_client, "consensus_v_two_one/ConsensusV21")

txn = TransactionWithSigner(
    txn=ApplicationUpdateTxn(
        sender=dispenser.address,
        index=client.app_id,
        approval_program=ap,
        clear_program=cp,
        sp=get_sp(algorand_client, 1),
    ),
    signer=dispenser.signer,
)
atc = AtomicTransactionComposer()
atc.add_transaction(txn).execute(client.algod_client, 1)

# After deploy of new contract, switch to new client (same app ID)
client = cv21.ConsensusClient(client.algod_client, app_id=client.app_id)

# ------------------------------
# ---Initialize V2.1 contract --
# ------------------------------
client.initialise(
    transaction_parameters=TransactionParameters(
        sender=dispenser.address,
        signer=dispenser.signer,
    ),
)

# ------------------------------
# ---- Enable delay minting ----
# ------------------------------
client.pause_minting(
    minting_type="can_delay_mint",
    to_pause=False,
    transaction_parameters=TransactionParameters(
        sender=dispenser.address,
        signer=dispenser.signer,
        suggested_params=get_sp(algorand_client, 1),
    ),
)




# -------------------------------------

atc = AtomicTransactionComposer()
# Mint with delay
mbr_txn = TransactionWithSigner(
    algorand_client.transactions.payment(
        PayParams(
            sender=dispenser.address,
            signer=dispenser.signer,
            receiver=client.app_address,
            amount=MBR_USER_DELAY_MINT_BOX_NEW,
        )
    ),
    signer=dispenser.signer,
)
atc.add_transaction(mbr_txn)

send_algo = TransactionWithSigner(
    algorand_client.transactions.payment(
        PayParams(
            sender=dispenser.address,
            signer=dispenser.signer,
            receiver=client.app_address,
            amount=mint_amt,
        )
    ),
    signer=dispenser.signer,
)

nonce = 0
nonce_bytes = nonce.to_bytes(2, "big")
boxes = [
    (0, BOX_PROPOSERS_PREFIX),
    (0, BOX_USER_DELAY_MINT_PREFIX + AddressType().encode(dispenser.address) + nonce_bytes),
]

atc = client.compose(atc).delayed_mint(
    send_algo=send_algo,
    nonce=nonce_bytes,
    transaction_parameters=TransactionParameters(
        sender=dispenser.address,
        signer=dispenser.signer,
        suggested_params=get_sp(algorand_client, 2),
        accounts=[proposer.address],
        boxes=boxes,
        foreign_assets=[xalgo_id],
    ),
).build().execute(client.algod_client, 1)

# Wait for 320 rounds to pass
wait_for_rounds(algorand_client, CONSENSUS_DELAY, dispenser)

# Claim mint
client.claim_delayed_mint(
    receiver=dispenser.address,
    nonce=nonce_bytes,
    transaction_parameters=TransactionParameters(
        sender=dispenser.address,
        signer=dispenser.signer,
        suggested_params=get_sp(algorand_client, 3),
        accounts=[proposer.address],
        boxes=boxes,
        foreign_assets=[xalgo_id],
    ),
)

return
```
