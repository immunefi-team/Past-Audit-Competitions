# 93026 w a critical a name sent with the portal s default plan bricks every third party subname beneath it and the name s new holder can re mint those labels to an address of their choosing

**Submitted on Sep 14th 2026 at 00:49:18 UTC by @I\_A\_J for** [**Audit Competition | ENS**](https://immunefi.com/audit-competition/audit-competition-ens)

* **Report ID:** #93026
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-ens/tree/audit-comp-ready/apps/portal>
* **Impacts:**
  * Hijacking or bricking an ENS name during a Manager flow (registration, v1 to v2 migration, or transfer), such as: - a registration or migration that assigns the name, its ownership, registry, or resolver to an attacker - redirecting the recipient of a name transfer - leaving a name permanently inaccessible after the flow

## Description

In apps/portal , the "Send name" flow ships a Detach registry step that is on by default and writes setSubregistry(parent, 0) . That pointer is the route every \*.parent.eth subname resolves through, so a routine, default-configured transfer stops every subname under the parent from resolving — including subnames owned by independent third parties who are not party to the transfer, are not notified, and hold no role that would let them repair it. The same write then opens a takeover: the registry-configuration form renders only when the subregistry slot is zero, so after the detach, whoever holds the parent can deploy a fresh registry and re-mint the victim's label to an address of their choosing, while the victim's original token sits stranded in the old, now-unreferenced registry. Both halves are executed on a pinned fork — the bricking, the gate opening, and the deploy-and-set takeover through the real UI in a browser, the final re-mint asserted on-chain.

**The separation from known issues are:**

* WEB-1249 (Functional, Manager — a create-subname can replace the existing child registry, orphaning nested subnames) is the adjacent entry, and its own section carves this finding out. Part 2 preamble, verbatim: "a demonstrated security consequence of one of them (for example, a role combination that grants authority it should not, rather than merely erroring) goes beyond the known breakage and is a new finding." No corpus entry states the orphaned label can be re-issued to an attacker-chosen address; that consequence is executed ( test\_L , browser run). The root causes differ on every axis, any one sufficient: (1) different function, different fix — create-subname path ( create-subname.helpers.ts:65 ) vs. transfer path ( useTransferDetachTargets / buildTransferPlan ); neither patch resolves the other, and test\_L asserts the slot zero at the write, so even a refuse-replace-on-occupied contract fix does not reach this path. (2) different consent posture — a permissioned account's deliberate create vs. a routine default-on sale, the takeover executed by a buyer who saw nothing. (3) different victim class — subnames under the actor's own name vs. an independently-owned third-party asset.
* QA-01 pre-declares the class new: "A demonstrated security consequence (e.g., a role combination that grants unintended authority) would be a new finding." ORPHAN-01 is a demonstrated security consequence of exactly such a transfer-flow role combination.
* WEB-1257 is the same flow with the opposite failure — the registry is not detached after the transfer; this finding is the detach succeeding by default. Different mechanism, victim, and consequence.
* WEB-716 is disjoint by construction: the takeover steps run only after this bug empties the slot, so the form is used exactly as designed ( test\_G , test\_K ).

**The default transfer plan includes a registry detach**

apps/portal/src/features/transfer/components/SendNameForm.tsx:69-73 ships all three plan options enabled, including detachRegistry: true . The plan modal for the executed run presented exactly \["Detach resolver","Detach registry","Transfer name"] with both detach switches on, as shipped (browser run, step 1; 04-modal-plan.png ). The seller need only fill in a recipient and confirm.

**The victim holds nothing**

The zeroed pointer lives on the parent; the victim's ERC-1155 token sits inside the orphaned registry, still owned by them, now unreachable. The app confirms both halves in one frame after the transfer: /example.shark-cat.eth renders "Name not found" while the old registry still shows the token owned by the victim (browser run, step 3). The repair form's role check ( ConfigureRegistryForm.tsx:94-99 ) fails for the victim by construction — the role it checks lives on the parent label, which the victim has never held.

**The empty slot is what opens the takeover**

Before the detach, the app offers no registry-replacement path at all — the configuration form is gated on the slot being empty. After the detach, the buyer's app instance renders "Configure registry" with "Deploy a new Permissioned Registry contract" preselected (browser run, step 4–5). Two clicks deploy a fresh registry and point the parent at it. From there, registering the victim's label to any address is the form's ordinary function.

**The composition:**

a routine sale hands a stranger the re-mint power

**Victim:**

an independent third party holding a subname under the transferred parent. In the fixture, example.shark-cat.eth (owner 0xA80aB75523b64Ea6968aD649b38cFeBB33eD1F43 , primary name sminky.eth ) — an account that has never held the parent and holds zero role bits on the parent label, asserted as raw stored bitmaps in setUp . A second, independent holder ( mrshark.shark-cat.eth ) is orphaned by the same single write ( test\_C ), so the harm scales to multiple independent owners per transfer.

**Attacker, version 1 — malicious seller:** detach with defaults, keep or move the token to a second controlled address, re-mint the victim's label.

**Attacker, version 2 — stranger buyer (executed):** an honest seller runs the documented default flow; the buyer — who never saw the toggle, was never told the subtree existed, and has no relationship with the victim — inherits ROLE\_SET\_SUBREGISTRY and its admin with the token ( test\_F ). The mechanics are identical for a Sybil self-sale; the stranger matters because no one who ends up holding the power was ever shown the disclosure.

**Economics:** parents with live subregistries and third-party holders are enumerable from public chain state. The attacker buys one at market price from a seller who does not know they are selling the power to re-mint every label under it, then takes the labels they want — to hijack an established identity or ransom it back.

A third party's name stops resolving as a direct result of someone else's routine defaultconfigured transfer, and from that state whoever holds the parent can assign the name — ownership, registry and all — to an address of their choosing. The asset is not burned: the ERC-1155 token still exists, still owned by the victim. It is unreachable, and unreachable specifically to the person who owns it — the app's own repair routes check roles on the parent, which the victim has never held. Every consumer resolving the subname in the interim gets "Name not found"; after the takeover, they get the attacker's address. Recovery is not available to the victim at all: the only restore paths run through the parent holder ( test\_D , test\_F , test\_H , test\_I ), and once the label is re-minted, the victim's original can never resolve again unless the attacker chooses to repoint.

Execution files — exact citations. Every executable claim runs from exactly two entry points; nothing else in the gist executes.

**Role** **Path cited to the report** **File in the gist** Contract PoC — entry point run.sh ORPHAN-01\_\_run.sh

Contract PoC — test logic (test\_A … test\_L) test/Orphan01ThirdParty.t.sol ORPHAN-01\_\_test\_\_Orphan01ThirdParty.t.sol

Contract PoC — recorded output evidence.txt ORPHAN-01\_\_evidence.txt

Browser run — entry point app-run/run.sh ORPHAN-01\_\_app-run\_\_run.sh

Browser run — UI driver (Playwright) app-run/drive.mjs ORPHAN-01\_\_app-run\_\_drive.mjs

Browser run — recorded transcript (tr: cites) app-run/evidence/transcript.log ORPHAN-01\_\_app-run\_\_evidence\_\_transcript.log

Foundry config (scopes forge test to test/ only) foundry.toml ORPHAN-01\_\_foundry.toml

## Link to Proof of Concept

<https://gist.github.com/ijomha20/2fe9c3fdf17d752dd80b9e94cb8f0db2>

## Proof of Concept

**Executed and pinned**, in two independent harnesses at Sepolia block 11,616,799 against pre-existing live chain state. Nothing touches a live network and no audited file is modified. The contract suite deploys nothing — the parent, child registries and both third-party owners are all pre-existing live Sepolia state; the browser run's only deployment is the fresh registry the buyer deploys on the local fork, which is itself the executed takeover step.

**Browser-driven run ( app-run/ ):** drives the real portal UI in headless Chromium against an anvil fork, using the app's own supported mock-wallet mode ( VITE\_USE\_MOCK\_WALLET , the mode the app ships for its own e2e). app-run/run.sh reproduces it end to end: clone the audited repository at the pin → fork Sepolia → two real portal instances (seller :3001 , buyer :3010 ) → Playwright driver, with final ownership assertions enforced by the script. Recorded run: apprun/evidence/transcript.log (cited as tr: ). Provenance is asserted first: the checkout is verified at 1c9b47f18fcddd2e864dfe385c4171061c9811ae ( tr:2 ); both instances' chain-data path reports the fork block ( tr:4-5 ); the parent's subregistry pre-state is the live 0x11CC26c54EE744fe8F98cC9cC223a4948cB2acA9 ( tr:6 ); the victim's name resolves and the "Configure registry" button is absent ( tr:9,11 ).

**Step 1** — seller runs the shipped flow with defaults untouched ( tr:28 ; 02–05 screenshots): all three transactions mine from the real UI (seller nonce 801→804, tr:35 ; receipts tr:165-167 ). **Step 2** — the detach commits: getSubregistry(shark-cat) → 0x0 ( tr:86 ). **Step 3** — the third party's name stops resolving: /example.shark-cat.eth renders "Name not found"; the old registry still shows the token owned by the victim ( tr:97-98 ; 06- post-victim-not-found.png ). **Step 4** — the empty-slot gate opens for the buyer (separate app instance, buyer wallet): "Configure registry" present only after the transfer ( tr:110 ; 07-buyer-registryform-appears.png ). **Step 5** — buyer clicks through with "Deploy a new Permissioned Registry contract" preselected: two UI-driven transactions deploy a fresh registry and point the parent at it (buyer nonce 0→2, tr:118 ; receipts tr:168-169 ); getSubregistry → 0x8CF76C0807cC32A6605A426121a6b364D54E6102 ( tr:119 ; 09-buyer-registry-set.png ). **Step 6** — the victim's label is re-minted to a receiver the buyer chose ( 0x…a11c ) — contract-level leg, boundary below: register receipt at block 11,616,805, status success, from the buyer account ( tr:145-146,170 ). **Step 7** — final assertions, enforced by the script: the app's profile page now shows the receiver, not "Name not found"; new-registry owner = receiver; old-registry owner = still the victim; final block 11,616,805 — every block past the pin is this run. RESULT: PASS ( tr:159-163,171 ; 12-final-resolves-to-receiver.png ).

Every step through the deploy-and-set takeover is executed by the real UI in the browser. The final re-mint leg is executed at contract level from the buyer account, because the app's own availability gate — fed by a stale external indexer, not by chain state — blocks its create-subname form on the fork. The form refused its own button with "example.shark-cat.eth is already registered" ( tr:133-143 ; 10- / 10b- screenshots — the run's 11-buyer-subname-created.png frame captures this same gated form, not a submitted creation). The transcript captures the availability query in flight: a getSubnames("sharkcat.eth") request to the app's compiled-in default indexer, staging-graphql.ens.dev ( tr:101,122-123 ) — an index of real Sepolia, a chain with no detach on it, so on the fork it can never observe the new state. It is a data-freshness check, not an on-chain guard: the gate is isLabelTaken in create-subname.tsx:145-153 , a getSubnames lookup then existingSubnames.some(s => s.labelName === label) , with no reachability filter of its own. The indexer supplies that filter by default — its DomainFilter exposes includeUnreachable as an opt-in, so names that no longer resolve are omitted from a parent's subdomains list unless asked for. Observed live on 2026-09-13 against the same endpoint: uncle.eth carries an unreachable child, agent3.uncle.eth , that is absent from its default subdomains list while its reachable sibling agent2 is present. After a real detach the victim's label is exactly such an unreachable child, so on production data the form reads it as free and permits the re-mint; on the fork the indexer still reflects the un-detached live chain. Because the disabled form never prepares calldata, no UI transaction exists to compare against; the run therefore completes step 6 as a contract-level register of the same label to the same receiver from the buyer account — runcontrolled on the private fork via anvil account impersonation, standing in for the attacker's own key (the buyer is the attacker; control of one's own account is the attack precondition, and the same account's deploy-and-set transactions went through the real UI at blocks 11,616,803–804) — and labels it as such in the transcript. The re-mint's on-chain outcome is asserted identically either way, and test\_L executes the full sequence on-chain without any UI.

