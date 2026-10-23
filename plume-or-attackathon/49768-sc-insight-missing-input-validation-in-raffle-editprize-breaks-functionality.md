# 49768 sc insight missing input validation in raffle editprize breaks functionality

**Submitted on Jul 19th 2025 at 09:43:06 UTC by @blackgrease for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49768
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Summary

During prize creation, prizes can exist with the same name. `Raffle::addPrize` validates `quantity` input while `Raffle::editPrize` does not, allowing an admin to edit a prize to have 0 winners. Both functions also do not validate `name` (allowing duplicate prize names) or `value` (allowing 0-value prizes).

## Description

Issue #1

* `addPrize` does not validate the `name` parameter, allowing multiple prizes with the same name which can confuse users.

Issue #2

* `editPrize` does not validate `quantity`, `value`, or `name`. `quantity` must be > 0 per intended design; omitting this validation allows a prize to be set to 0 winners.
* `addPrize` validates `quantity`, but `editPrize` does not — inconsistent validation leads to incorrect states after edits.

Exploitation

* Accidental misconfiguration by an admin may set `quantity` to 0 causing users to lose raffle tickets for that prize (no winners will be selected).
* A malicious admin could create a valid prize and later edit it to 0 winners or duplicate names after users have joined.

High-level issue path: Admin creates prize -> Users join prize -> Admin edits prize to 0 winners -> `requestWinner` fails due to checks -> users wait indefinitely.

## Affected functions

Raffle::addPrize()

```solidity
function addPrize(string calldata name, string calldata description, uint256 value, uint256 quantity ) external onlyRole(ADMIN_ROLE) {
	
	uint256 prizeId = nextPrizeId++;	
	prizeIds.push(prizeId);

	require(bytes(prizes[prizeId].name).length == 0, "Prize ID already in use");
	require(quantity > 0, "Quantity must be greater than 0");
	
	//@audit: no checks for duplicate name or value. Quantity is correctly validated
	
	prizes[prizeId] = Prize({	name: name,
		description: description,
		value: value, 
		endTimestamp: 0,
		isActive: true,
		winner: address(0), // deprecated
		winnerIndex: 0, // deprecated
		claimed: false, // deprecated
		quantity: quantity
	});
	emit PrizeAdded(prizeId, name);

}
```

Raffle::editPrize()

```solidity
function editPrize( uint256 prizeId, string calldata name, string calldata description, uint256 value, uint256 quantity ) external onlyRole(ADMIN_ROLE) prizeIsActive(prizeId) {

	// Update prize details without affecting tickets or active status
		
		//@audit: there are no checks here. Differs from pattern in `addPrize`. Does not check quantity, value or name.
		
		Prize storage prize = prizes[prizeId];
		prize.name = name;
		prize.description = description;
		prize.value = value;
		prize.quantity = quantity;
		emit PrizeEdited(prizeId, name, description, value, quantity);
	
}
```

## Impact

* Logic error resulting in:
  * Prizes with `quantity = 0` meaning no winners can be drawn and users who joined may effectively lose their raffle tickets.
  * Duplicate prize names causing user confusion.
* `requestWinner` will incorrectly revert with `AllWinnersDrawn()` when `quantity` is 0 (since `winnersDrawn[prizeId] >= prizes[prizeId].quantity`), causing admin to assume prize completed while users wait indefinitely.
* Low severity overall since no funds are directly lost, but user experience and promised returns are broken. A malicious or mistaken admin can cause harm.

## Mitigation (recommended)

* Enforce the same checks in `editPrize` as in `addPrize`.
* Validate `value > 0` in both functions.
* Prevent duplicate prize names (e.g., mapping of name hash to existence), with appropriate handling for edits (allow editing the same prize name).
* Consider checking that a prize is active in `requestWinner` before proceeding.

Suggested validation additions are shown below.

### Improved Validation in `Raffle::addPrize`

