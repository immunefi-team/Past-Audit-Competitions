# 57348 sc insight incorrectly returned values and emitted data on staking emergency functionality

**Submitted on Oct 25th 2025 at 12:16:31 UTC by @blackgrease for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57348
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

**Affected Files:** `Staking.sol`

The `Staking` contract has emergency withdrawal functions — `emergencyWithdraw` and `emergencyRedeem` — that allow a user to bypass the minimum staking period in order to remove their locked stakes.

However, the returned value in the `emergencyWithdraw` and `emergencyRedeem` functions is incorrect because it misrepresents the amount of assets/shares actually withdrawn/redeemed. The returned value does not factor in the penalty fee that is charged to users on locked tokens.

Scenario:

{% stepper %}
{% step %}

### Scenario example

1. User stakes 1000 LONG while the minimum staking period is 5 days.
2. They initiate an emergency withdrawal of the full amount. The penalty is 10% meaning they will receive only 900 LONG.
3. `emergencyWithdraw` and `emergencyRedeem` incorrectly return that the user has received 1000 LONG while in practice they have been transferred 900 LONG.
   {% endstep %}
   {% endstepper %}

Furthermore, this incorrect data is emitted in the related events — `EmergencyWithdraw` and `Withdraw` — propagating the misrepresented data.

Note: The natspec includes a note: "// @return assets /shares Assets calculated from `shares`/`assets` before penalty." In practice, returning the actual payout after penalty would be more beneficial for users and for indexers/analytics.

This is a logic issue where an external function meant to return a value does not return the correct thing. Importantly, users are transferred the correct amount of tokens; no value is lost.

### The problematic code

Below are the relevant functions highlighting where the issue exists:

```solidity
function emergencyWithdraw(uint256 assets, address to, address _owner) external returns (uint256 shares) { //@audit-issue: incorrect value returned
    if (assets > maxWithdraw(_owner)) revert WithdrawMoreThanMax(); 
    shares = previewWithdraw(assets); //@audit-issue: this does not factor in the penalty
    _emergencyWithdraw(msg.sender, to, _owner, assets, shares);
}

function emergencyRedeem(uint256 shares, address to, address _owner) external returns (uint256 assets) {//@audit-issue: incorrect value returned
    if (shares > maxRedeem(_owner)) revert RedeemMoreThanMax();
    assets = previewRedeem(shares); //@audit-issue: this does not factor in the penalty
    _emergencyWithdraw(msg.sender, to, _owner, assets, shares);
}

function _emergencyWithdraw(address by, address to, address _owner, uint256 assets, uint256 shares) internal {
    require(shares > 0, SharesEqZero());

    uint256 penalty = FixedPointMathLib.fullMulDiv(assets, penaltyPercentage, SCALING_FACTOR);
    uint256 payout;
    unchecked {
        payout = assets - penalty; //@audit-issue: this is the correct value that should be returned to the external functions
    }

    if (by != _owner) _spendAllowance(_owner, by, shares);

    _removeAnySharesFor(_owner, shares);
    _burn(_owner, shares);

    LONG.safeTransfer(to, payout);
    LONG.safeTransfer(treasury, penalty);

    emit EmergencyWithdraw(by, to, _owner, assets, shares); //@audit-issue: emits the wrong data. Should emit payout for both assets and shares (as they share the same decimals) -> incorrect information
    
    // also emit standard ERC4626 Withdraw for indexers/analytics
    emit Withdraw(by, to, _owner, assets, shares);
}
```

## Impact

This is a low impact issue because the external functions `emergencyWithdraw` and `emergencyRedeem` do not return the correct values, and the returned value is only provided to callers (it is not consumed internally). However, indexers and analytics that rely on returned values or emitted events will see incorrect data, misrepresenting the actual payout amounts. The user receives the correct token amount; no funds are lost.

## Mitigation

Suggested fix: return the actual payout (after penalty) from the external functions and from the internal `_emergencyWithdraw`, and emit the correct values in `EmergencyWithdraw`.

A tested fix (applies small API change: the external functions return `payout` instead of pre-penalty assets/shares):

