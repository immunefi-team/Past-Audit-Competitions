# 51070 sc low winning raffle ticket can be re used to maintain unfair advantage over other players in raffle&#x20;

**Submitted on Jul 30th 2025 at 21:38:10 UTC by @blackgrease for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51070
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

> Note: The severity and impact mismatch is intentional. While the stated impact is *Contract fails to deliver promised returns, but doesn't lose value*, the reporter believes the overall severity is High/Medium due to breaking fairness the Protocol claims and user related consequences.

### Summary

The `Raffle` contract — during winner selection — does not track which ticket index won. For multi-winner prizes, this allows the same raffle ticket to be used for multiple wins (even when different RNG values are provided), resulting in an unfair advantage for the holder of that ticket.

### Root cause

* The `handleWinnerSelection` function computes a winning ticket index as `(rng[0] % totalTickets[prizeId]) + 1` then binary-searches the ranges to map that index to a user.
* The selected winning index is not recorded as "used" or removed, so subsequent draws can pick the same index again.
* Different RNG values can still yield the same computed index; combined with the missing tracking, a single ticket can win multiple times.

### Example scenario

Prize with 4 winners, players and ticket ranges: Player1 \[Tickets 1–5] | Player2 \[6–15] | Player3 \[16–18] | Player4 \[19–25]

* First selection: RNG = 111 → index 5 → Player1 wins.
* Second selection: RNG = 12414298124 → index 5 → Player1 wins again (same ticket reused).

Many distinct RNG values map to the same index; because the winning index isn't marked used, the same ticket can be repeatedly selected.

### Affected code (excerpt)

The reporter highlighted the `handleWinnerSelection` function (unchanged from the original except for comments):

```solidity
//@audit-high/med: no tracking of winning Index allowing for winning raffle ticket to be reused.
function handleWinnerSelection(uint256 requestId, uint256[] memory rng) external onlyRole(SUPRA_ROLE) {
    uint256 prizeId = pendingVRFRequests[requestId];

    isWinnerRequestPending[prizeId] = false;
    delete pendingVRFRequests[requestId];

    if (!prizes[prizeId].isActive) {
        revert PrizeInactive();
    }

    if (winnersDrawn[prizeId] >= prizes[prizeId].quantity) {
        revert NoMoreWinners();
    }

    uint256 winningTicketIndex = (rng[0] % totalTickets[prizeId]) + 1;

    // Binary search for the winner
    Range[] storage ranges = prizeRanges[prizeId];
    address winnerAddress;

    if (ranges.length > 0) {
        uint256 lo = 0;
        uint256 hi = ranges.length - 1;
        while (lo < hi) {
            uint256 mid = (lo + hi) >> 1;
            if (winningTicketIndex <= ranges[mid].cumulativeEnd) {
                hi = mid;
            } else {
                lo = mid + 1;
            }
        }

        winnerAddress = ranges[lo].user;
    }

  // Store winner details
  prizeWinners[prizeId].push(Winner({
    winnerAddress: winnerAddress,
    winningTicketIndex: winningTicketIndex,
    drawnAt: block.timestamp,
    claimed: false}));


    winnersDrawn[prizeId]++;
    userWinCount[prizeId][winnerAddress]++;

  // Deactivate prize if all winners have been drawn
 if (winnersDrawn[prizeId] == prizes[prizeId].quantity) {
    prizes[prizeId].isActive = false; 
 }

 emit WinnerSelected(prizeId, winnerAddress, winningTicketIndex);

}
```

## Impact

(Reporter notes severity mismatch; they consider this High/Medium due to fairness concerns.)

This issue can lead to:

1. Perception that the raffle is rigged if the same player repeatedly wins with the same ticket.
2. Reduced participation in the Raffle and associated Spin game.
3. Decreased revenue from spin fees and lower ecosystem engagement.
4. Broken protocol claim of fairness.

The issue has a high likelihood because many RNG values map to the same index, making repeated wins plausible unless indices are tracked and excluded.

## Mitigation (recommended)

The reporter suggests tracking used winning indices and requesting multiple RNG values per request. The mitigation is provided below as a stepper (sequential steps recommended):

{% stepper %}
{% step %}

### Track used winning indexes

Add a mapping to record which ticket indexes have already been used per prize:

* mapping(uint256 prizeId => mapping(uint256 winningIndex => bool isUsed)) public usedWinningIndexes;
* Emit an event when indexes are exhausted: event IndexesExhausted(uint256 indexed prizeId);
  {% endstep %}

{% step %}

### Request multiple RNG values

When requesting randomness, ask Supra Router for multiple RNG values (e.g., 10):

* Use callback signature "handleWinnerSelection(uint256,uint256\[])".
* Example change: requestId = supraRouter.generateRequest(callbackSig, 10, 1, uint256(keccak256(abi.encodePacked(prizeId, block.timestamp))), msg.sender);
  {% endstep %}

