# #37768 \[SC-Insight] Missing Event Emission when proposer are added prevents safe retrieval of index

**Submitted on Dec 15th 2024 at 11:28:16 UTC by @danvinci\_20 for** [**Audit Comp | Folks: Liquid Staking**](https://immunefi.com/audit-competition/folks-finance-liquid-staking-audit-competition)

* **Report ID:** #37768
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algo-liquid-staking-contracts/blob/8bd890fde7981335e9b042a99db432e327681e1a/contracts/xalgo/consensus\\_v2.py>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Design Flaw

## Description

## Brief/Intro

It's a common design pattern to emit event/log when crucial operation are carried out onchain to be used by offchain systems, adding a proposer to the to the proposers box is an important onchain action that events/log are to be emitted when a proposer is successfully added, but this was not implemented in the current version of the `consensus_v2.py`

## Vulnerability Details

The `add_proposer` method does not return or emit event when a proposer is added this can lead to lack of transparency or easy retrievability of the proposer index for subsequent operations. This will lead to lack of usability and introduce vulnerability in dependent workflow like

1. Adding of proposer admin
2. Offline and Online Registration
3. Subscribe xgov and unsubscribe xgov This is the current implementation of the add\_proposer function

```
@router.method(no_op=CallConfig.CALL)
def add_proposer(proposer: abi.Account) -> Expr:
    proposer_rekeyed_to = proposer.params().auth_address()
    num_proposers = ScratchVar(TealType.uint64)

    return Seq(
        rekey_and_close_to_check(),
        # ensure initialised
        Assert(App.globalGet(initialised_key)),
        # verify caller is register admin
        check_register_admin_call(),
        # verify proposer has been rekeyed to the app
        proposer_rekeyed_to,
        Assert(proposer_rekeyed_to.hasValue()),
        Assert(proposer_rekeyed_to.value() == Global.current_application_address()),
        # check num proposers won't exceed max
        num_proposers.store(App.globalGet(num_proposers_key)),
        Assert(num_proposers.load() < ProposersBox.MAX_NUM_PROPOSERS),
        # add proposer, verifying it hasn't already been added
        Assert(BoxCreate(Concat(AddedProposerBox.NAME, proposer.address()), Int(0))),
        BoxReplace(ProposersBox.NAME, num_proposers.load() * ProposersBox.ADDRESS_SIZE, proposer.address()),
        App.globalPut(num_proposers_key, num_proposers.load() + Int(1)),
    )
```

## Impact Details

The impact caused by this are

1. It leads to disruption of workflow, The absence of an event or explicit return value providing the proposer's index makes it challenging for users to correctly interact with functions that require the index as a parameter.
2. Increased computational complexity and taking of unnecessary risk, for user must guess or calculate the proposer's index manually by iterating through the stored data (ProposersBox), which is computationally expensive.

## Proof of Concept

## Proof of Concept

This issue can be resolved by:

1. The system should implement mechanism to retrieve the proposer index given a particular address

```
class ProposerManager:
    def __init__(self):
        self.address_list = []  # List to store proposer addresses
        self.address_to_index = {}  # Mapping to store poposer  address to index 

    def add_address(self, address):
        
        if address in self.address_to_index:
            print(f"Address {address} is already added at index {self.address_to_index[address]}.")
            return

        # Add the address to the list
        self.address_list.append(address)

        # Map the address to its index
        self.address_to_index[address] = len(self.address_list) - 1

        print(f"Address {address} added at index {self.address_to_index[address]}.")

    def get_index(self, address):
        #retrieve proposer index given the address
        return self.address_to_index.get(address, None)

    def get_address(self, index):
        
        if 0 <= index < len(self.address_list):
            return self.address_list[index]
        return None


# Example usage
manager = ProposerManager()

# Adding addresses
manager.add_address("0x1234567890abcdef1234567890abcdef12345678")
manager.add_address("0xabcdefabcdefabcdefabcdefabcdefabcdefabcd")
manager.add_address("0x9876543210fedcba9876543210fedcba98765432")

# Trying to add a duplicate address
manager.add_address("0x1234567890abcdef1234567890abcdef12345678")

# Retrieving index by address
index = manager.get_index("0xabcdefabcdefabcdefabcdefabcdefabcdefabcd")
print(f"Index of the address: {index}")

# Retrieving address by index
address = manager.get_address(1)
print(f"Address at index 1: {address}")

# Attempting to get an index that doesn't exist
print(manager.get_index("0xnonexistentaddress"))
```

2. The protocol should emit an event when proposer is added

```
# Log addition of proposer with index and address
Log(Concat(MethodSignature("AddProposer(uint64,address)"), Itob(index), proposer.address()))
```
