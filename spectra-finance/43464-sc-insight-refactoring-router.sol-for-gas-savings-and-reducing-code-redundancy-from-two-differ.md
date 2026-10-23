# #43464 \[SC-Insight] Refactoring \`Router.sol\` for gas savings and reducing code redundancy from two different \`Router::execute()\` which can result in undesirable outcomes for potentially delayed tra...

**Submitted on Apr 6th 2025 at 18:15:49 UTC by @blackgrease for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43464
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**

## Description

## Description

The `Dispatcher.sol` is an abstract contract that is inherited by the `Router.sol`. The `Dispatcher.sol` has multiple functions but the most notable for this report is the `_dispatch()`. This function is called by `Router::execute` which "executes encoded commands along with provided inputs"

> The Code Issue

The `Router.sol` defines two execute() functions.

1. The first execute function takes 3 arguments, is external and has the modifier `checkDeadline` which creates a time restriction for when a transaction can be executed.-> `function execute( bytes calldata _commands, bytes[] calldata _inputs, uint256 _deadline) external payable override checkDeadline(_deadline) {.... }` This first `execute` will then call the second \`execute function
2. The second execute function takes 2 arguments, is public and DOES NOT have the modifier `checkDeadline` -> `function execute(bytes calldata _commands,bytes[] calldata _inputs ) public payable override whenNotPaused {..}`

With the current code setup,

* the first `execute()#L130` is more restrictive and efficient in making sure that transactions can be executed on time.
* The second `execute` is less restrictive and can cause undesirable outcomes, especially with time sensitive transactions and depending on which blockchain is used, may be vulnerable to variations of MEV attacks.

The `Dispatcher::_dispatch#L141` function carries out a variety of functionality such as transfers, flashloans, curve swaps; some transactions where time will affect the outcome.

> Addressing functionality concerns

While The protocol provides two execute functions, one intended for time-sensitive transactions and the other for general use. If a user needs to execute a time-sensitive transaction, they should use the execute function that includes the `checkDeadline` modifier. Notably, the first execute function calls the second `execute` internally, which introduces a degree of code redundancy.

Secondly, there is the potential for users to accidentally call the wrong/unintended `execute` function which can potentially produce undesirable outcomes if the transaction is delayed.

This creates an opportunity to refactor the contract to

* avoid code redundancy
* improve efficiency in terms of gas savings
* reduce chance for user to call wrong `execute` function on a time-sensitive transaction.

> Code Optimizations for Gas Savings and Reduced Code Redundancy

The below code optimizations are aimed at reducing the possible chance that a user may accidentally call the wrong `execute` and cause undesirable outcomes. Furthermore, the refactoring aims at saving gas during the deployment process.

The only `execute` function is now: `function execute( bytes calldata _commands, bytes[] calldata _inputs, uint256 _deadline ) public payable checkDeadline(_deadline) whenNotPaused {...}`

1. The user will simply assign `deadline` as 0 to dismiss the transaction as not time sensitive
2. `checkDeadline` modifier: now checks if the deadline is 0 - which is default value for unintialized uint256's. If deadline is not 0, it runs a the deadline check.
3. `whenNotPaused` modifier: is now directly in the only existing`execute` function
4. The logic of the second `execute#L141` is merged with the first `execute#130` function to
   * reduce chance of wrong `execute` being called
   * reduce code redundancy. There is no need to call 2 `execute` functions along the same path to call `_dispatch`
5. The `IRouter` import is removed and functions are declared directly within contract:
   * which also means the `override` modifier on most functions is now removed.
6. Within the `onFlashLoan` it used the `execute` which does not use the deadline. To factor this in, a `deadline` variable is declared with explicit value of 0 which is now passed to the `execute` function as an argument to maintain functionality.
7. The `execute` has a visibility of public in order to allow the `onFlashLoan` function to still call it.

Note: The functionality of the contract remains the same and has not been altered.

For Readability purposes, the refactored contract is located in the private gist link: <https://gist.github.com/blackgrease/172d4afab14df5dc8b7e149e7553bd10>

> Impact

In the case where the second `execute#L141` is will be executed - either by lack of knowledge or forgetting the deadline parameter - and where potentially transaction delays happen that disrupt time-sensitive actions, the contract "fails to deliver promised returns, but doesn't lose value. "

For users, this results in

* variety of outcomes depending on the nature of the transaction
* can create a environment for MEV to successfully attacks to occur

Therefore by refactoring the `Router.sol` there are

1. Reduced chance for user errors when using the `execute` function while still maintaining contract functionality
2. Reduced gas costs during deployment.

## Link to Proof of Concept

<https://gist.github.com/blackgrease/172d4afab14df5dc8b7e149e7553bd10>

## Proof of Concept

> Proof-of-Concept

The following test suite is in a private gist link - <https://gist.github.com/blackgrease/172d4afab14df5dc8b7e149e7553bd10> - and simply

1. Deploys the original Router contract and calculates the amount of gas used
2. Deploys the refactored contract and calculates the amount of gas used.
3. An assert makes sure that the refactored Router.sol uses less gas than the Original

Run with `forge test --mt testDeploymentGasSavings -vvv`
