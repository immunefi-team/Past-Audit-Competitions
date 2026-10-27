# 57203 sc medium revised malicious accesstoken creator can steal gas via mintstaticprice or mintdynamicprice&#x20;

* **Submitted on:** Oct 24th 2025 at 11:12:18 UTC by @blackgrease for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)
* **Report ID:** #57203
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol>
* **Impacts:** Theft of gas; Griefing (no profit motive required)

## Description

TO IMMUNEFI TRIAGERS

This section responds to a comment in report #57040 to clarify some points. The rest of the report remains unchanged.

{% stepper %}
{% step %}

### 1. Victim interaction and facilitator contract

The only contract the victim interacts with is the deployed `AccessToken`. The exploit leverages the `AccessToken` logic — the `AccessToken` contract is the facilitator of the attack. During normal execution the victim is not aware that an external call will be made to a malicious contract (external calls are made to the platform, referrer and creator).
{% endstep %}

{% step %}

### 2. Signed message does not include intended owner

To deploy an instance of `AccessToken`, the Signer address signs a message consisting of `NFT name, symbol, contractURI, feeNumerator`. This message does not include the intended owner/deployer of `AccessToken`. Upon deployment the deployer becomes the owner. Consequently:

* A user can provide NFT data to be signed and then call deploy using a different address.
* For comparison, `Factory::deployVestingWallet` includes the intended owner in the signed message (`function deployVestingWallet(address _owner, VestingWalletInfo calldata vestingWalletInfo, bytes calldata signature)`). That pattern prevents this class of problem for vesting wallets, but it is not applied to `AccessToken`.
* During `AccessToken::initialize` the owner is set as `_initializeOwner(_params.creator)` which is not controlled by the user submitting the signed data.
  {% endstep %}

{% step %}

### 3. Not simply user error

The user only interacts with Belong’s `AccessToken` code. The code assumes creators are benign; the user has no control over interactions with the creator address during payment flows. This is not purely a user mistake.
{% endstep %}

{% step %}

### 4. Functionality preserved but with hidden attack surface

`AccessToken` still functions as an access pass (NFT minting), but the hidden doorway in payment logic allows a malicious creator to execute arbitrary actions (unrelated to minting) that impose extra gas costs on the caller.
{% endstep %}
{% endstepper %}

For a visualization of the exploit flow, the reporter attached "AccessToken-TheftOfGasExploitFlow\.png".

Therefore, because the user has no control over the interaction with the creator address and because the AccessToken purpose can be abused to harm users, the issue is considered valid.

***

## Report: Description

The `AccessToken` allows users to pay for NFT mints using ETH through `mintStaticPrice` or `mintDynamicPrice`. When mints are made using ETH, these functions call an internal `_pay` function which sends payments to:

1. the platform,
2. the referrer, and
3. the creator

using the low-level `SafeTransferLib::safeTransferETH`. The low-level call forwards all available gas to recipients. A malicious creator (or a malicious referrer, if applicable) can use this to steal gas from users when they pay using ETH, by executing arbitrary code in their fallback/receive while receiving the payment. The caller pays for the extra gas.

Important things to note:

1. `Factory::produce` does not verify the caller. Once the Signer provides a valid signature for `NFT name, symbol, contractURI, feeNumerator`, `produce` can be called by any address.
2. `SafeTransferLib.sol` contains a more gas-restrictive function `forceSafeTransferETH` which uses a gas stipend, but it was not used.
3. If the referrer is not protocol-controlled, the same attack can extend to payments made to the referrer address if the AccessToken was created using a referral code.

### The associated Code for `_pay` called by `mintStaticPrice()` and `mintDynamicPrice()`

