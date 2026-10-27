# 49954 sc insight raffle editprizes lacks logic to make prizes immutable once winner selection starts or users join breaking user trust&#x20;

Submitted on Jul 20th 2025 at 18:49:15 UTC by @blackgrease for Attackathon | Plume Network

* Report ID: #49954
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* Impacts: Contract fails to deliver promised returns, but doesn't lose value

## Summary

In a Raffle system, prizes should be immutable once winner selection begins or users join a prize pool. The `Raffle` contract does not prevent prize parameters from being changed while users are joining or while winner selection is in progress, which breaks user trust and the promise of fair, transparent prize distribution.

## Description

The `Raffle` contract allows prizes to be edited in all arguments while:

* users are joining a prize, and
* a winner is being selected.

Critically, while those activities are ongoing, a prize's `value` and `quantity` parameters can be changed. This undermines the project's stated transparency (e.g., "Winner selection is an admin-initiated process that can be repeated for each available prize slot. It is designed to be fair and transparent") because announced prizes can be modified after users have committed tickets.

Because there is no redeem/release mechanism for tickets, users are effectively locked into a prize whose parameters may later be reduced or otherwise changed. This breaks trust and can be considered unfair.

Functions associated with the issue

The function `Raffle::requestWinner()`

```solidity
//@audit: does not lock editPrize on specific prize allowing prize to change. Once winner selection starts, prizes should be immutable.
function requestWinner(uint256 prizeId) external onlyRole(ADMIN_ROLE) { 
	
	if (winnersDrawn[prizeId] >= prizes[prizeId].quantity) {
		revert AllWinnersDrawn();
	}
	
	if (prizeRanges[prizeId].length == 0) {
		revert EmptyTicketPool();	
	}
	
	require(prizes[prizeId].isActive, "Prize not available"); 		
	if (isWinnerRequestPending[prizeId]) {
		revert WinnerRequestPending(prizeId);	
	}
	
	isWinnerRequestPending[prizeId] = true;		  
	
	string memory callbackSig = "handleWinnerSelection(uint256,uint256[])";
	
	uint256 requestId = supraRouter.generateRequest(callbackSig, 1, 1, uint256(keccak256(abi.encodePacked(prizeId, block.timestamp))), msg.sender );
	
	pendingVRFRequests[requestId] = prizeId;
	emit WinnerRequested(prizeId, requestId);
}
```

The function `Raffle::editPrize()`

```solidity
//@audit: the `editPrize` has no logic prevent prizes being edited while winner section is active or users have joined.
function editPrize( uint256 prizeId, string calldata name, string calldata description, uint256 value, uint256 quantity ) external onlyRole(ADMIN_ROLE) prizeIsActive(prizeId) {
	
	// Update prize details without affecting tickets or active status
	Prize storage prize = prizes[prizeId];
	prize.name = name;
	prize.description = description;
	prize.value = value;
	prize.quantity = quantity;

	emit PrizeEdited(prizeId, name, description, value, quantity);

}
```

## Impact

The reported severity label states: "Contract fails to deliver promised returns, but doesn't lose value." The author rates the issue as Medium because the core problem is broken transparency and unfairness to users.

{% stepper %}
{% step %}

### 1. Protocol trust broken

The protocol can change announced prizes while users are participating, breaking transparency and user trust.
{% endstep %}

{% step %}

### 2. Users stuck with altered prizes

Users join pools for a specific reward expectation. If prize `value` or `quantity` is later reduced, users cannot redeem their tickets and are locked into a less favorable prize.
{% endstep %}

{% step %}

### 3. Unfairness

Mutable prizes combined with lack of ticket redemption create unfair outcomes: users might not have entered a pool had they known updated prize parameters.
{% endstep %}

{% step %}

### 4. Reputation damage

This behavior can damage Plume’s reputation due to perceived lack of full transparency and fairness.
{% endstep %}
{% endstepper %}

Likelihood: High — easily triggered during normal operations by an admin call to `editPrize`.

## Mitigation (recommended)

Enforce immutability for prize economic parameters (`value` and `quantity`) once users have joined a prize or once winner selection has started. Suggested changes per function:

* Raffle::spendRaffle()
  * When a user joins a prize pool, set a locked state for that prize so `editPrize` cannot change `value` or `quantity` for that prize.
* Raffle::requestWinner()
  * As an additional precaution, set the prize as locked when winner selection is initiated.
* Raffle::editPrize()
  * Check the prize locked state. If locked, only allow changes to `name` and `description`. Do not allow `value` or `quantity` changes when locked.
* updatePrizeEndTimestamp
  * Leave unchanged—allow admin to extend a prize's duration.

Once locked, the admin should only be able to update non-economic fields (name, description) and extend expiration times.

## Proof of Concept (walk-through)

{% stepper %}
{% step %}
An admin adds an attractive prize:

* addPrize("Annual Prize","Only the luckiest win big", 50000, 20)
  {% endstep %}

{% step %}
Prize is set to run for one month (assume Report #49868 is fixed).
{% endstep %}

{% step %}
Multiple users join this prize due to its high reward and many winners.
{% endstep %}

{% step %}
Admin calls requestWinner(prizeId) and begins winner selection.
{% endstep %}

{% step %}
Admin then calls editPrize(prizeId, ..., value = 10000, quantity = 5), reducing prize attractiveness.
{% endstep %}

{% step %}
Because editPrize has no locking logic, the changes succeed even though users have joined and winner selection started.
{% endstep %}

{% step %}
Users who bought tickets are now stuck: they cannot redeem tickets and the prize they joined is materially different.
{% endstep %}

{% step %}
Users lost their tickets and associated investment relative to the originally advertised prize parameters.
{% endstep %}
{% endstepper %}

## Notes

* The recommended fixes maintain admin flexibility (name/description/time extension) while protecting users by preventing economic parameter changes once the prize is active or winner selection has begun.
* No external links or code was changed from the original report.
