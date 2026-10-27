# Attackathon \_ Fuel Network 33360 - \[Blockchain\_DLT - Medium] The typescript SDK has no awareness of

Submitted on Thu Jul 18 2024 16:50:03 GMT-0400 (Atlantic Standard Time) by @n4nika for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33360

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/FuelLabs/fuels-ts/tree/v0.91.0>

Impacts:

* A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

The `Typescript SDK` provides the `fund` function which retrieves `UTXOs`, which belong to the owner and can be used to fund the request in question, from fuel's graphql api. These then get added to the request making it possible to send it to the network as it now has inputs which can be spent by its outputs. Now this works when a user only wants to fund one transaction per block as in the next block, the spent UTXO will not exist anymore. However if a user wants to fund multiple transactions within one block, the following can happen:

It is important to note, that the graphql API will return a random UTXO which has enough value to fund the transaction in question.

* user has 2 spendable `UTXOs` in their wallet which can cover all expenses
* user funds transaction `tA` with an input gotten from the API `iA`
* user submits `tA` to fuel
* `iA` is still in possession of the user as no new block has been produced
* user funds a transaction `tB` and gets the same input `iA` from the API
* user tries to submit transaction `tB` to fuel but now one of the following can happen:

1. if the recipient and all other parameters are the same as in `tA`, submission will fail as `tB` will have the same `txHash` as `tA`
2. if the parameters are different, there will be a collision in the `txpool` and `tA` will be removed from the `txpool`

## Vulnerability Details

The problem occurs, because the `fund` function in `fuels-ts/packages/account/src/account.ts` gets the needed ressources statelessly with the function `getResourcesToSpend` without taking into consideration already used `UTXOs`:

```ts
  async fund<T extends TransactionRequest>(request: T, params: EstimatedTxParams): Promise<T> {

    // [...]

    let missingQuantities: CoinQuantity[] = [];
    Object.entries(quantitiesDict).forEach(([assetId, { owned, required }]) => {
      if (owned.lt(required)) {
        missingQuantities.push({
          assetId,
          amount: required.sub(owned),
        });
      }
    });

    let needsToBeFunded = missingQuantities.length > 0;
    let fundingAttempts = 0;
    while (needsToBeFunded && fundingAttempts < MAX_FUNDING_ATTEMPTS) {
      const resources = await this.getResourcesToSpend(
        missingQuantities,
        cacheRequestInputsResourcesFromOwner(request.inputs, this.address)
      ); // @audit-issue here we do not exclude ids we already got and used for another transaction in the current block

      request.addResources(resources);

      // [...]
    }

    // [...]

    return request;
  }
```

## Impact Details

This issue will lead to unexpected SDK behaviour. Looking at the scenario in `Brief/Intro`, it could have the following impacts for users:

1. A transaction does not get included in the `txpool` / in a block
2. A previous transaction silently gets removed from the `txpool` and replaced with a new one

## Recommendation

I would recommend adding a buffer to the `Account` class, in which retrieved `resources` are saved. These can then be provided to `getResourcesToSpend` to be excluded from future queries but need to be removed from the buffer if their respective transaction fails to be included, in order to be able to use those `resources` again in such cases.

## Proof of concept

## Proof of Concept

The following PoC transfers 100 coins from `wallet2` to `wallet` after which `wallet2` has two `UTXOs` one with value `100` and one with a very high value (this is printed to the console). Afterwards, `wallet` will attempt transfering `80` coins back to `wallet2` twice in one block, each in a separate transaction. This should work perfectly fine as `wallet` has two `UTXOs` where each can cover the cost of each respective transaction. Now when running this one of the following will happen:

1. both transfers from `wallet` to `wallet2` get a different UTXO. This is the case if execution is successful and `wallet2` has `80` coins more than `wallet` in the end.
2. both transfers get the same UTXO. In this case the script will fail and throw an error as then both transactions will have the same hash

In order to execute this PoC, please deploy a local node with a blocktime of `5secs` as I wrote my PoC for that blocktime. Note that with a small change it will also work with other blocktimes. Then add the PoC to a file `poc_resources.ts` and compile it with `tsc poc_resources.ts`.\
Finally execute it with `node poc_resources.js`.

Since the choice which `UTXO` is taken as input is random, it might take a few tries to trigger the bug!

```ts
import { JsonAbi, Script, Provider, WalletUnlocked, Account, Predicate, Wallet, CoinQuantityLike, coinQuantityfy, EstimatedTxParams, BN, Coin, AbstractAddress, Address, Contract, ScriptTransactionRequest } from 'fuels';

const abi: JsonAbi = {
  'encoding': '1',
  'types': [
    {
      'typeId': 0,
      'type': '()',
      'components': [],
      'typeParameters': null
    }
  ],
  'functions': [
    {
      'inputs': [],
      'name': 'main',
      'output': {
        'name': '',
        'type': 0,
        'typeArguments': null
      },
      'attributes': null
    }
  ],
  'loggedTypes': [],
  'messagesTypes': [],
  'configurables': []
};

const FUEL_NETWORK_URL = 'http://127.0.0.1:4000/v1/graphql';

async function executeTransaction() {

  const provider = await Provider.create(FUEL_NETWORK_URL);
  
  const wallet: WalletUnlocked = Wallet.fromPrivateKey('0x37fa81c84ccd547c30c176b118d5cb892bdb113e8e80141f266519422ef9eefd', provider);
  const wallet2: WalletUnlocked = Wallet.fromPrivateKey('0xde97d8624a438121b86a1956544bd72ed68cd69f2c99555b08b1e8c51ffd511c', provider);
  const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));


  console.log("Balance wallet before: ", await wallet.getBalance());
  console.log("Balance wallet2 before: ", await wallet2.getBalance());

  wallet2.transfer(wallet.address, 100);

  await sleep(5500);


  await wallet.transfer(wallet2.address, 80);
  console.log('wallet -> wallet2');

  await wallet.transfer(wallet2.address, 80);
  console.log('wallet -> wallet2');

  console.log("Balance wallet after: ", await wallet.getBalance());
  console.log("Balance wallet2 after: ", await wallet2.getBalance());
};

executeTransaction().catch(console.error);
```
