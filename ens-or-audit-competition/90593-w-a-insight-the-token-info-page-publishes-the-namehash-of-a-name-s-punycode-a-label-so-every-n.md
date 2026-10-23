# 90593 w a insight the token info page publishes the namehash of a name s punycode a label so every non ascii ens name shows the node of a different separately registerable name

**Submitted on Aug 25th 2026 at 20:04:51 UTC by @rev3rse for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #90593
* **Report Type:** Websites & Apps
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Causing the application to display or pre-fill an incorrect address, name, or transaction detail, so that a user sends funds or submits a transaction to an unintended recipient, such as:
    * resolving or rendering an ENS name to an attacker-controlled address
    * showing the wrong recipient or amount on a transaction the user then approves

## Description

## Brief/Intro

`routes/$name/token.tsx:288` computes `namehash(asciiEncode(name))`, and `asciiEncode` (`utils/token/ascii.ts`) is `new URL('https://' + name).hostname` — the WHATWG/UTS-46 host parser. ENS namehash is defined over the **UTF-8, ENSIP-15-normalized** name, so for every non-ASCII name the page prints the node of a **different name**, beneath a green **"Normalized"** badge and next to a copy button.

No attacker and no malformed input are required: `münchen.eth` is fully normalized, `isNormalized()` returns **true**, and the badge is green above a wrong node. Of the **16** production `namehash(` call sites under `apps/`, this is the only one that hashes an IDNA-transformed value.

## Vulnerability Details

```tsx
// routes/$name/token.tsx
211   namehash(name)             // ENSv1 token id — correct
288   const hash = namehash(ascii)   // the Namehash row — the defect
```

Two rows derived from the same route parameter, **77 lines apart in one file**, returning different values. `utils/token/ascii.ts` returns `xn--mnchen-3ya.eth` for `münchen.eth`, and `ens_normalize` **throws** on that string (`invalid label extension: "xn--"`) — the value has no intended consumer.

## Impact Details

The Explorer publishes, under the victim's name, an identifier that resolves to a **separately registerable** node. The Manager accepts `xn--` labels (`-` is absent from `INVALID_LABEL_CHARS`), so an attacker registers the twin and owns the node the Explorer prints. On the fork, `registry.ownerOf(0x6525ac23…)` returns the attacker; the victim's own name and an unregistered id are the negative controls.

The app's single ENSIP-15 gate on that path fires **after the mint** (`setPrimaryName.ts:49`), so the attacker sees *"Registration Failed"* for a name that is already theirs.

## Recommended Fix

{% hint style="warning" %}
**`namehash(name)` alone is not the fix** — `asciiEncode` lower-cases (correct) *and* punycodes (incorrect), so raw is wrong for `ALICE.eth` where the current code is right:
{% endhint %}

```
              raw           asciiEncode    normalize
ALICE.eth     0x7ccc7bd6 ✗  0x787192fc ✓   0x787192fc
münchen.eth   0xbbb99291 ✓  0x6525ac23 ✗   0xbbb99291
```

Use `namehash(normalize(name))` — correct in both directions, and the pattern `useNameHistoryTimeline.ts:234` already uses.

## References

1. `apps/portal/src/routes/$name/token.tsx` (288, 211) — the defect and the same-file contradiction 77 lines apart.
2. `apps/portal/src/utils/token/ascii.ts` (1-7) — `asciiEncode` is `new URL('https://'+name).hostname`, i.e. IDNA ToASCII.
3. `apps/portal/src/routes/$name/resolver.tsx` (170, 198) — a row also labelled `Namehash`, computed correctly from the same parameter.
4. `apps/portal/src/utils/token/isNormalized.ts` (18) — `ens_normalize(label) === label`, the green badge's source; true for `münchen.eth`.
5. `apps/manager/src/features/register-v2/utils/name-parser.ts` (11) — `INVALID_LABEL_CHARS` omits `-`, which is why the `xn--` twin registers.

## Link to Proof of Concept

<https://gist.github.com/almog-shachar/c2a3e14a55963f5a8a6f682ee0d02b6c>

## Proof of Concept

Single-file patch, `1 file changed`. **18 of 18 pass offline** (case 10, the chain half, is skipped and prints a banner saying a skip is not a pass).

```bash
git checkout 1c9b47f1 && git apply poc.patch && pnpm install
cd apps/portal && ./node_modules/.bin/vitest run src/__ens_poc_token_namehash__/
```

The **rendering arm** mounts the app's real `Route` through a memory router — attached with the same `TokenRoute.update(...)` call the generated `routeTree.gen.ts` makes — and reads the assertions out of the mounted DOM, not out of a re-implementation. **R7** proves fixture-independence: replace the owner fixture entirely and the `Namehash` row does not move. **R5** is the sharpest: `münchen.eth` (green *Normalized*) and `xn--mnchen-3ya.eth` (red *Not Normalized*) render the **byte-identical** node.

`evidence/SUBMISSION-EXHIBIT.png` — one image, two panels, Sepolia fork pinned at block **11562283**:

1. `/münchen.eth/token` — green **Normalized**, `ASCII xn--mnchen-3ya.eth`, `Namehash 0x6525ac23…`. The `DNS encoded` row two lines above is `0x086dc3bc…` — **UTF-8 `münchen`** — so the card uses the name where punycode is *mandatory* and the punycode form where it is *forbidden*.
2. `/münchen.eth/resolver` — a row **also labelled `Namehash`**, same app, same parameter: `0xbbb99291…`, the correct node.

Adding gist link here again: <https://gist.github.com/almog-shachar/c2a3e14a55963f5a8a6f682ee0d02b6c>
