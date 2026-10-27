# 49868 sc insight raffle sol does not enforce prize endtimestamp allowing user and admin interactions with expired prizes

**Submitted on Jul 20th 2025 at 03:51:08 UTC by @blackgrease for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49868
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### The severity and impact mismatch is intentional. While the stated impact is *Contract fails to deliver promised returns, but doesn't lose value*, I believe the overall severity is High/Medium in relation to failed functionality causing disparity between the protocols expectation and the actual contract execution.

## Summary

The `Raffle` allows an admin to update the ending timestamp of a created prize. However, there are no checks in place enforcing that user raffle tickets cannot be spent on expired prizes. Furthermore, expired prizes can be modified by admins. This results in incorrect interactions involving users and admins with prizes that are supposed to be expired.

## Details

The `Raffle` contract by default sets all created prizes to have an `endTimestamp` of 0 in `addPrize`. This means prizes will continue indefinitely unless updated.

```solidity
function addPrize( string calldata name, string calldata description, uint256 value,uint256 quantity) external onlyRole(ADMIN_ROLE) {
    uint256 prizeId = nextPrizeId++;
    prizeIds.push(prizeId);
    
    require(bytes(prizes[prizeId].name).length == 0, "Prize ID already in use");
    require(quantity > 0, "Quantity must be greater than 0");
    
    prizes[prizeId] = Prize({
            name: name,
            description: description,
            value: value,
            endTimestamp: 0, /@audit: by default, prizes have no expiration date.
            isActive: true,
            winner: address(0), // deprecated
            winnerIndex: 0, // deprecated
            claimed: false, // deprecated
            quantity: quantity
            }); 
    
    emit PrizeAdded(prizeId, name);
}
```

An admin can update a prize's `endTimestamp` using:

```solidity
// Timestamp update for prizes
function updatePrizeEndTimestamp(uint256 prizeId, uint256 endTimestamp) external onlyRole(ADMIN_ROLE) prizeIsActive(prizeId) {
    prizes[prizeId].endTimestamp = endTimestamp;
}
```

The `Raffle` contract does not check if a prize has expired; therefore interactions with technically expired prizes are allowed. A non-exhaustive list of problematic interactions:

* Users: spending raffle tickets on expired prizes.
* Admin:
  * Can edit an expired prize.
  * Can request a winner on a technically expired prize.
  * Can remove an expired prize (not a security loss but wasteful).

The `prizeIsActive` modifier only checks `isActive` and does not consider `endTimestamp`:

```solidity
modifier prizeIsActive(uint256 prizeId) {
    require(prizes[prizeId].isActive, "Prize not available");
    _;
}
```

While the contract runs without reverting, it functions incorrectly relative to the intended semantics: when a prize has an expiration set, interactions should be prevented after that time.

## Impact

The stated impact in the original report is: Contract fails to deliver promised returns, but doesn't lose value. The author argues the functional mismatch (admin expects a prize to be expired; users can still participate) elevates the effective severity to High/Medium because:

* Admins may set an expiration and stop interacting with a prize off-chain, while users can still spend tickets on it unaware that no draws will be requested.
* Any prize with a non-zero `endTimestamp` that is in the past will still accept interactions unless actively deactivated.

Likelihood: High for any time-limited prize that is left with `isActive == true` but with `endTimestamp` in the past.

Resulting incorrect behaviors include:

* Users spending raffle tickets on expired prizes.
* Admins editing or requesting winners for expired prizes.

## Mitigation

Recommended global fix: incorporate expiration checks into the `prizeIsActive` modifier so all functions using it automatically inherit expiration enforcement.

Suggested behavior:

* If `endTimestamp == 0` → prize does not expire, skip expiration check.
* If `endTimestamp != 0` and `block.timestamp > endTimestamp` → mark prize inactive and revert.

Proposed modifier change:

```diff
modifier prizeIsActive(uint256 prizeId) {
+    if(prizes[prizeId].endTimestamp != 0){ // If the prize is not supposed to run forever, check expiration.
+        if(block.timestamp > prizes[prizeId].endTimestamp){
+            // Automatically set the prize inactive globally
+            prizes[prizeId].isActive = false;
+            // Revert
+            revert("Prize has expired");
+        }
+    }
-    require(prizes[prizeId].isActive, "Prize not available");
+    // No checks if prize is set to run indefinitely
+    require(prizes[prizeId].isActive, "Prize not available");
    _;
}
```

Additionally, make `requestWinner` use this modifier instead of checking `isActive` inline:

```diff
-  function requestWinner(uint256 prizeId) external onlyRole(ADMIN_ROLE) {
+   function requestWinner(uint256 prizeId) external onlyRole(ADMIN_ROLE) prizeIsActive(prizeId) { 

    if (winnersDrawn[prizeId] >= prizes[prizeId].quantity) {
        revert AllWinnersDrawn();
    }
    
    if (prizeRanges[prizeId].length == 0) {
        revert EmptyTicketPool();
    }
    
-   require(prizes[prizeId].isActive, "Prize not available"); 
    
    if (isWinnerRequestPending[prizeId]) {
        revert WinnerRequestPending(prizeId);
    }
    
    isWinnerRequestPending[prizeId] = true;
        
    string memory callbackSig = "handleWinnerSelection(uint256,uint256[])";
    uint256 requestId = supraRouter.generateRequest(
    callbackSig, 1, 1, uint256(keccak256(abi.encodePacked(prizeId, block.timestamp))), msg.sender);

    pendingVRFRequests[requestId] = prizeId;
    emit WinnerRequested(prizeId, requestId);

}
```

This change ensures:

* Admins cannot edit/remove/request winners for expired prizes.
* Users cannot spend tickets on expired prizes.
* Any call using `prizeIsActive` will automatically enforce expiration.

## Proof of Concept

Private gist with a runnable Foundry test and walkthrough: <https://gist.github.com/blackgrease/40569a7feced0ab1acc8785aa1fcaf8d>

Run with:

```
forge test --mt testPrizeEndtimestampIsIgnored --via-ir -vvv
```

There is also a screenshot of the Foundry test stack trace included in the original report.

{% stepper %}
{% step %}

### Walk-through — create prize

1. An Admin creates a prize by calling `Raffle::addPrize`.
   {% endstep %}

{% step %}

### Walk-through — set expiration

2. This prize is updated to end after 30 days by calling `Raffle::updatePrizeEndTimestamp`.
   {% endstep %}

{% step %}

### Walk-through — confirm update

3. Test confirms the prize is active and correctly updated using `Raffle::getPrizeDetails`.
   {% endstep %}

{% step %}

### Walk-through — advance time past expiration

4. The expiration date for the prize is advanced past 30 days (60 days since created in test).
   {% endstep %}

{% step %}

### Walk-through — show admin can edit expired prize

5. Confirms an admin can still edit a prize that has "expired".
   {% endstep %}

{% step %}

### Walk-through — show user spending tickets on expired prize

6. Confirms a user can spend their raffle tickets on "expired" prizes.
   {% endstep %}
   {% endstepper %}

## Link to Proof of Concept

<https://gist.github.com/blackgrease/40569a7feced0ab1acc8785aa1fcaf8d>