```solidity
function _pay(uint256 price, address expectedPayingToken) private returns (uint256 amount) {
    AccessTokenParameters memory _parameters = parameters;
    Factory.FactoryParameters memory factoryParameters = _parameters.factory.nftFactoryParameters();

    amount = expectedPayingToken == NATIVE_CURRENCY_ADDRESS ? msg.value : price;

    require(amount == price, IncorrectNativeCurrencyAmountSent(amount));

    uint256 fees = (amount * factoryParameters.platformCommission) / PLATFORM_COMISSION_DENOMINATOR;
    uint256 amountToCreator;
    unchecked {
        amountToCreator = amount - fees;
    }

    bytes32 referralCode = _parameters.referralCode;
    uint256 referralFees;
    address refferalCreator;
    if (referralCode != bytes32(0)) {
        referralFees = _parameters.factory.getReferralRate(_parameters.creator, referralCode, fees);
        if (referralFees > 0) {
            refferalCreator = _parameters.factory.getReferralCreator(referralCode);
            unchecked {
                fees -= referralFees;
            }
        }
    }

    if (expectedPayingToken == NATIVE_CURRENCY_ADDRESS) {
        if (fees > 0) {
            factoryParameters.platformAddress.safeTransferETH(fees);
        }
        if (referralFees > 0) {
            refferalCreator.safeTransferETH(referralFees); //@audit-issue: (Secondary case) if not protocol controlled, then can also abuse to steal gas
        }

        _parameters.creator.safeTransferETH(amountToCreator); //@audit-issue: (Primary case): possible for a malicious creator to use the low-level call to steal gas
    } else {
        //--snip--
```

### The `SafeTransferLib::safeTransferETH` logic

```solidity
function safeTransferETH(address to, uint256 amount) internal {
    /// @solidity memory-safe-assembly
    assembly {
        if iszero(call(gas(), to, amount, codesize(), 0x00, codesize(), 0x00)) { //@audit-issue: low-level call forwards all available gas enabling gas theft attacks
            mstore(0x00, 0xb12d13eb) // `ETHTransferFailed()`.
            revert(0x1c, 0x04)
        }
    }
}
```

***

## Impact

A malicious creator (or referrer) can abuse the payment logic to steal gas by executing additional arbitrary logic in their fallback/receive when receiving ETH. The unsuspecting buyer pays for the extra gas, leading to:

* Theft of gas (attacker causes buyer to spend more gas than necessary).
* Griefing (attacker causes damage without direct monetary profit).

***

## Mitigation

{% hint style="info" %}
Replace `safeTransferETH` calls with the more restrictive `forceSafeTransferETH` from the same `SafeTransferLib.sol`, which uses a gas stipend to limit forwarded gas and mitigate this class of gas-stealing attacks.
{% endhint %}

No additional functionality or protocol-level assumptions are suggested beyond using a gas-stipend-limited transfer to protocol-controlled addresses (and ensuring referral/creator addresses are validated or protocol-controlled where appropriate).

***

## Proof of Concept

A runnable Foundry PoC and a stack trace are available in the private gist:

<https://gist.github.com/blackgrease/840497c0aa53d6a459e03c5708b1108e>

The gist contains:

* a runnable Foundry PoC
* PoC-StackTrace\_TheftOfGasFromMaliciousCreator.txt

{% stepper %}
{% step %}

### PoC Steps — Setup and exploit flow

1. Deploy an `AccessToken` via `Factory::produce` from a contract that implements a malicious `receive`.
2. A buyer pays the required price and mints the NFT.
3. When the payment is made to the creator, the malicious `receive` is triggered and executes arbitrary logic, stealing gas.
4. Console logs show starting and remaining gas and the amount of gas consumed via the malicious interaction.
   {% endstep %}
   {% endstepper %}

(Note: referral-based attack is described in the report and can be applied analogously to any non-protocol-controlled referrer.)

***

If you need the PoC reproduced inline or want extracted test cases / specific locations in the repository mapped to suggested fixes, I can prepare a patch-style suggestion applying `forceSafeTransferETH` where appropriate and highlighting where referral/creator addresses should be protocol-controlled or validated.