{% step %}

### Use RNG values until an unused index is found

In the callback, iterate RNG values and compute an index for each until you find one that wasn't used yet:

* For each rng\[i]:
  * winningTicketIndex = (rng\[i] % totalTickets\[prizeId]) + 1;
  * if usedWinningIndexes\[prizeId]\[winningTicketIndex] == false:
    * mark it used and choose that index.
      {% endstep %}

{% step %}

### Handle exhaustion of RNG-derived indexes

If all supplied RNG values map to previously used indexes, emit IndexesExhausted(prizeId) and exit the callback so the admin can request fresh randomness.
{% endstep %}

{% step %}

### Continue existing winner selection flow

Once an unused winning index is found and marked used, proceed to map index → user via binary search and store the winner as before.
{% endstep %}
{% endstepper %}

Alternative mitigations noted by the reporter:

* Remove the selected raffle ticket from the ticket pool and rearrange ranges (more complex).
* Use a different index calculation approach (may still suffer collisions unless indices are tracked/excluded).

### Example suggested code diff (from reporter)

The reporter provided a proposed patch sketch (kept as-is):

```diff
//Declare mapping
+    mapping(uint256 prizeId => mapping(uint256 winningIndex=> bool isUsed)) public usedWinningIndexes;

//Declare Event
+   event IndexesExhausted(uint256 indexed prizeId);

//Request more RNG values in `requestWinner`
string memory callbackSig = "handleWinnerSelection(uint256,uint256[])";

-    uint256 requestId = supraRouter.generateRequest(callbackSig, 1, 1, uint256(keccak256(abi.encodePacked(prizeId, block.timestamp))), msg.sender );
+    uint256 requestId = supraRouter.generateRequest(callbackSig, 10, 1, uint256(keccak256(abi.encodePacked(prizeId, block.timestamp))), msg.sender ); //Request 10 RNG values

     pendingVRFRequests[requestId] = prizeId;
     emit WinnerRequested(prizeId, requestId);
}

    //Implementation
    function handleWinnerSelection(uint256 requestId, uint256[] memory rng) external onlyRole(SUPRA_ROLE) {

        uint256 prizeId = pendingVRFRequests[requestId];

        isWinnerRequestPending[prizeId] = false;
        delete pendingVRFRequests[requestId];

        if (!prizes[prizeId].isActive) {
            revert PrizeInactive();
        }

        if (winnersDrawn[prizeId] >= prizes[prizeId].quantity) {
            revert NoMoreWinners();
        }

        
-        uint256 winningTicketIndex = (rng[0] % totalTickets[prizeId]) + 1;

+       uint256 winningTicketIndex;

+        for(uint256 i; i < rng.length;i++){
+            //Calculating the winning index;
+            winningTicketIndex = (rng[i] % totalTickets[prizeId]) + 1; 
+            bool indexUsed = usedWinningIndexes[prizeId][winningTicketIndex];
+
+            if(!indexUsed){
+                usedWinningIndexes[prizeId][winningTicketIndex] = true; //Update the mapping to mark as true
+                break; //The winning index is not used therefore no need to continue the loop. Can exit and use in following logic
+            }
+
+            if(i == rng.length - 1){
+                emit IndexesExhausted(prizeId);
+                return; //Exit Early if all RNG values give previously used indexes. No need to continue below logic. New Values need to be requested
+                }
+            }


        // Binary search for the winner ()
        Range[] storage ranges = prizeRanges[prizeId];
        address winnerAddress;
        
        //---snip--- continue existing logic
```

## Proof of Concept

<details>

<summary>Proof of Concept (Foundry test, private Gist)</summary>

The reporter provided a runnable Foundry PoC demonstrating:

* Two very different RNG values producing the same winning index.
* Because the winning index isn't tracked, the same raffle ticket wins twice.
* Private Gist link: <https://gist.github.com/blackgrease/9052e25ef9335cd9392ce317e1543cf1>

Run with: forge test --mt testWinningIndexIsNotRemoved --via-ir -vvvv

Walk-through (from reporter):

1. Admin creates a prize with 4 winners.
2. 6 players join with different ticket amounts.
3. Supra Router returns RNG = 111 → index 5 → Player1 wins.
4. Later Supra Router returns RNG = 981247701242982890712535812410452300985578 → index 5 again → same ticket wins again.
5. Test asserts both draws returned the same winning index and winner.

</details>

## Links

* Affected file: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* PoC gist: <https://gist.github.com/blackgrease/9052e25ef9335cd9392ce317e1543cf1>

***

If you want, I can:

* Draft a minimal, ready-to-review patch implementing the mapping + RNG-array loop and unit tests; or
* Propose an alternative approach that removes a winning ticket from the pool (with pros/cons and complexity assessment).
