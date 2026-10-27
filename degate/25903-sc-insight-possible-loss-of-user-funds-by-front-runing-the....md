# 25903 - \[SC - Insight] Possible loss of user funds by front-runing the...

Submitted on Nov 20th 2023 at 23:39:49 UTC by @infosec\_us\_team for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #25903

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Permanent freezing of funds in the Default Deposit Contract that is less than 2,500,000 USD.
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Bug description

The current deployment process for the `ExchangeV3` and `DefaultDepositContract` smart contracts is as follows:

**Step 1- Deploy the implementation** (transaction 1)

> Example tx:<https://etherscan.io/tx/0xb5b283c538741a1513b2f100cfd3ab4058560712e4ec0752fd587e30e23edddd>

**Step 2- Deploy proxy** (transaction 2)

> Example tx: <https://etherscan.io/tx/0x989a6e990ebe16ce29a4d2e649c7ed61b05edf6870d84872109c141ef0db5f98>

**Step 3- Make the proxy point to the implementation** (transaction 3)

> Example tx: <https://etherscan.io/tx/0x618d6d6bdaa4be3257aa4c695f9c10806e261f0e9759fc3133a5798fed43c062>

**Step 4- Initialize the implementation** (transaction 4)

> Example tx: <https://etherscan.io/tx/0x3c5629d35e75eb6b1e3e17e73ea16f3638c46a120a1f31bb8988f9db89bf483a>

They are coded in a deployment script (<https://github.com/degatedev/protocols/blob/degate\\_mainnet/packages/loopring\\_v3/migrations/8\\_deploy\\_exchange\\_v3.js#L121>). These transactions happen automatically one after the other.

When deploying and initializing, the current workflow doesn't check if the previous transaction succeeds before continuing with the next one.

The `initialize(address _exchange)` function inside `DefaultDepositContract` allows to set the exchange address and the `owner` of the smart contract.

In the best case scenario the devs will have to re-deploy all the smart contracts once again, if the `initialize(address _exchange)` is front-run with an attacker-controlled `_exchange` address.

In the worst case scenario, because the deployment script does not check if the previous transaction succeeded, the address of this maliciously initialized `DefaultDepositContract` may end up being used in the front-end of the site and users will directly transfer funds to it using the ERCO20 interface of the tokens (also known in DeGate as *realizing an unconfirmed standard deposit*), and their funds will be lock forever inside that smart contract.

## Permanent fix

There are 3 ways of fixing this and we consider it is so easy to fix that doing so should not be overlooked:

* **Run Step 1 to Step 4 atomically**.

> Within the same transaction, do everything. Deploy, upgrade, and initialize the smart contracts.

* **Alternatively, in the script check if the previous transaction succeed before going to the next one. Stop the deployment if something failed**

> Not recommended, there's really not point on having to re-deploy everything if the initialization gets front-run.

* **Modify the `initialize(...)` function of `DefaultDepositContract` and `ExchangeV3` to only allow the deployer to initialize them**

## Proof of concept

Here's the deployment script proving that the result from the `initialize(...)` transaction is not checked:

```javascript
      const depositContractImp = await DefaultDepositContract.deployed();
      await depositContractProxy.upgradeTo(depositContractImp.address);
      const depositContract = await DefaultDepositContract.at(depositContractProxy.address);

      // -------------------------
      // Here we get front-run
      // -------------------------

      // Here we try to initialize but it reverts because we were front-run
      await depositContract.initialize(exchangeV3.address);
      // Here we set the deposit contract in the exchangeV3 to compromised instance of DefaultDepositContract 
      await exchangeV3.setDepositContract(depositContract.address);
```

> <https://github.com/degatedev/protocols/blob/degate\\_mainnet/packages/loopring\\_v3/migrations/8\\_deploy\\_exchange\\_v3.js#L142-L147>
