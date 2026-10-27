# IOP \_ ThunderNFT 34677 - \[Smart Contract - Insight] NFTs can not be canceled since the cancel\_order

Submitted on Tue Aug 20 2024 14:58:06 GMT-0400 (Atlantic Standard Time) by @zeroK for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34677

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Permanent freezing of NFTs
* Block stuffing

## Description

## Brief/Intro

the function `cancel_order` in the thunder\_exchange contract is meant to be used to cancel NFTs order and withdraw the nft back to its owner, this function have storage read attribute `#[storage(read)]` which mean its a pure function, however this function will call the `cancel_order() function in strategy fixed` which have write and read storage attribute, this will lead to sway compiler error since pure function can not call impure function and nfts owners can not transfer back their NFTs since they deposit it.

## Vulnerability Details

the `thunder_exchange#cancel_order()` function is implemented as below:

```sway

    /// Cancels MakerOrder
    #[storage(read)] //@audit pure function
    fn cancel_order(
        strategy: ContractId,
        nonce: u64,
        side: Side
    ) {
        let caller = get_msg_sender_address_or_panic();

        let execution_manager_addr = storage.execution_manager.read().unwrap().bits();
        let execution_manager = abi(ExecutionManager, execution_manager_addr);

        require(strategy != ZERO_CONTRACT_ID, ThunderExchangeErrors::StrategyMustBeNonZeroContract);
        require(execution_manager.is_strategy_whitelisted(strategy), ThunderExchangeErrors::StrategyNotWhitelisted);

        let strategy_caller = abi(ExecutionStrategy, strategy.bits());  
        let order = strategy_caller.get_maker_order_of_user(caller, nonce, side); // get the order for the caller

        match side {
            Side::Buy => {
                // Cancels buy MakerOrder (e.g. offer)
                strategy_caller.cancel_order(caller, nonce, side);
            },
            Side::Sell => {
                // Cancel sell MakerOrder (e.g. listing)
                if (order.is_some()) {
                    // If order is valid, then transfers the asset back to the user
                    let unwrapped_order = order.unwrap();
                    strategy_caller.cancel_order(caller, nonce, side);
                    transfer(
                        Identity::Address(unwrapped_order.maker),
                        AssetId::new(unwrapped_order.collection, unwrapped_order.token_id),
                        unwrapped_order.amount
                    );
                }
            },
        }

        log(OrderCanceled {
            user: caller,
            strategy,
            side,
            nonce,
        });
    }
```

and the cancel\_order in fixed strategy is implemented as below:

```sway


    /// Cancels MakerOrder of the user
    /// Only callable by Thunder Exchange contract
    #[storage(read, write)] //@audit impure
    fn cancel_order(
        maker: Address,
        nonce: u64,
        side: Side
    ) {
        only_exchange();

        match side {
            Side::Buy => {
                let none: Option<MakerOrder> = Option::None;
                storage.buy_order.insert((maker, nonce), none);
            },
            Side::Sell => {
                let none: Option<MakerOrder> = Option::None;
                storage.sell_order.insert((maker, nonce), none);
            },
        }
    }
```

this wat NFTs can be locked forever in scenario below:

* Bob listed NFT.
* time passed and bob want to cancel his order since the bid offer not met his interest and no one bought the NFT directly.
* the cancel\_order will revert since its impossible to call impure function inside pure function.

CHECK fuel docs about the pure and impure function in the link below:

<https://docs.fuel.network/docs/sway/blockchain-development/purity/#purity>

```
in fuel docs:

Impure functions which call other impure functions must have at least the same storage privileges or a superset of those for the function called. For example, to call a function with write access a caller must also have write access, or both read and write access. To call a function with read and write access the caller must also have both privileges

asking fuel bot too:

In Sway, a pure function cannot call an impure function. This restriction ensures that pure functions do not access any persistent storage, maintaining their purity.

If you attempt to call an impure function from a pure function, the compiler will generate an error. This is because pure functions are guaranteed not to incur storage gas costs, and calling an impure function would violate this guarantee.

```

## Impact Details

incorrect set of storage attribute in cancel\_order lead to stuck NFTs.

## References

change the `#[storage(read)]` in cancel\_order to `#[storage(read,write)]`

## Proof of concept

## Proof of Concept

create new project and run the code in the src/main.sw with forc test:

```sway
contract;
 
abi ContractA {
    #[storage(read)]  //NOTICE: change this to read and write in abi and impl the the test won't revert
    fn receive() -> u64;
}
 
impl ContractA for Contract {
    #[storage(read)]
    fn receive() -> u64 {
 
        return return_45();
    }
}

#[storage(read, write)] // call revert here because of pure calling impure
fn return_45() -> u64 {
  45
}

// no test require since the compiler revert before reaching testing stage

```
