# #42351 \[SC-Insight] Yeetback complex rewards system

**Submitted on Mar 23rd 2025 at 08:01:58 UTC by @Dimaranti for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42351
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**

## Description

## Brief/Intro

Yeetback has a reward system where for each round it is remembered how much is going to be paid to winners and how many wins the user has in this round. This also leads to the fact that the user needs to claim his rewards for each round separately\
What can be done instead is there could be a mapping `mapping(address => uint256) amountWon;` which accumulates all the amounts won by the user. Simplifying the contract logic and easing users who have won in multiple rounds to be able to claim their rewards at once.

## Vulnerability Details

Here are the 3 mappings currently responsible for rewards management:\
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeetback.sol#L38-L40\\>
And here is how they are updated first the winnings for the current round are calculated and set into the `amountToWinners` mapping.\
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeetback.sol#L77\\>
Then for each time the user is drawn by the rng for the current round the `amountOfWins` for the user is increased\
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeetback.sol#L85\\>
And finally when the user claims his reward he provides a `round`. And the `claimed` mapping for the given round and user is set to true.\
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeetback.sol#L129\\>
And he is being sent `amountToWinners[round] * amountOfWins[round][msg.sender]`\
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeetback.sol#L130>

Well this all can be refactored to a single `mapping(address => uint256) amountWon;` which accumulates all the rewards won by the user and when the user claims it is set back to 0. See PoC for suggested diff.

## Impact Details

## References

## Proof of Concept

## Proof of Concept

Suggested changes

```diff
@@ -35,9 +35,7 @@
     address private entropyProvider;
 
     //Mapping
-    mapping(uint256 => mapping(address => uint16)) public amountOfWins;
-    mapping(uint256 => mapping(address => bool)) public claimed;
-    mapping(uint256 => uint256) public amountToWinners;
+    mapping(address => uint256) amountWon;
 
     /// @notice Mapping of the pot value for each round
     /// @dev round => pot value
@@ -55,7 +53,7 @@
     event RandomNumberRequested(uint256 indexed sequenceNumber);
     event YeetbackAdded(uint256 indexed round, uint256 amount);
     event YeetbackWinner(uint256 indexed round, address indexed winner, uint256 amount, uint256 winningYeetIndex);
-    event Claimed(uint256 indexed round, address indexed user, uint256 amount);
+    event Claimed(address indexed user, uint256 amount);
 
     constructor(address _entropy, address _entropyProvider) Ownable(msg.sender) {
         require(_entropy != address(0), "Yeetback: Invalid entropy address");
@@ -74,7 +72,6 @@
         uint256 nrOfWinners = 10;
 
         uint256 winnings = potValue / nrOfWinners;
-        amountToWinners[round] = winnings;
 
         for (uint256 i; i < nrOfWinners; i++) {
             uint256 randomDataNumber = uint256(keccak256(abi.encodePacked(randomNumber, i)));
@@ -82,7 +79,7 @@
             address winnerAddress = yeetsInRound[round][winningYeetIndex];
 
             // Update amountToWinners and amountOfWins
-            amountOfWins[round][winnerAddress] += 1;
+            amountWon[winnerAddress] += winnings;
 
             emit YeetbackWinner(round, winnerAddress, winnings, winningYeetIndex);
         }
@@ -119,33 +116,21 @@
     }
 
     /// @notice Claims the winnings for the user
-    /// @param round The round number
-    function claim(uint256 round) public nonReentrant {
-        require(round != 0, "Yeetback: Invalid round");
-        uint16 nrOfWins = amountOfWins[round][msg.sender];
-        require(nrOfWins != 0, "Yeetback: No winnings to claim");
-        require(!claimed[round][msg.sender], "Yeetback: Already claimed");
-
-        claimed[round][msg.sender] = true;
-        uint256 amount = amountToWinners[round] * nrOfWins;
-        (bool success,) = payable(msg.sender).call{value: amount}("");
+    function claim() public nonReentrant {
+        uint256 amountWonByUser = amountWon[msg.sender];
+        require(amountWonByUser != 0, "Yeetback: No winnings to claim");
+
+        amountWon[msg.sender] = 0;
+        (bool success,) = payable(msg.sender).call{value: amountWonByUser}("");
         require(success, "Transfer failed.");
-        emit Claimed(round, msg.sender, amount);
+        emit Claimed(msg.sender, amountWonByUser);
     }
 
     /// @notice Returns the amount that the user can claim
-    /// @param round The round number
     /// @param user The user address
     /// @return The amount that the user can claim
-    function claimable(uint256 round, address user) public view returns (uint256) {
-        uint16 nrOfWins = amountOfWins[round][user];
-        uint256 amount = amountToWinners[round] * nrOfWins;
-        bool haveClaimed = claimed[round][user];
-        if (haveClaimed || amount == 0) {
-            return 0;
-        }
-
-        return amount;
+    function claimable(address user) public view returns (uint256) {
+        return amountWon[user];
     }
 
     /// @notice Returns the entropy fee, used in the dApp to show the user how much they need to pay


```