```diff
//Fix in `emergencyWithdraw` function
-    function emergencyWithdraw(uint256 assets, address to, address _owner) external returns (uint256 shares) { //@audit-issue: incorrect value returned
+    function emergencyWithdraw(uint256 assets, address to, address _owner) external returns (uint256 payout) { //@audit-fix
   
        if (assets > maxWithdraw(_owner)) revert WithdrawMoreThanMax(); 
-        shares = previewWithdraw(assets); //@audit-issue: this does not factor in the penalty
+        uint256 shares = previewWithdraw(assets); //@audit-fix
-        _emergencyWithdraw(msg.sender, to, _owner, assets, shares);
+        payout = _emergencyWithdraw(msg.sender, to, _owner, assets, shares); //@audit-fix
    }


//Fix in `emergencyRedeem` function
-    function emergencyRedeem(uint256 shares, address to, address _owner) external returns (uint256 assets) {//@audit-issue: incorrect value returned
+    function emergencyRedeem(uint256 shares, address to, address _owner) external returns (uint256 payout) {//@audit-fix
   
        if (shares > maxRedeem(_owner)) revert RedeemMoreThanMax();
-        assets = previewRedeem(shares); //@audit-issue: this does not factor in the penalty
+        uint256 assets = previewRedeem(shares);
-        _emergencyWithdraw(msg.sender, to, _owner, assets, shares);
+        payout = _emergencyWithdraw(msg.sender, to, _owner, assets, shares);
    }


//Fix in the internal `_emergencyWithdraw` function
+    function _emergencyWithdraw(address by, address to, address _owner, uint256 assets, uint256 shares) internal returns(uint256 payout){
        require(shares > 0, SharesEqZero());

        uint256 penalty = FixedPointMathLib.fullMulDiv(assets, penaltyPercentage, SCALING_FACTOR);
        // uint256 payout;
        unchecked {
            payout = assets - penalty; 
        }

        if (by != _owner) _spendAllowance(_owner, by, shares); //@audit-info: protects against other users stealing funds

        _removeAnySharesFor(_owner, shares);
        _burn(_owner, shares);

        LONG.safeTransfer(to, payout);
        LONG.safeTransfer(treasury, penalty);

-        emit EmergencyWithdraw(by, to, _owner, assets, shares);
+        emit EmergencyWithdraw(by, to, _owner, payout, payout); //@audit:fix
        // also emit standard ERC4626 Withdraw for indexers/analytics
        emit Withdraw(by, to, _owner, assets, shares); //@audit-note: this can remain the same or remove or change data as it may be conflicting with an actual withdraw operation where there are no penalties
    }
```

## Link to Proof of Concept

<https://gist.github.com/blackgrease/fe2ecf8ec4a0e9793ea829dfd9247648>

## Proof of Concept

A Foundry PoC outlines the incorrect data returned and emitted and what data should be returned. It demonstrates that the returned value does not correctly represent the assets that the user receives.

* Run with: `forge test --mt testWrongDataReturnedAndEmittedOnEmergencyWithdraw -vvv --via-ir`

Logs from the PoC:

```
[PASS] testWrongDataReturnedAndEmittedOnEmergencyWithdraw() (gas: 222213)
Logs:
  --PoC incorrectly emitted and returned data---
  Amount of shares/assets the withdrawal is SUPPOSED to return:  900000000000000000000
  Amount of shares/assets that is incorrectly returned:  1000000000000000000000
  Amount of shares/assets that are incorrectly emitted in `EmergencyWithdraw`/`Withdraw` events:  1000000000000000000000
  Note: user does not loose any value as they correctly receive the right amount
```

Foundry setup (how the author ran the PoC):

{% stepper %}
{% step %}

### 1. Clone the repo

git clone <https://github.com/belongnet/checkin-contracts.git>
{% endstep %}

{% step %}

### 2. Install Foundry dependencies

1. forge install OpenZeppelin/openzeppelin-contracts-upgradeable\@v5.4.0 --no-commit
2. forge install OpenZeppelin/openzeppelin-contracts\@v5.4.0 --no-commit
3. npm install solady --force
   {% endstep %}

{% step %}

### 3. Update remappings in foundry.toml

Replace previous remappings (commented out in the PoC) with:

```toml
remappings = ["@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts","@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts"]
```

{% endstep %}
{% endstepper %}