**Contract suite ( test/Orphan01ThirdParty.t.sol , 12/12 PASS at the pin):** driven by run.sh ; recorded output evidence.txt . Foundry fork at block 11,616,799, nothing deployed; setUp asserts ten fixture invariants and aborts with a FIXTURE guard failure if any has moved. What each group establishes:

**One default step bricks third-party names; tokens untouched; scales to multiple independent owners** — test\_A , test\_C : both children stop resolving after one setSubregistry(parent, 0) ; both tokens remain their owners'.

**The victim has no available action** — test\_B : the victim's repair reverts with the exact EACUnauthorizedAccountRoles(resource, ROLE\_SET\_SUBREGISTRY, victim) , resource read from chain.

**The repair routes run through the parent holder** — after a sale, the buyer, never the victim — test\_D , test\_F , test\_H , test\_I : restoration works only from the parent holder; the role and its admin move with the token to the buyer; six specified accounts are each asserted at the pin to hold neither the role nor its admin; the only route that ends with the victim acting still begins with the counterparty.

**The bounds are exact** — test\_G , test\_K : the detach alone takes nothing (the same caller registers an unused label successfully, then is refused on the victim's label with the exact LabelAlreadyRegistered("example") ); the contract permits replace-on-occupied, so the app's empty-slot gate is the sole guard.

**The repair default finalises the loss** — test\_E : the preselected "deploy" path re-points the parent at a fresh empty registry; state looks repaired, the victim stays broken, and the only live record of where their token lives is overwritten.

**The takeover, end to end** — test\_J , test\_L : the orphaned label is re-minted; the victim's name resolves to the actor's address while their original token is stranded. test\_L runs the buyer variant: default plan completes, the slot is asserted zero at the moment of the write — so even a contract-level refuse-replace-on-occupied patch does not reach this path — and every takeover call is made by the buyer, who held nothing before the transfer.

**Controls record ( evidence.txt §2–§6):** three consecutive pinned runs byte-identical in status and gas; a different pin inside the fixture window (11,500,000) passes identically; the fixture guard fires when pinned before the subregistry existed (FORK\_BLOCK=11,423,957 — one block before the subregistry's set block; the capture's section banner labels it by that set block, 11,423,958); all 12 pass live at head. Five mutation controls prove the assertions can fail — delete the detach (M1), let the parent owner attempt the victim's repair (M2), point the "repair" at the correct registry (M3), delete the re-mint (M4), delete the token transfer (M5) — each produces the targeted failure, and run.sh enforces them: a control that passes, or fails for an unrelated reason, fails the script itself.

**Reproduce: SEPOLIA\_RPC\_URL**= ./run.sh (contract suite; FORK\_BLOCK may be set to any pin inside the fixture window). cd app-run && ./run.sh for the browser run (selforchestrating; requires git, node, pnpm, foundry; Playwright fetches a Chromium or point CHROME\_PATH at one). forge-std v1.16.2 is vendored in lib/ as plain files, so the tree compiles from the unzipped archive with no install step. No credential is embedded in the package; the default pin is beyond the window a pruning full node retains, so an archive endpoint is required.
