# 57691 sc medium malicious referrer can permanently block eth payment flow

**Submitted on Oct 28th 2025 at 08:24:15 UTC by @blackgrease for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57691
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

**Affected Files:** `AccessToken.sol`, `Factory.sol` and `ReferralSystemV2.sol`

The `AccessToken` allows users to pay for NFT mints using ETH either through `mintStaticPrice` or `mintDynamicPrice`. When mints are made using ETH, the mentioned functions call the internal function, `AccessToken::_pay` which will send payments to:

1. the platform,
2. referrer and
3. creator

using a low-level call in `SafeTransferLib::safeTransferETH`. The low level correctly checks the success status of the call and reverts the transaction if the payment to any of the 3 addresses fails.

When deploying an `AccessToken` the protocol allows deployments to include a referral code. Anyone can create a referral code by calling `Factory::createReferralCode` as the code is a hash that includes their address. The code is linked to an address that will receive a small percentage when mint payments are made.

This code is included in the `Factory::produce` function arguments as `referralCode`. It is important to note, this code is not part of the data signed by the `Signer` address.

The `AccessToken` uses a "push-payment" flow. As a result of "pushing payments", a malicious referrer has the ability to permanently brick the ETH payment flow in `AccessToken` for **all users** by reverting. This renders the `AccessToken` contract permanently unable to receive mint payments via ETH and either have to:

* resort to only payments using ERC20 tokens (the referrer will still get payments)
* or, as an extreme measure, redeploy the `AccessToken` using a non-malicious referrer code; though this will require a name and symbol change.

This vector may also be triggered accidentally by a referrer creating a code from a contract that cannot receive ETH payments. Furthermore, once set, the code cannot be changed.

{% stepper %}
{% step %}

### Consider this scenario

* Malicious User creates referral code from address that does not accept ETH payments.
* Victim uses this code in deploying `AccessToken`.
* Any user transaction to mint an Access Token using ETH as the payment reverts at the referrer's payment.
* The `AccessToken` ETH payment path is permanently unavailable.
  {% endstep %}
  {% endstepper %}

### The Associated Code

Below is the logic for the functions that make this issue possible:

> **The referral code creation - `ReferralSystemV2::createReferralCode` - inherited by `Factory`**

```solidity
function createReferralCode() external returns (bytes32 hashedCode) {
        hashedCode = keccak256(abi.encodePacked(msg.sender, address(this), block.chainid));

        require(referrals[hashedCode].creator == address(0), ReferralCodeExists(msg.sender, hashedCode));

        referrals[hashedCode].creator = msg.sender;

        emit ReferralCodeCreated(msg.sender, hashedCode);
    }
```

> **The internal `AccessToken::_pay` function**

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
                refferalCreator.safeTransferETH(referralFees); //@audit-issue: a malicious referrer can block native payments path(or if code was accidentally created from address that cannot receive ETH payments)
            }

            _parameters.creator.safeTransferETH(amountToCreator);
        } else {
            expectedPayingToken.safeTransferFrom(msg.sender, address(this), amount);

            if (fees > 0) {
                expectedPayingToken.safeTransfer(factoryParameters.platformAddress, fees);
            }
            if (referralFees > 0) {
                expectedPayingToken.safeTransfer(refferalCreator, referralFees);
            }

            expectedPayingToken.safeTransfer(_parameters.creator, amountToCreator);
        }

        emit Paid(msg.sender, expectedPayingToken, amount);
    }
```

> **The `SafeTransferLib` ETH payment logic**

```solidity
function safeTransferETH(address to, uint256 amount) internal {
        /// @solidity memory-safe-assembly
        assembly {
            if iszero(call(gas(), to, amount, codesize(), 0x00, codesize(), 0x00)) {
                mstore(0x00, 0xb12d13eb) // `ETHTransferFailed()`.
                revert(0x1c, 0x04) //@audit: reverts where because `success == false`
            }
        }
    }
```

## Impact

The impact of this issue is griefing due to the ETH flow becoming permanently blocked. The block is permanent because once set, the referral code cannot be changed. The owner/creator will be limited to only mint payments via ERC20 tokens. In this case, the malicious referrer will still receive the payments.

This also damages the potential profits for the owner/creator as it removes the flexibility of using native payments which limits how users can pay — depending on demand.

Furthermore, the contract logic promises an ETH payment flow but by this flow becoming unavailable, this promise is unfulfilled.

## Mitigation

The recommended mitigation for this issue is to redesign payments to the referrer. The `platformAddress` can be considered safe as it is protocol owned. The `creator` address can also be seen as safe due to it making no sense to exploit this.

However, for the referrer payment, this should be a pull payment, where the referrer has to call the contract to claim their ETH payments. This fix will effectively prevent bricking the ETH payment flow.

## Link to Proof of Concept

<https://gist.github.com/blackgrease/36d6ac771dcdcf6ee617491fdb559023>

## Proof of Concept

As proof of this issue, I have created a runnable Foundry PoC — "<https://gist.github.com/blackgrease/36d6ac771dcdcf6ee617491fdb559023>" — that outlines the above scenario (in Description).

Run with:

```
forge test --mt testMaliciousReferrerBlocksETHPaymentFlow -vvv --via-ir
```

Additionally, in case of any issues running the PoC, the gist has a .txt Foundry Stack trace from the test - "PoC\_StackTrace-testMaliciousReferrerBlocksETHPaymentFlow\.txt" - which shows the revert on the referrer's payment.

{% stepper %}
{% step %}

### Foundry Setup — Step 1: Clone the repo

Clone the Github repo:

```bash
git clone https://github.com/belongnet/checkin-contracts.git
```

{% endstep %}

{% step %}

### Foundry Setup — Step 2: Install Foundry dependencies

1. Install OpenZeppelin upgradeable contracts (no commit):

```bash
forge install OpenZeppelin/openzeppelin-contracts-upgradeable@v5.4.0 --no-commit
```

2. Install OpenZeppelin contracts (no commit):

```bash
forge install OpenZeppelin/openzeppelin-contracts@v5.4.0 --no-commit
```

3. Install solady via npm:

```bash
npm install solady --force
```

{% endstep %}

{% step %}

### Foundry Setup — Step 3: Update Mappings in `foundry.toml`

Comment out the previous remappings block if present and replace with the new remappings.

Previous (example, commented out):

```toml
# remappings = [
#     "@ensdomains/=node_modules/@ensdomains/",
#     "@openzeppelin/=node_modules/@openzeppelin/",
#     "eth-gas-reporter/=node_modules/eth-gas-reporter/",
#     "hardhat-deploy/=node_modules/hardhat-deploy/",
#     "hardhat/=node_modules/hardhat/",
#     "operator-filter-registry/=node_modules/operator-filter-registry/",
#     "solady/=node_modules/solady/",
# ]
```

New remappings to add:

```toml
remappings = ["@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts","@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts"]
```

{% endstep %}
{% endstepper %}
