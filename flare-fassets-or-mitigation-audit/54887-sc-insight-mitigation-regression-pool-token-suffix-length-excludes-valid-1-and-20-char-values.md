# #54887 \[SC-Insight] mitigation regression pool token suffix length excludes valid 1 and 20 char values the fix rejects valid edge lengths and breaks agent creation&#x20;

* **Submitted on Sep 18th 2025 at 11:55:26 UTC by @Disqualified-User for** [**Mitigation Audit | Flare | FAssets**](https://immunefi.com/audit-competition/flare-fassets--mitigation-audit)
* Report ID: #54887
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/flare-foundation/fassets/commit/59373cee12e6d2a9fa0a9cc8735bb486faa51b36>
* Impacts: Griefing (no profit motive for an attacker, but damage to the users or the protocol)

## Description

Brief/Intro While fixing #45439 to disallow empty pool token suffixes, the contract introduced a boundary regression that narrows the allowed suffix length range to 2–19 characters. The mitigation added a new minimum check len >= 2 while the maximum check remained exclusive len < 20. As a result, both a one-character suffix and a twenty-character suffix are now rejected on-chain in createAgentVault. The change is in commit 59373cee (“require pool token suffix length >= 2 – immunify 45439”), which modifies AgentsCreateDestroy.sol and adjusts unit tests accordingly. The end result is an inconsistent and unnecessarily strict validator that causes reverts for otherwise reasonable and expected suffix choices.

This is a single validation-range mismatch that manifests through two boundary choices (raising the minimum to 2 and keeping the maximum exclusive). The fix is minimal, restores a natural 1–20 range, and the added tests pin the boundary so this class of drift doesn’t recur.

### Severity

Medium — Griefing (no profit motive for an attacker, but damage to the users or the protocol). No funds are stolen or frozen, but valid configurations are rejected at the contract layer, blocking otherwise legitimate agent setups and creating avoidable friction/failed transactions.

## Vulnerability Details

The validator lives in contracts/assetManager/library/AgentsCreateDestroy.sol and is reached from createAgentVault, where the pool token suffix is reserved and checked before creating the agent. In the mitigation commit, two key lines were added:

```
uint256 internal constant MIN_SUFFIX_LEN = 2;
uint256 internal constant MAX_SUFFIX_LEN = 20;
...
require(len >= MIN_SUFFIX_LEN, "suffix too short");
require(len <  MAX_SUFFIX_LEN, "suffix too long");
```

Because the upper bound uses a strict “less than,” the accepted length set becomes 2…19. That rejects len == 1 and len == 20. Everything else in the validator remains the same: only A–Z, 0–9, and - are allowed, with hyphens not permitted at the start or end. The commit diff and the accompanying test change make this easy to verify. The test file adds an assertion that an empty suffix reverts as “suffix too short,” but there are no positive tests pinning the len == 1 or len == 20 boundaries, which is how this drift slipped through CI.

This is a real, user-visible regression: short branding-style suffixes (e.g., A) and exactly twenty-character suffixes now hard-revert, preventing otherwise legitimate agent setups. People frequently choose one-character suffixes (branding, initials) or hit the round figure of exactly twenty characters when they construct mechanized identifiers. With the current checks, those reasonable inputs hard-revert, and agent creation can’t proceed. The behavior is purely a byproduct of the boundary arithmetic in the mitigation; there is no deeper protocol reason to forbid 1 or 20.

## Impact Details

The issue denies service for a subset of otherwise sensible suffixes. Attempting to create an agent with a one-character or exactly twenty-character suffix reverts during validation, so the agent cannot be created without changing the chosen name. This is damaging without any attacker profit motive: it burns gas, blocks deployments, and forces users to alter naming conventions, while leaving all funds safe.

In practice, an agent owner who picks a one-letter suffix like `A` or a 20-character suffix will hit a hard revert in createAgentVault. That is not a loss-of-funds event, but it’s a contract-level rejection of otherwise sensible inputs that breaks parity with user expectations and tooling. At scale, this presents as repeated failed transactions, support load, and inconsistent naming across deployments—classic “griefing” style damage with no attacker profit required. That's the basis for Medium severity under the posted scope.

## Recommendation

Align the validator to inclusive, human-friendly bounds and protect the intent with explicit boundary tests. The simplest change is to accept 1…20 characters by lowering the minimum to one and making the maximum check inclusive. If you prefer keeping an exclusive upper bound, rename the constant to reflect exclusivity and set it to 21, but the inclusive form below is clearer and matches how the rest of the code reads.

Proposed patch (contract):

```
diff --git a/contracts/assetManager/library/AgentsCreateDestroy.sol b/contracts/assetManager/library/AgentsCreateDestroy.sol
--- a/contracts/assetManager/library/AgentsCreateDestroy.sol
+++ b/contracts/assetManager/library/AgentsCreateDestroy.sol
@@ -27,8 +27,8 @@ library AgentsCreateDestroy {
     using Agent for Agent.State;
     using Agents for Agent.State;
 
-    uint256 internal constant MIN_SUFFIX_LEN = 2;
+    uint256 internal constant MIN_SUFFIX_LEN = 1;
+    uint256 internal constant MAX_SUFFIX_LEN = 20;
 
     modifier onlyAgentVaultOwner(address _agentVault) {
         Agents.requireAgentVaultOwner(_agentVault);
@@ -246,8 +246,8 @@ library AgentsCreateDestroy {
         // validate - require only printable ASCII characters (no spaces) and limited length
         bytes memory suffixb = bytes(_suffix);
         uint256 len = suffixb.length;

          require(len >= MIN_SUFFIX_LEN, "suffix too short");
-        require(len < MAX_SUFFIX_LEN, "suffix too long");
+        require(len <= MAX_SUFFIX_LEN, "suffix too long");

         for (uint256 i = 0; i < len; i++) {
             bytes1 ch = suffixb[i];
             // allow A-Z, 0-9 and '-' (but not at start or end)
```

## Proof of Concept

Step-by-step reproduction

{% stepper %}
{% step %}

### 1. Clone and check out the mitigation commit

```
git clone https://github.com/flare-foundation/fassets.git
cd fassets
git checkout 59373cee12e6d2a9fa0a9cc8735bb486faa51b36
```

This is the commit that introduced `MIN_SUFFIX_LEN = 2` and the new minimum-length check.
{% endstep %}

{% step %}

### 2. Install and run tests

```
yarn
yarn test
```

{% endstep %}

{% step %}

### 3. Boundary tests (add tests that should pass)

Apply the following additions to test/unit/fasset/library/Agents.ts to assert acceptance of 1- and 20-character suffixes and continued rejection of invalid cases:

```
@@ -225,6 +225,43 @@ contract(`Agent.sol; ${getTestFile(__filename)}; Agent basic tests`, async accou
         assert.equal(await poolToken.symbol(), "FCPT-BTC-AGX");
     });
 
+    it("should accept 1-character pool token suffix", async () => {
+        const agent = await createAgent(agentOwner1, underlyingAgent1 + "_len1", { poolTokenSuffix: "A" });
+        const pool = await CollateralPool.at(await agent.collateralPool());
+        const poolToken = await CollateralPoolToken.at(await pool.poolToken());
+        assert.equal(await poolToken.name(), "FAsset Collateral Pool Token BTC-A");
+        assert.equal(await poolToken.symbol(), "FCPT-BTC-A");
+    });
+
+    it("should accept exactly 20-character pool token suffix", async () => {
+        const s20 = "ABCDEFGHIJKLMNOPQRST"; // length = 20
+        const agent = await createAgent(agentOwner1, underlyingAgent1 + "_len20", { poolTokenSuffix: s20 });
+        const pool = await CollateralPool.at(await agent.collateralPool());
+        const poolToken = await CollateralPoolToken.at(await pool.poolToken());
+        assert.equal(await poolToken.name(), "FAsset Collateral Pool Token BTC-" + s20);
+        assert.equal(await poolToken.symbol(), "FCPT-BTC-" + s20);
+    });
+
+    it("should continue to reject empty, >20, bad chars, and hyphens at ends", async () => {
+        await expectRevert(
+            createAgent(agentOwner1, underlyingAgent1 + "_empty", { poolTokenSuffix: "" }),
+            "suffix too short"
+        );
+        await expectRevert(
+            createAgent(agentOwner1, underlyingAgent1 + "_tooLong", { poolTokenSuffix: "ABCDEFGHIJKLMNOPQRSTU" }), // 21
+            "suffix too long"
+        );
+        await expectRevert(
+            createAgent(agentOwner1, underlyingAgent1 + "_badChars", { poolTokenSuffix: "A B" }),
+            "invalid character in suffix"
+        );
+        await expectRevert(createAgent(agentOwner1, underlyingAgent1 + "_hyStart", { poolTokenSuffix: "-AB" }), "invalid character in suffix");
+        await expectRevert(createAgent(agentOwner1, underlyingAgent1 + "_hyEnd",   { poolTokenSuffix: "AB-" }), "invalid character in suffix");
+    });
```

Re-run the tests:

```
yarn test
```

You will observe that attempts to create an agent with "A" (length 1) and "ABCDEFGHIJKLMNOPQRST" (length 20) revert during createAgentVault on the mitigation commit, confirming the regression.
{% endstep %}

{% step %}

### 4. Apply the fix and verify

Apply the contract patch shown in the Recommendation section (changing MIN\_SUFFIX\_LEN to 1 and using <= for MAX\_SUFFIX\_LEN). Then re-run the full test suite:

```
yarn test
```

Observed result: the new 1-char and 20-char acceptance tests will pass. Existing negative cases (empty string, length 21, invalid characters, hyphen at either end) will continue to revert as before, proving the intent is preserved and the regression is fixed.
{% endstep %}
{% endstepper %}

## References

* Flare FAssets | Mitigation Audit: <https://immunefi.com/audit-competition/flare-fassets--mitigation-audit/scope/#top>
* \#45439 \[SC-Low] Empty String Allowed as Pool Token Suffix in \_reserveAndValidatePoolTokenSuffix: <https://reports.immunefi.com/flare-fassets/45439-sc-low-empty-string-allowed-as-pool-token-suffix-in-\\_reserveandvalidatepooltokensuffix>
* Smart Contract - Fix of Report - 45439: <https://github.com/flare-foundation/fassets/commit/59373cee12e6d2a9fa0a9cc8735bb486faa51b36>

(End of report)
