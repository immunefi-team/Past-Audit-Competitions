# 91098 w a critical renew page shows the correct name but signs a different one invisible unicode causes wrong recipient renewal

**Submitted on Aug 29th 2026 at 16:35:53 UTC by @debuhkzr for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #91098
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/manager>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

While going through the renew flow in apps/manager, I noticed the route loader for /renew/$name reads the same URL parameter twice, using two different functions, and never checks that they agree.

At line 20 it calls parseRenewableName(name) to get the label that eventually gets encoded into the renew() transaction. This function just does trim().toLowerCase() plus a blocklist of a few ASCII characters — it doesn't run ENSIP-15 normalization at all.

A few lines later, at line 26, it calls profileExpiryQuery(name), which internally calls normalizeEth2LdName -> viem's normalize(). This is the ENSIP-15-compliant one. This second value is what drives everything the user actually sees and everything that gates the flow: the expiry date, the eligibility checks, canRenewV2Name, all of it.

So if you feed it a label that contains a character ENSIP-15 strips out but the loose parser doesn't touch — U+200B (zero width space) is a good example — you get a page that displays and validates against the victim's real, clean name, while the transaction underneath encodes a completely different label. Since U+200B renders as nothing, the heading and the tooltip look 100% identical to the real name. There is no visual tell at all.

I checked and this isn't just U+200B — there's a whole set of invisible codepoints that ens\_normalize deletes but the loose parser lets through unchanged: U+00AD, U+180B through U+180F, U+200B, U+2060, U+2064, U+FE00 through U+FE0F, and U+1BCA0 through U+1BCA3.

### Where this lives in the code

* Execute path: apps/manager/src/routes/renew/$name.tsx line 20 (returned line 61)
* Display/gate path: same file, lines 26-28, gates checked at 30/40/44/52, expiry shown at 62
* The loose normalizer itself: apps/manager/src/features/register-v2/utils/name-parser.ts line 46
* The strict one: apps/manager/src/features/profile/service/profileName.ts line 18, via normalizeEth2LdName at line 62
* Where the gate reads from: apps/manager/src/features/profile/service/profileExpiry.ts line 77
* Where it finally gets encoded into calldata: packages/transaction-manager/src/machines/registration/registration.actors.ts, encodeRenewData at lines 1020-1031, submitted around 1072-1094

### How you'd actually pull this off

{% stepper %}
{% step %}

## Register the attacker-controlled label

Register the label `"foobar"` + U+200B somewhere. I confirmed this is possible on-chain by pulling the contracts-v2 source at the commit the app is actually pinned to (8d1c893) — the only check the registrar does is on byte length, nothing about normalization.

I'm not claiming a bug in the contracts here since they're out of scope, I'm just establishing that this label can exist and reach the manager app.
{% endstep %}

{% step %}

## Identify a victim name

Victim already owns the real foobar.eth and it's within its renewal window.
{% endstep %}

{% step %}

## Send the renewal link

Send them a link like app.ens.domains/renew/foobar%E2%80%8B.eth. It's just a URL, nothing suspicious to click through.
{% endstep %}

{% step %}

## Victim approves the transaction