```diff
+   mapping(bytes32 => bool) private prizeNameExists; //declare mapping

function addPrize(string calldata name, string calldata description, uint256 value, uint256 quantity ) external onlyRole(ADMIN_ROLE) {
	
	uint256 prizeId = nextPrizeId++;	
	prizeIds.push(prizeId);

	require(bytes(prizes[prizeId].name).length == 0, "Prize ID already in use");
	require(quantity > 0, "Quantity must be greater than 0");
+   bytes32 nameHash = keccak256(abi.encodePacked(name));
+   require(!prizeNameExists[nameHash], "Prize name already exists");
+   require(value > 0, "Value must be greater than 0");
	
	prizes[prizeId] = Prize({	name: name,
		description: description,
		value: value, 
		endTimestamp: 0,
		isActive: true,
		winner: address(0), // deprecated
		winnerIndex: 0, // deprecated
		claimed: false, // deprecated
		quantity: quantity
	});

+	 prizeNameExists[nameHash] = true; //Update mapping
  
	emit PrizeAdded(prizeId, name);
}
```

### Improved Validation in `editPrize`

```diff
function editPrize( uint256 prizeId, string calldata name, string calldata description, uint256 value, uint256 quantity ) external onlyRole(ADMIN_ROLE) prizeIsActive(prizeId) {
	
+	require(quantity > 0, "Quantity must be greater than 0");
+   bytes32 nameHash = keccak256(abi.encodePacked(name));
+   require(!prizeNameExists[nameHash], "Prize name already exists"); //needs improvement in logic as this current logic prevents any edits. Multiple ways to implement logic here.
+   require(value > 0, "Value must be greater than 0");

	// Update prize details without affecting tickets or active status
		Prize storage prize = prizes[prizeId];
		prize.name = name;
		prize.description = description;
		prize.value = value;
		prize.quantity = quantity;
		emit PrizeEdited(prizeId, name, description, value, quantity);
	
}
```

Note: The simple `prizeNameExists` approach requires careful handling during edits (allowing the same prize to keep its name, updating mapping on name changes, etc.). Implement logic to clear the old name hash and set the new one atomically when editing.

## Proof of Concept

PoC (private gist): <https://gist.github.com/blackgrease/d9d093be8f1b295122a2d3ae61eee305>

Run test with:

```
forge test --mt testMissingInputValidationAndDuplication  --via-ir -vvv
```

{% stepper %}
{% step %}

### Walk-through: Step 1 — Create prizes

* The admin creates three prizes by calling `addPrize`.
  * Prize 3 has the same name as Prize 1.
  * Prize 2 has a different name.
  * All prizes have value of 0.
  * Quantity is set correctly in all three prizes, passing validation in `Raffle::addPrize`.
    {% endstep %}

{% step %}

### Walk-through: Step 2 — After creation

* All 3 prizes are successfully created.
* It is asserted that Prize 3 has the same name as Prize 1.
  {% endstep %}

{% step %}

### Walk-through: Step 3 — Edit prize to duplicate name and zero quantity

* Admin calls `Raffle::editPrize` to edit Prize 2 to:
  * have the same `name` as Prize 1
  * have a `quantity` of 0 (no winners)
* The call passes because `editPrize` lacks checks.
* Now there are three prizes with the same name and one has `quantity = 0`.
  {% endstep %}

{% step %}

### Walk-through: Step 4 — Request winner fails

* Admin calls `Raffle::requestWinner` for Prize 2 but it reverts with `AllWinnersDrawn()` error because `winnersDrawn[prizeId] >= prizes[prizeId].quantity`.
* Test assertions use `getPrizeDetails(prizeId)`. Console logs display informative messages.
  {% endstep %}
  {% endstepper %}

<details>

<summary>Additional PoC details (private gist)</summary>

* The PoC includes a Foundry test demonstrating:
  * Creation of prizes including duplicate names via `addPrize`.
  * Editing a prize to `quantity = 0` via `editPrize`.
  * Attempting to request a winner and observing the `AllWinnersDrawn()` revert.
* Link: <https://gist.github.com/blackgrease/d9d093be8f1b295122a2d3ae61eee305>

</details>

## Notes

* Both `addPrize` and `editPrize` do not validate `value` allowing prizes of zero value.
* When implementing name-uniqueness checks, ensure edit flows allow retaining the same name for the same prize and properly update any name-existence mappings on name changes.

***

If you want, I can:

* Propose a more complete, edit-safe implementation (with mapping updates on name change and proper checks),
* Draft unit tests to cover these edge cases (adding duplicates, editing to zero quantity, `requestWinner` failure), or
* Produce a minimal patch/PR diff ready to apply to the repo.
