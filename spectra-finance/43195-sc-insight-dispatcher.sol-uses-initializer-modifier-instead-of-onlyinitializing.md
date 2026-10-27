# #43195 \[SC-Insight] \`Dispatcher.sol\` uses \`initializer\` modifier instead of \`onlyInitializing\`

**Submitted on Apr 3rd 2025 at 15:31:11 UTC by @Kyosi for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43195
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`Dispatcher` contract is a child contract of the `Router` contract and it uses `initializer` modifier instead of `onlyInitializing`. In the inheritance model, the contract `Router` has its own `initialize` function, which includes the `initializer` modifier and calls the `__Dispatcher_init` function of `Dispatcher` contract. The problem here is that both the parent contract (`Router`) and the child contract (`Dispatcher`) are using the `initializer` modifier, which limits initialization to only one call.

## Vulnerability Details

According to the Openzeppelin documentation <https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-initializer--> , the `onlyInitializing` modifier should be used to allow initialization in both the parent and child contracts (<https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-onlyInitializing-->). The `onlyInitializing` modifier ensures that when the `initialize` function is called, any contracts in the inheritance chain can still complete their own initialization. From the Openzeppelin docs: `A modifier that defines a protected initializer function that can be invoked at most once. In its scope, onlyInitializing functions can be used to initialize parent contracts.`

## Impact Details

The vulnerability causes an operational issue, preventing inheriting contracts from completing initialization. This could lead to a failure in the deployment of critical protocol contracts, affecting the overall system functionality.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L73-L79>

Same issue found in another protocol: <https://solodit.cyfrin.io/issues/m-5-masteramo-should-not-use-the-initializer-modifier-sherlock-axion-git>

## Mitigation

Replace `initializer` with `onlyInitializing` on `Dispatcher.sol`

```solidity
    function __Dispatcher_init(address _routerUtil, address _kyberRouter) internal onlyInitializing {
        if (_routerUtil == address(0)) {
            revert AddressError();
        }
        routerUtil = _routerUtil;
        kyberRouter = _kyberRouter;
    }
```

## Proof of Concept

## Proof of Concept

None