They land on the page, everything looks completely normal — correct name, correct expiry, correct price (same length tier so the price doesn't even look off) — and they approve.

Their payment ends up renewing the attacker's registration. Their own name just keeps counting down to expiry, untouched.
{% endstep %}
{% endstepper %}

### On duplicates

I want to flag this myself rather than let a reviewer find it. There's a disclosed issue, EXP-INPUT-005, that's in the same general area: "ENS name validators accept inputs that ENS normalization later rejects or transforms... causing homograph-style display confusion in the UI rather than incorrect resolution." U+200B is a "transforms" case, not a "rejects" case, so this does fall into the character class that disclosure already covers. I'm not going to pretend otherwise.

What I think makes this a separate finding is the impact ceiling that disclosure states explicitly — it says the result is UI confusion, "rather than incorrect resolution." What I'm describing isn't incorrect resolution either, it's an incorrect argument getting encoded into the actual transaction. That's a different and worse outcome than what's disclosed, and the disclosure's own wording rules it out of its scope.

There's also a pattern the program uses elsewhere in the same document that I think supports this: QA-03 (a display-only price mismatch) explicitly says a mismatch between what's displayed and what's actually charged on-chain would count as a new finding. This is the exact same shape of argument, just on the name field instead of the price field.

I'll be upfront that I think there's real risk a reviewer disagrees with me here and treats it as covered by EXP-INPUT-005 — I'd rather say that plainly than have it look like I tried to hide it.

### Existing correct handling

The codebase already handles this correctly somewhere else. NamesTable.tsx (around lines 83-90) deliberately keeps displayName and the actual normalized on-chain label as two separate fields, and there's a comment there acknowledging exactly this kind of hazard. The bulk-renew path does it right. This single-name route is the one place that doesn't.

### Suggested fix

Use one normalizer for both what's shown and what's encoded, and redirect to the canonical URL if the input wasn't already normalized. Longer term, parseName should probably just reject anything where ens\_normalize(input) !== input.

## Link to Proof of Concept

<https://gist.github.com/Devendrakumarsinha/476c05d51da4809bf634a5273135d05c>

## Proof of Concept

I wrote a test file rather than describing this only in prose, since I wanted to show the actual bytes at every step instead of just asserting it happens. File is apps/manager/src/features/renew/RESEARCH\_renew-divergence.poc.test.tsx — nothing in production code was touched, this only adds a new test file.

{% stepper %}
{% step %}

## Clone the repository

```bash
git clone https://github.com/immunefi-team/audit-comp-ens.git
```

{% endstep %}

{% step %}

## Check out the target branch

```bash
cd audit-comp-ens && git checkout audit-comp-ready
```

Confirmed commit:

```
1c9b47f18fcddd2e864dfe385c4171061c9811ae
```

{% endstep %}

{% step %}

## Install dependencies

```bash
corepack enable && pnpm install
```

{% endstep %}

{% step %}

## Add the test file

Drop the attached test file into:

```
apps/manager/src/features/renew/
```

{% endstep %}

{% step %}

## Run the test

```bash
cd apps/manager
pnpm exec vitest run src/features/renew/RESEARCH_renew-divergence.poc.test.tsx --reporter=verbose
```

{% endstep %}
{% endstepper %}

The test uses the actual parseRenewableName, actual normalizeEth2LdName, and actually renders PricingDomainHeader and calls submitRenewActor. The only things mocked are the transaction-manager singleton and the payment preflight check — same pattern the repo already uses in its own registration.renew\.actors.test.ts, so it's not an unusual setup.

Here's what it printed, tracing the same route param through every step:

```
route param            : "foobar" + U+200B + ".eth"
loader execute value    : "foobar" + U+200B
loader display/gate value: "foobar.eth"
labelhash of execute    : 0xbca3b73ec1206acd214bc012f1a833df9c2b12a4cd8ef9c0f8a307975aa9570d
labelhash of "foobar"   : 0x38d18acb67d25c8bb9942764b62f18e17054f66a817bd4295423adf9ed98873e
rendered heading        : looks identical to "foobar.eth", U+200B present but invisible
what gets submitted     : "foobar" + U+200B
final calldata target   : 0xa88553F454b77203B0D036A05c894d555EAAa2Cc
encoded label in calldata: "foobar" + U+200B
```

You can actually see it in the raw calldata bytes — length byte 0x09, then 666f6f626172 which is "foobar", then e2808b right after, which is the UTF-8 bytes for U+200B.

I also included a negative-control test with a clean label with no weird characters, just to show it agrees at every step when there's nothing to diverge on. Both tests pass.

I also spent some time checking whether this label could even be registered in the first place, since if the registrar rejected it the whole thing would be moot. I cloned ensdomains/contracts-v2 locally and checked at the exact commit the app pins to (8d1c893). The relevant check is in StandardRentPriceOracle.getBasePrice() (lines 365-374) and it only checks that the byte length is between 1 and 255 — nothing about character content. I grepped both register() and renew() for any additional validity check and found nothing. The label-hashing function, LibLabel.id(), is just keccak256 of the raw bytes, so there's no normalization happening there either. There's actually a comment right on the contract itself that says "Does not check if normalized," so this isn't really in question.

I want to be clear I'm not filing anything against the contracts here since they're out of scope for this competition — I only checked this to make sure the attack scenario in apps/manager is actually reachable and not just theoretical.

Screenshot and screen recording of the test run passing are attached.
