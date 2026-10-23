# 89300 w a insight patched rhinestone sdk identity is undocumented in source two provenance comments assert two different superseded sha 256 values&#x20;

**Submitted on Aug 18th 2026 at 13:59:43 UTC by @Oxb4b for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #89300
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/packages/smart-account>
* **Impacts:**

## Description

## Brief/Intro

`packages/smart-account/src/providers/rhinestone/manifest.ts` and `session-storage.ts` each contain a hand-maintained comment recording the SHA-256 of the patched `@rhinestone/sdk@1.8.0` dependency. The two comments disagree with each other, and neither matches the hash of the patch file actually on disk and pinned in `pnpm-lock.yaml`. The result is that no comment anywhere in the source tree correctly identifies the patch that ships in production. This is a documentation/provenance defect confined to review and incident-response accuracy — patch integrity itself is fully enforced by the pnpm lockfile and `--frozen-lockfile` CI installs, so there is no supply-chain gap and no runtime or user-facing impact.

***

## Vulnerability Details

**Affected files:** `packages/smart-account/src/providers/rhinestone/manifest.ts:17`, `packages/smart-account/src/providers/rhinestone/session-storage.ts:20-22`, `patches/@rhinestone%2Fsdk@1.8.0.patch`, `pnpm-lock.yaml` (patch hash entries).

`manifest.ts:17` records the patched SDK's identity in a header comment:

```ts
// manifest.ts:17
 * SDK patch SHA-256: 5e0a5f328ccf65514b051f255c217e693d81a9bfcf8ccbbd71dc2729e8932867
```

`session-storage.ts`'s storage-key versioning log, in the same package, records that this exact value was superseded:

```ts
// session-storage.ts:20-22
 *   v6 → v7: SDK patch bumped (5e0a5f32… → 7603298e…). The session
 *            authorization / enable-data encoding changed, so any session
 *            signed under the old patch is invalid and must be re-authorized.
```

The patch file actually shipped hashes to a **third**, different value:

```
$ sha256sum 'patches/@rhinestone%2Fsdk@1.8.0.patch'
2731d300bb788ac89bf12ea4b7f063b45878701e044d346dccad7cfb6fe91728
```

This third value is exactly what `pnpm-lock.yaml` pins as the patch's integrity hash (`patchedDependencies["@rhinestone/sdk@1.8.0"].hash`), confirmed identical everywhere the entry recurs in the lockfile (root `patchedDependencies` block and both the `smart-account` and `transaction-manager` importer specifiers).

A repo-wide search confirms `5e0a5f32…` and `7603298e…` appear **only** in the two source comments above, and `2731d300…` appears **only** in `pnpm-lock.yaml`. So:

* `manifest.ts` names a patch generation at least two bumps old.
* `session-storage.ts` names the *next* generation as "current" — which is itself one bump stale.
* The patch actually in the tree and enforced by the lockfile is a third, undocumented revision.
* No comment in the source tree names the SHA-256 of the patch that ships.

**Referent is unambiguous — alternative readings ruled out.** `5e0a5f32…` is not the npm package's own integrity digest — the lockfile records that separately, as a base64 SHA-512 (`integrity: sha512-Y1YMbHBogNLjfCEI3jhjl4Q1fdEqfiyC13FZSUP/loTHPW2xCmcBBpMT9upefBkCIhRf4Mo04mozqtsqDN4dGQ==`), a different encoding and a different unit entirely. And pnpm's `patchedDependencies` hash for this entry is exactly `sha256sum` of the patch file — verified: both equal `2731d300…`. So "SDK patch SHA-256" has exactly one correct referent, and neither in-source comment supplies it.

One benign reading remains available: `manifest.ts:17` might deliberately freeze the patch generation that the file's *contract addresses* were validated against, rather than tracking the current patch. That would explain the staleness but does not resolve the defect — under either reading, no comment names the patch that actually ships, and `session-storage.ts` still separately asserts a different, also-wrong value as "current."

**Why this is Insight-tier, not a control weakness:** Patch integrity is enforced correctly and independently of these comments. `pnpm-lock.yaml` pins `hash: 2731d300bb788ac89bf12ea4b7f063b45878701e044d346dccad7cfb6fe91728` for `@rhinestone/sdk@1.8.0`, matching the on-disk patch file byte-for-byte, and CI runs `pnpm install --frozen-lockfile` on every install path (`test.yml`, `build.yml`, `security.yml`, `deploy-portal.yml`, `e2e.yml`), so a patch whose bytes diverge from the pinned hash fails the build outright. There is no tamper vector and no supply-chain gap here. Separately, no workflow or script reads or validates the in-source SHA comments — `security.yml` runs `pnpm audit` and CodeQL, neither of which inspects code comments — so this drift is silent today and will remain silent going forward.

What the defect actually costs is **review integrity**. This package documents load-bearing invariants in comments by deliberate convention elsewhere too — e.g. the exact salt field order in `session.ts:101-106`, the exact reveal-batch ordering in `registration-calls.ts:7-17`, and the calldata hash-check in `registration-calls.ts:237-247` — and the patched SDK is the dependency carrying HCA address derivation, session-enable encoding, and signature packing for this same provider. A reviewer or incident responder who reads `manifest.ts:17` to answer "is this the exact patch we audited?" gets a confidently wrong answer, and the version log they would consult next supplies a second, differently wrong answer.

***

## Impact and Likelihood Details

**Impact:** None at runtime, and none exploitable by an attacker. No code path in `manifest.ts`, `session-storage.ts`, `session.ts`, or `registration-calls.ts` reads or branches on either SHA comment — contract addresses, session salts, and storage keys are all driven by separate typed constants and the `SESSION_STORAGE_KEY` string, not by the comments. The impact is confined to the accuracy of review and incident-response documentation for the one in-scope modified third-party dependency: anyone using these comments to confirm which patched SDK build is deployed will reach the wrong conclusion, and would have to fall back to computing `sha256sum` against the lockfile directly (as this report did) to get the right answer.

**Likelihood:** Not probabilistic — this is a present, verifiable defect, not a conditional one. It reproduces on a clean checkout today and will keep reproducing until the comments are corrected, independent of timing, an attacker, or any runtime state. The concrete cost scenario is also not hypothetical: `session-storage.ts`'s version log exists specifically to tell a maintainer whether a session signed under a prior SDK patch is still valid after a bump (per its own v6→v7 note). Answering that question requires knowing the currently-deployed patch generation, and the only two places that answer is documented in source are the two comments audited here — both wrong. The value has already drifted silently across two undocumented bumps (`5e0a5f32…` → `7603298e…` → `2731d300…`, with only the first transition ever logged), so the next patch bump will reproduce this same failure unless the process changes.

***

## References

* `packages/smart-account/src/providers/rhinestone/manifest.ts:17`
* `packages/smart-account/src/providers/rhinestone/session-storage.ts:20-22`
* `patches/@rhinestone%2Fsdk@1.8.0.patch`
* `pnpm-lock.yaml` — `patchedDependencies["@rhinestone/sdk@1.8.0"]` (`hash: 2731d300bb788ac89bf12ea4b7f063b45878701e044d346dccad7cfb6fe91728`, `path: patches/@rhinestone%2Fsdk@1.8.0.patch`), and the package's own `integrity: sha512-...` entry (different value, different unit — ruled out as an alternative referent)
* `.github/workflows/test.yml`, `build.yml`, `security.yml`, `deploy-portal.yml`, `e2e.yml` — all run `pnpm install --frozen-lockfile`, evidencing that patch integrity is enforced independently of the source comments

## Proof of Concept

### Scenario 1 — Direct reproduction of the mismatch

Run on any clean checkout:

```bash
$ sha256sum 'patches/@rhinestone%2Fsdk@1.8.0.patch'
2731d300bb788ac89bf12ea4b7f063b45878701e044d346dccad7cfb6fe91728  patches/@rhinestone%2Fsdk@1.8.0.patch

$ grep -n "SDK patch SHA-256" packages/smart-account/src/providers/rhinestone/manifest.ts
17: * SDK patch SHA-256: 5e0a5f328ccf65514b051f255c217e693d81a9bfcf8ccbbd71dc2729e8932867

$ grep -n "SDK patch bumped" packages/smart-account/src/providers/rhinestone/session-storage.ts
20: *   v6 → v7: SDK patch bumped (5e0a5f32… → 7603298e…). The session
```

Three different SHA-256 values are produced; none matches the shipping patch. This is deterministic and requires no timing, no attacker, and no runtime state — it will keep reproducing until the comments are fixed.

### Scenario 2 — The version log fails at its own stated purpose

{% stepper %}
{% step %}

### A future SDK patch bump changes session authorization / enable-data encoding again

This occurs as the v6→v7 note says the `5e0a5f32…→7603298e…` bump did.
{% endstep %}

{% step %}

### A maintainer or incident responder needs to determine session validity

They need to know: "are stored sessions, signed under the previous patch, still valid, or must they be invalidated?"
{% endstep %}

{% step %}

### They consult the version log

They open `session-storage.ts`'s version log — the only place in source that is supposed to answer this — and find it still asserts `7603298e…` as "current," which is already one generation behind the `2731d300…` patch actually shipping.
{% endstep %}

{% step %}

### They cross-check `manifest.ts`

`manifest.ts:17` asserts a second, different, even-older value.
{% endstep %}

{% step %}

### Neither comment identifies the current patch

The only reliable method is to independently run `sha256sum` against the patch file and cross-reference `pnpm-lock.yaml`, exactly as this report did, bypassing the documentation that exists for this purpose.
{% endstep %}
{% endstepper %}

## Remediation

Remove both hand-maintained SHA values from `manifest.ts` and `session-storage.ts`; the lockfile already enforces patch integrity, so the comments add drift risk without adding a control. If an in-source marker is still wanted, generate it in CI from the lockfile's `patchedDependencies` hash and fail the build on mismatch. Separately, add the missing `session-storage.ts` version-log entry for the `7603298e…→2731d300…` transition (or note explicitly that this bump did not change the stored-session encoding), since that log's stated purpose is telling a maintainer whether existing sessions must be invalidated.
