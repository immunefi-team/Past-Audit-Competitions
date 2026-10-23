# ENS | Audit Competition

## Reports by Severity

<details>

<summary>Critical</summary>

* \#90165 \[W\&A-Critical] Resolver role removal merges name-scoped permissions and leaves a delegate authorized
* \#91637 \[W\&A-Critical] Manager profile page renders an attacker-controlled receiving address under a genuine name's verified owner identity
* \#91641 \[W\&A-Critical] Explorer resolver-role editor writes grants and revokes to a wrongly-derived EAC resource, so revoking a delegate's permissions silently leaves them in place
* \#92818 \[W\&A-Critical] Missing label validation in the Portal's Configure Registry flow leads to subregistry detachment of a different, real ENS name
* \#92112 \[W\&A-Critical] A direct Registry URL applies Registry bit 0 to a resolver and grants address control
* \#92902 \[W\&A-Critical] Portal treats an empty ETH record as a V1 controller resolution and transfers to the wrong recipient
* \#90320 \[W\&A-Critical] Removed resolver user can still redirect ENS names
* \#89548 \[W\&A-Critical] Registration and renewal use un-normalized labels: the name the Manager shows is not the name it writes on chain
* \#92880 \[W\&A-Critical] Explorer transfers a user's ENSv2 name to a separate address the seller selected before the sale
* \#92881 \[W\&A-Critical] ENSv1 registrant/manager conflation redirects ENSv2 name transfers to a non-owner manager
* \#92887 \[W\&A-Critical] Incomplete mixed resolver-role removal leaves a former delegate able to change an ENS address record
* \#92889 \[W\&A-Critical] Manager migration grants a seller-selected address resolver authority over the buyer's ENSv2 name
* \#92892 \[W\&A-Critical] Remove user revokes only UI-recognized role bits and can leave official authority active
* \#92890 \[W\&A-Critical] \[HIGH] Manager authorizes record edits via the resolver's stale internal roles instead of registry ownership — a name's previous owner keeps full write access (incl. the coins\[...
* \#91768 \[W\&A-Critical] "Remove user from all roles" preserves global resolver authority
* \#91717 \[W\&A-Critical] Explorer transfer flow moves a different ENS name than the UI and wallet prompt display, because on-chain identity is derived from the normalized label
* \#89253 \[W\&A-Critical] Wrong-resource revocation in Resolver Roles lets a removed account retain \`ROLE\_SET\_ADDR\` and redirect another ENS name
* \#91837 \[W\&A-Critical] Manager claims a primary name the user does not own: the reverse registrar receives the normalized name while every displayed surface and the forward addr(60) record use the ra...
* \#92278 \[W\&A-Critical] Both renewal routes prove one ENS name is renewable and then bill the user to renew a different one
* \#91224 \[W\&A-Critical] Explorer renew/transfer encode the on-chain write from a normalizing helper while the ownership gate and displayed transaction use the raw name — a victim renews or transfers a...
* \#90639 \[W\&A-Critical] Transfer authorises the raw label, moves the normalized label's token
* \#89314 \[W\&A-Critical] Manager V1 to V2 migration silently keeps an attacker's resolver and grants the attacker \`ROLE\_SET\_RESOLVER\` on the V2 registry for gifted/secondary-market names
* \#92645 \[W\&A-Critical] Reused resolver-role row state makes a Content Hash edit grant another delegate resolver-wide address control
* \#92914 \[W\&A-Critical] Missing ENSIP-15 normalization check on the primary-name render path lets an attacker's address be displayed as another party's name
* \#92658 \[W\&A-Critical] Manager migration grants a former V1 controller resolver authority over the buyer’s V2 name
* \#91855 \[W\&A-Critical] Portal treats an unresolved resolver-role scope as ROOT, enabling cross-name resolution hijacking
* \#92842 \[W\&A-Critical] A name-scoped role grant gives a limited registrar root resolver authority
* \#92877 \[W\&A-Critical] ENS Explorer’s “Remove user” action can leave global resolver permissions active
* \#92555 \[W\&A-Critical] Name transfer never revokes the seller's resolver roles, letting the former owner rewrite the buyer's ETH-address record and redirect resolution to an attacker (name hijack / a...
* \#90161 \[W\&A-Critical] Explorer authorizes a transfer against the name in the URL but moves the token of the normalized name, so disposing of an unsolicited look-alike transfers the victim's real name
* \#91095 \[W\&A-Critical] Portal “Remove user from all roles” sends a one-name revoke and leaves root resolver authority usable
* \#92944 \[W\&A-Critical] Recipient resolution returns a different address than every conformant ENS client, sending an irreversible name transfer to whoever registered a look-alike name
* \#92945 \[W\&A-Critical] # \`/renew/$name\` displays the victim's expiry but signs \`renew()\` for the attacker's label the victim pays to extend someone else's name
* \#93135 \[W\&A-Critical] Unresolved record-part role rows silently grant global resolver authority
* \#93127 \[W\&A-Critical] ENS Manager profile page reads ownership from the normalized name and records from the raw URL name, letting an attacker show their own receiving address on a victim's profile
* \#91396 \[W\&A-Critical] An IOST-only delegate can replace Manager's copied Solana recipient without modifying the SOL record
* \#92955 \[W\&A-Critical] Scope substitution in the resolver role editor converts an unresolvable name-scoped role into a resolver-root grant, giving unintended global authority over every name on the r...
* \#93129 \[W\&A-Critical] Stale role state tied to the row position in the resolver Roles sheet quietly adds root ROLE\_SET\_ADDR to a grant, so ENS names resolve to an attacker-controlled address
* \#89465 \[W\&A-Critical] Lookalike-label renewal trap: /renew displays the normalized ENS name but renews the raw label — victim pays to renew an attacker's name, then loses their own
* \#92962 \[W\&A-Critical] Resolver role editor revokes the wrong scope
* \#92967 \[W\&A-Critical] The Manager checks name ownership on one node and writes to another
* \#92971 \[W\&A-Critical] Manager falsely confirms a canonical ENS renewal while extending an attacker-owned raw Unicode name
* \#92990 \[W\&A-Critical] Cross-account receipt reuse skips resolver-role revocation and makes Portal render an attacker-controlled ENS address
* \#90071 \[W\&A-Critical] Lossy resolver-role grouping makes "Remove user from all roles" leave global ROLE\_SET\_ADDR active
* \#89601 \[W\&A-Critical] Missing ENSIP-15 normalisation in the Manager registration path causes users to pay for and receive a different, permanently unresolvable name than the one the app showed as av...
* \#89630 \[W\&A-Critical] Renewal payment is redirected to an attacker's look‑alike name
* \#90931 \[W\&A-Critical] Resolver row-index reuse carries a removed user's root Upgrade permission into another collaborator and enables shared-resolver takeover
* \#90693 \[W\&A-Critical] The Manager profile page reads records from the raw URL name and ownership from its ENSIP-15 name, so an attacker-controlled address record is rendered under a real name's owner
* \#92993 \[W\&A-Critical] Explorer Send name checks the name in the URL and signs away the owner's real name
* \#89467 \[W\&A-Critical] Inconsistent name normalization between the renewal checks and the renewal transaction leads to the user signing a renewal for an attacker-controlled name
* \#89483 \[W\&A-Critical] Empty resolvedNames falls back to root scope, so editing one role row signs grantRootRoles over every name on the resolver
* \#91900 \[W\&A-Critical] Explorer resolves a recipient ENS name that has no addr(60) record to the owner of its ENS v2 registry entry, so one ordinary paid v2 registration by a stranger redirects a nam...
* \#89270 \[W\&A-Critical] Manager profile page resolves records from the raw URL name but owner/expiry from the normalized name, letting an attacker display their own address under a victim name's identity
* \#92411 \[W\&A-Critical] Incorrect labelhash resolution in Portal leads to deletion of other unintended ENS subnames, and resolving hijacking of unintended
* \#89387 \[W\&A-Critical] "Remove user from all roles" leaves root resolver authority active, enabling ENS address redirection
* \#89563 \[W\&A-Critical] ENS Explorer resolver "Remove user" / Save silently leaves the root (all-names) grant intact, so a removed manager keeps write access to every name
* \#91098 \[W\&A-Critical] Renew page shows the correct name but signs a different one — invisible unicode causes wrong-recipient renewal
* \#92510 \[W\&A-Critical] Portal authorizes an attacker-gifted raw-NFD name but transfers the victim's distinct NFC ERC-1155 name token
* \#92111 \[W\&A-Critical] Portal authorizes a fullwidth raw ENSv2 name but transfers the victim's ASCII canonical twin NFT
* \#90951 \[W\&A-Critical] Portal "Remove user from all roles" submits incomplete revocation calldata and leaves the delegate authorized
* \#89611 \[W\&A-Critical] Resolver “Remove user” revokes only one name and leaves global address/upgrade authority active
* \#89761 \[W\&A-Critical] Removing a resolver user revokes only one displayed ENS name, allowing redirected incoming payments
* \#89990 \[W\&A-Critical] Resolver roles page revokes a single name while claiming removal of all roles — removed users keep live record-write authority on other names and ROOT
* \#90784 \[W\&A-Critical] Critical: Portal account-only role grouping collapses resolver scopes, leaving an offboarded collaborator able to redirect ENS resolution
* \#92860 \[W\&A-Critical] Missing grantee validation in the v1→v2 migration's "manager restoration" grants an attacker ROLE\_SET\_RESOLVER over a victim's migrated ENS name, hijacking where the name resolves
* \#89336 \[W\&A-Critical] V1-to-V2 migration grants stale owners resolver control
* \#90498 \[W\&A-Critical] Resolver role table merges different names into one row but signs calldata for only one name
* \#90725 \[W\&A-Critical] Encoded-labelhash confusion in Portal deletion unregisters a different victim-owned subname and enables registrar takeover
* \#90622 \[W\&A-Critical] Lossy resolver-role aggregation causes “Remove user from all roles” to retain global permissions, enabling ENS record hijacking
* \#92222 \[W\&A-Critical] Portal “Remove user” leaves root text authority, allowing a former delegate to overwrite another owner's profile
* \#92094 \[W\&A-Critical] \[Critical] Explorer role removal preserves global ROLE\_SET\_ADDR, enabling post-offboarding ENS address diversion
* \#90637 \[W\&A-Critical] Removing a resolver user leaves every root-scoped role in place
* \#91757 \[W\&A-Critical] Resolver role aggregation mis-scopes removal and preserves global authority
* \#92465 \[W\&A-Critical] Owner/controller role confusion in the Explorer's address-resolution fallback (resolveEnsOwner) leads to irreversible transfer of the user's ENS name to an unintended recipient
* \#91602 \[W\&A-Critical] Incorrect privilege assignment in the v1 to v2 migration leads to resolver hijack and loss of funds
* \#92416 \[W\&A-Critical] Mixed-case transfer route sends the lowercase sibling ENS NFT
* \#90894 \[W\&A-Critical] apps/portal resolver roles UI: "Remove user" revokes only the name scope while confirming removal from all roles; the root grant stays active
* \#90896 \[W\&A-Critical] apps/portal: "Remove user" confirms removal from all roles; the approved transaction revokes one name scope
* \#91097 \[W\&A-Critical] Explorer's “Remove user from all roles” signs a one-resource revoke, leaving a former delegate able to redirect another ENS name
* \#91333 \[W\&A-Critical] Portal's “Remove user from all roles” submits a one-resource revocation and leaves the removed delegate's root resolver authority active
* \#91449 \[W\&A-Critical] Broken Access Control: Multi-Resource Role Aggregation in Resolver Roles Management Causes Incomplete Revocation and Persistent Root-Level Access
* \#91848 \[W\&A-Critical] Editing a record-scoped resolver delegate silently submits \`grantRootRoles\` — authority over every name on the resolver
* \#91921 \[W\&A-Critical] Granular Resolver Role Escalation to Root
* \#92128 \[W\&A-Critical] Inconsistent ENS name normalization in the Manager profile page renders a trusted name's owner beside an attacker-controlled receiving address, leading to theft of user funds
* \#92429 \[W\&A-Critical] The Explorer's "Remove user" filters the root scope out of the resource it revokes, so an account the operator removed keeps write authority over the address record of every na...
* \#92239 \[W\&A-Critical] One ticked permission on a record-scoped row makes the Explorer's Roles page sign grantRootRoles — authority over every name the resolver serves — with no scope shown on any sc...
* \#92257 \[W\&A-Critical] Resolver role editor upgrades a name-scoped grant to resolver root, submitting a broader transaction than the operator selected
* \#93010 \[W\&A-Critical] Profile page reads ownership and records from two different names, so a look-alike name displays an attacker-controlled receiving address
* \#93012 \[W\&A-Critical] Explorer transfer flow checks ownership of one name and signs transactions for a different one
* \#93018 \[W\&A-Critical] Raw-cased profile URL renders the real owner over an attacker resolver's address
* \#89293 \[W\&A-Critical] Explorer resolver "Remove user" confirms a full revocation and signs a single-resource one, leaving the target holding root \`ROLE\_SET\_ADDR\`
* \#89398 \[W\&A-Critical] Missing address-record validation in resolveAddressOrName causes irreversible ENS name transfers to the wrong recipient
* \#93030 \[W\&A-Critical] Broken scope derivation in the Portal resolver roles editor leads to resolving ENS names to an attacker-controlled address
* \#92155 \[W\&A-Critical] Renewal flow checks and displays one ENS name but signs renew() calldata for a different one, so the user pays to extend a name they were never shown
* \#91645 \[W\&A-Critical] Wrong-field ENS ownership resolution in the Explorer's recipient input sends name transfers to a v1 name's former controller instead of its owner
* \#90206 \[W\&A-Critical] The transfer flow authorises against one ENS name and signs the transfer of a different one
* \#90770 \[W\&A-Critical] "Remove user from all roles" revokes one resolver scope and leaves address-control authority active
* \#89408 \[W\&A-Critical] The Manager profile page asks the attacker's resolver for the victim's node, and renders the answer under the victim's real ownership and expiry
* \#93043 \[W\&A-Critical] ERC1271 signature-mode confusion lets a bounded HCA session drain the account's refund token
* \#93045 \[W\&A-Critical] Explorer resolves a recipient ENS name to the ENS v1 registry *controller*, so a retained-controller name silently redirects name transfers, subname ownership and role grants t...
* \#91004 \[W\&A-Critical] The Explorer transfer flow checks ownership of the name in the URL and moves its normalized sibling
* \#90094 \[W\&A-Critical] Renewing a name from the portal confirm screen builds the transaction for a different on-chain name than the one it displays — the payment renews \`normalize(name)\`, not the nam...
* \#91337 \[W\&A-Critical] v1 to v2 migration silently grants the V1 registry controller resolver authority over the buyer's name
* \#91335 \[W\&A-Critical] Portal applies the Registry role schema to a PermissionedResolver: one ordinary "Registrar" Add User grant becomes global Set Address authority
* \#92028 \[W\&A-Critical] Manager renew signs a transaction for a look-alike name while the UI shows the victim’s real name
* \#89278 \[W\&A-Critical] Portal displays raw ALICE.eth but transfers the distinct canonical alice.eth NFT
* \#89363 \[W\&A-Critical] ENS v1→v2 migration grants resolver authority over the migrated name to an unverified, never-displayed third party
* \#90245 \[W\&A-Critical] Explorer "Remove user" confirms removal from all roles but only revokes the name scope; root roles keep resolver-wide addr-write
* \#89593 \[W\&A-Critical] Incorrect scope inference in the resolver roles editor leaves a removed user's resolver-wide grant live, up to ROLE\_UPGRADE
* \#90190 \[W\&A-Critical] Manager bulk renewal displays the raw label but renews the normalized one, so a user's rent payment extends a name they do not own
* \#91014 \[W\&A-Critical] Scoped HCA sessions are downgraded to owner mode, bypassing refund authorization and transferring HCA tokens
* \#92042 \[W\&A-Critical] ENSv2 migration grants ROLE\_SET\_RESOLVER on migrated names to a subgraph-sourced "manager" address it never verifies on-chain or shows the user (resolver hijack)
* \#92043 \[W\&A-Critical] Explorer renewal sends \`renew()\` the ENSIP-15-normalized label while every rendered string is the raw on-chain label: the user pays full rent, their own name gains zero seconds...
* \#92121 \[W\&A-Critical] Manager renewal signs a different ENS registration from the one the page describes: the route parameter is ENSIP-15 normalized for the expiry it displays and only lower-cased f...
* \#91698 \[W\&A-Critical] Crafted /renew URL renews an attacker's look-alike name while the page displays the victim's own name and on-chain expiry
* \#92775 \[W\&A-Critical] Manager renders a lookalike ENS name with the victim's identity above the attacker's receiving address
* \#92820 \[W\&A-Critical] Explorer: "Remove user" / "Save" on a resolver operator revokes a single scope while the row aggregates root and name scopes, so root-level roles survive a "remove this user fr...
* \#89460 \[W\&A-Critical] Every ENS registration overwrites the previous one's transaction history
* \#92978 \[W\&A-Critical] Portal transfer flow authorises ownership of one ENS node and then executes irreversible writes against a different one
* \#93044 \[W\&A-Critical] The v1→v2 migration automatically grants \`ROLE\_SET\_RESOLVER\` on the newly migrated v2 name to the legacy v1 registry controller — an address that is, by construction, never the...
* \#93049 \[W\&A-Critical] Missing resolver contract-type validation turns record-role delegation into registry-wide name theft
* \#93071 \[W\&A-Critical] Portal reuses a completed resolver step after recipient correction, leaving the transferred name resolving to the prior recipient
* \#89431 \[W\&A-Critical] Stale transaction success falsely completes later role revocations, leaving resolver authority active
* \#89433 \[W\&A-Critical] Transfer flow never resets the transaction manager, so a stale step actor from an abandoned attempt is matched by id to the next attempt and the steps before it are skipped whi...
* \#93073 \[W\&A-Critical] Explorer "Remove user" on a resolver role holder with root plus named roles only revokes one named scope, leaving the root role active
* \#91485 \[W\&A-Critical] Missing ENS normalization in the Manager renewal flow leads to a victim extending the attacker's name with their own funds
* \#89420 \[W\&A-Critical] Resolver roles sidebar collapses an account's multiple role resources into one write scope, so "Remove user" displays one scope and writes another, leaving removed accounts wit...
* \#92565 \[W\&A-Critical] The Explorer's Extend flow renders the raw indexer name on every surface but signs the ENSIP-15-normalized label, so a renewal reached from the address names table pays to exte...
* \#92564 \[W\&A-Critical] apps/portal: Remove user revokes only the root grant it promised to clear entirely
* \#92567 \[W\&A-Critical] Bulk renew displays the raw name from the indexer but signs the normalized label, so a victim's stablecoins extend a registration they do not own
* \#92568 \[W\&A-Critical] Profile editing authorizes the normalized name but encodes the raw route param, so a crafted URL makes the victim sign a record write to an attacker's node on an attacker's res...
* \#92570 \[W\&A-Critical] Set primary name writes its forward legs against the raw name and signs the reverse leg against the normalized name, so one click points the victim's reverse record at a name t...
* \#92572 \[W\&A-Critical] ENSv2 registration applies no ENSIP-15 normalization at any step, so a crafted URL prices, displays and registers a label the user cannot see and that ENS does not consider par...
* \#90794 \[W\&A-Critical] Explorer confirms an uppercase ENSv2 transfer but signs the distinct lowercase token ID, causing direct NFT theft
* \#93081 \[W\&A-Critical] Resolver-roles sidebar coerces an un-invertible resource hash to the empty string, which the save path reads as ROOT, so an operator scoping a grant to one name silently grants...
* \#93080 \[W\&A-Critical] Improper resolver-role grouping causes incomplete revocation and attacker-controlled ENS resolution
* \#90255 \[W\&A-Critical] apps/portal resolver roles UI: "Remove user" revokes only the name scope while confirming removal from all roles; the root grant stays active
* \#91150 \[W\&A-Critical] Explorer replaces the displayed ENSv2 asset's ERC-1155 tokenId before the connected-wallet transfer
* \#92064 \[W\&A-Critical] Incorrect EAC resource scoping in the Resolver role editor leads to an unintended global SET\_ADDR grant
* \#90820 \[W\&A-Critical] The Manager checks name ownership with one canonicalization function and builds the renewal transaction with another, so one invisible character in a URL makes a user pay to re...
* \#91722 \[W\&A-Critical] Choosing a primary name claims the ENSIP-15-normalized twin of the displayed row, handing the victim's on-chain identity to whoever owns that twin
* \#91299 \[W\&A-Critical] Unmapped resolver role becomes global and lets a delegate redirect other ENS names
* \#93084 \[W\&A-Critical] Renewal flow signs a paid transaction against a different ENS name than the one it resolves, gates and displays
* \#92452 \[W\&A-Critical] Explorer collapses per-name resolver roles by account, so Remove User leaves a delegate able to redirect another ENS name to an attacker-controlled address
* \#90843 \[W\&A-Critical] Portal turns an unresolved name-scoped resolver role into ROOT, granting resolver-wide control over unrelated names
* \#91731 \[W\&A-Critical] Renewal screen displays an expiry date computed from a different ENS name than the one the transaction renews
* \#93090 \[W\&A-Critical] Explorer "Send name" resolves a recipient ENS name with no address record to the name's registry owner instead of failing
* \#92464 \[W\&A-Critical] Resolver Role Aggregation Causes “Remove User from All Roles” to Revoke Only One Resource, Leaving the Delegate Able to Redirect ENS Addresses
* \#90852 \[W\&A-Critical] The Explorer's irreversible name transfer resolves a recipient name with no address record to its registry controller, silently delivering the name to the previous owner while ...
* \#93093 \[W\&A-Critical] A pending role grant can be relabelled with a trusted recipient while retaining attacker calldata
* \#91688 \[W\&A-Critical] Manager authorizes a canonical-name victim to write to an attacker-owned raw-name resolver
* \#93103 \[W\&A-Critical] Deleting a resolver delegate can grant an unselected address-record role to the next delegate
* \#90459 \[W\&A-Critical] Incomplete Resolver Role Revocation Leaves Attacker Control Over Another ENS Name and Enables Direct Fund Theft
* \#91345 \[W\&A-Critical] \`resolveAddressOrName\` treats an unresolved V1 Manager as a transfer recipient, sending a V2 name NFT to an unintended address
* \#91348 \[W\&A-Critical] Public Safe ERC-1271 transcript can be redeemed first to steal an ENS seven-day JWT
* \#91306 \[W\&A-Critical] Explorer's resolver-roles editor builds every transaction against a scope it never displays, silently granting an account authority over every name a resolver serves
* \#91181 \[W\&A-Critical] Normalization desync in the Manager renewal route validates one name and renews another, letting a link make a victim pay to extend an attacker's name
* \#91062 \[W\&A-Critical] Scope collapse in the resolver-roles editor causes role writes to target the resolver ROOT resource, granting authority over every name it serves and making "Remove user" repor...
* \#92474 \[W\&A-Critical] Improper resource binding in Portal “Remove user” leaves root resolver authority untouched
* \#91318 \[W\&A-Critical] Portal's “remove all roles” action revokes only one name, leaving a delegate's global resolver authority active
* \#89952 \[W\&A-Critical] An omitted non-root role mapping makes Edit user roles encode a root-scoped grant
* \#89614 \[W\&A-Critical] Cross-resource role aggregation makes "Remove user" revoke only one name and leaves delegated resolver control
* \#90883 \[W\&A-Critical] The Manager renewal flow reads one name and pays for another, so a name's owner funds an attacker's registration while their own name is untouched
* \#89960 \[W\&A-Critical] Raw-name ownership check can transfer a different normalized ENSv2 token
* \#93111 \[W\&A-Critical] ENS falsely shows a victim as owner of an attacker's v1 name via unconsented setOwner, suppressing the third-party renewal warning so the victim pays to renew the attacker's name
* \#93112 \[W\&A-Critical] Unresolved resolver role resource becomes a root grant
* \#93115 \[W\&A-Critical] Manager renew route: the victim pays to renew an attacker's name and their own lapses
* \#93116 \[W\&A-Critical] After abandoning Alpha and switching Portal to Bravo, the surviving registration actor later submits \`register(Alpha, ...)\` while Portal displays Bravo
* \#93118 \[W\&A-Critical] Missing ENSIP-15 normalization on the register and renew write paths lets an attacker redirect a victim's renewal payment to a name the attacker owns
* \#91569 \[W\&A-Critical] Inconsistent ENS name normalization displays and copies an attacker controlled address as the victim profile's main receiving address
* \#92840 \[W\&A-Critical] Explorer's renewal builder substitutes the name it signs — the user pays to renew a different ENS name than the one displayed
* \#92535 \[W\&A-Critical] Portal row reuse can grant one resolver delegate another account's \`UPGRADE\` role
* \#93119 \[W\&A-Critical] Transfer flow authorizes one ENS name and signs transactions for a different one
* \#90337 \[W\&A-Critical] Explorer removes only one grouped role scope, allowing removed users to redirect ENS transfers
* \#92522 \[W\&A-Critical] Unconditional owner fallback in Explorer name resolution leads to the app resolving a record-less ENS name to its owner, an attacker-controlled address, pre-filled into the nam...
* \#92225 \[W\&A-Critical] Resolver “Remove user” leaves root write authority active, allowing a removed delegate to redirect ENS resolution
* \#89360 \[W\&A-Critical] The \`/$name\` profile route resolves one URL parameter as two ENS names, rendering an attacker's receiving address under the victim's genuine owner and expiry
* \#93121 \[W\&A-Critical] Renewal flow signs a paid transaction against a different ENS name than the one it resolves, gates and displays
* \#89535 \[W\&A-Critical] Manager profile page renders an attacker-chosen ETH address next to a name's genuine owner
* \#93099 \[W\&A-Critical] Resolver roles: the sidebar displays one permission scope and signs a different one
* \#90904 \[W\&A-Critical] Mis-scoped resolver role removal leaves a removed account able to repoint any name the resolver serves
* \#91566 \[W\&A-Critical] Explorer “Remove user from all roles” revokes one name resource and leaves root address authority live
* \#89930 \[W\&A-Critical] Manager profile displays a trusted name's owner but an attacker-controlled receiving address (owner-node vs records-node normalization split)
* \#90907 \[W\&A-Critical] The migration flow silently grants a third party permanent ROLE\_SET\_RESOLVER over the user's name, and destroys the only revocation path in the same transaction
* \#92483 \[W\&A-Critical] Migrate to v2, hand your name's resolver to a third party the app never validated
* \#89728 \[W\&A-Critical] Explorer app fabricates a 'Resolved:' for recordless ENS names - silently substitutes the name's owner, sending name transfers and registry-wide role grants to an attacker's ad...
* \#92738 \[W\&A-Critical] Resolver “Remove user from all roles” revokes only one scope, leaving global address-hijack authority active
* \#92764 \[W\&A-Critical] Encoded-label recipient names select an attacker-owned literal label but query the honest hash-only node, redirecting Portal name transfers
* \#92616 \[W\&A-Critical] Explorer resolves a typed ENS name to a previous owner and builds the wrong recipient transaction
* \#92671 \[W\&A-Critical] ENS Manager registers one name and then points the buyer's primary name at a registration owned by someone else
* \#92622 \[W\&A-Critical] An unmappable name-scoped resolver role becomes a resolver-wide root grant signed by the victim
* \#92771 \[W\&A-Critical] Portal treats a factory-verified UserRegistry as a PermissionedResolver, so “Set Pubkey” grants registry-wide UNREGISTER
* \#92773 \[W\&A-Critical] Unresolved resolver scopes turn scoped Set Address edits into resolver-wide grants
* \#92782 \[W\&A-Critical] registration and renewal write an on-chain node that no compliant client resolves
* \#90363 \[W\&A-Critical] Editing a Fine-Grained Resolver Role Silently Grants Root-Wide Authority
* \#92632 \[W\&A-Critical] A resolver role row whose scope the UI cannot resolve is saved as a ROOT grant, so one click on a single account’s row signs grantRootRoles over every name on the resolver
* \#91063 \[W\&A-Critical] Owner fallback in Portal's ENS name resolution pre-fills an unpublished address, sending role grants and name transfers to an unintended recipient
* \#92527 \[W\&A-Critical] The Explorer's resolver-roles editor writes a scope the administrator never chose: narrow grants are signed as resolver-wide (ROOT), and "remove from all roles" leaves ROOT sta...
* \#90874 \[W\&A-Critical] Portal authorizes a literal bracket-label decoy but changes the resolver of the embedded-hash ENS name
* \#92699 \[W\&A-Critical] A role scope the app cannot resolve is granted on every name instead
* \#92475 \[W\&A-Critical] A profile URL carrying one invisible codepoint renders the victim's verified owner beside an address published by a name the attacker registered
* \#92925 \[W\&A-Critical] The Manager profile page normalizes the name for the owner query but not for the records query, resolving an ENS name to an attacker-controlled address
* \#92310 \[W\&A-Critical] A recipient name with no address record resolves to the name's manager, and the transfer goes there
* \#91909 \[W\&A-Critical] Grouped resolver-role removal signs a single-resource revoke, leaving a removed delegate able to replace the ENS address
* \#92794 \[W\&A-Critical] Manager's bulk-renew dialog substitutes a normalized lookalike for the victim's real registered label, letting a victim's own connected wallet pay to renew an attacker's name
* \#92716 \[W\&A-Critical] Index-based row identity in Portal's resolver-roles table lets an admin's stale draft silently grant a global resolver role to an uninvolved account
* \#92713 \[W\&A-Critical] A phishing link renews the attacker's own junk registration with the victim's money while Manager displays the victim's real name
* \#92552 \[W\&A-Critical] Portal “Remove user from all roles” revokes only one resolver scope; the surviving scope permits a profile-description write that remains after role cleanup
* \#92714 \[W\&A-Critical] Resolver "Remove User" Silently No-Ops Root-Scoped Roles — Enables Persistent ENS Name-Resolution Hijack
* \#92624 \[W\&A-Critical] Two same-named withEthSuffix helpers in one post-registration sequence, one raw and one ENSIP-15, make the registration flow set the victim's primary name to a registration the...
* \#92626 \[W\&A-Critical] Manager bulk renewal renders the raw indexer name on every row and signs the normalized label, so the victim's stablecoins extend a registration that appears nowhere in the dialog
* \#92806 \[W\&A-Critical] Resolver role removal silently leaves ROLE\_SET\_DATA authority on-chain
* \#91838 \[W\&A-Critical] Migration grants a stale v1 controller a permanent ROLE\_SET\_RESOLVER on the migrated v2 name, letting a third party repoint what the victim's name resolves to
* \#92761 \[W\&A-Critical] Inconsistent ENS normalization makes Manager submit an attacker-owned label in a victim-approved renewal
* \#92174 \[W\&A-Critical] Registration and migration grant resolver-global (not name-scoped) root roles that "Detach the resolver" never revokes, so a name's seller keeps redirecting the buyer's \`addr(6...
* \#89456 \[W\&A-Critical] Renewal route prices and displays the normalized name but signs the raw label, so a crafted URL renews an attacker's lookalike name with the victim's funds
* \#92651 \[W\&A-Critical] Explorer transfer resolves an ENS recipient to the name's v1 controller rather than its owner, sending the user's name to a party that does not own the name they addressed
* \#92817 \[W\&A-Critical] Missing label validation in the Portal's Change Resolver flow leads to resolver hijack of a different, real ENS name
* \#91635 \[W\&A-Critical] Incorrect resolution on the Manager profile page displays an attacker-controlled receiving address under a legitimate ENS name's owner identity
* \#91681 \[W\&A-Critical] Manager displays the victim's NFC expiry but renews an attacker-owned raw NFD ENS token
* \#91967 \[W\&A-Critical] Ownership check on the wrong name in the Explorer transfer flow leads to hijacking of a victim's ENS name, leaving it inaccessible after the flow
* \#91080 \[W\&A-Critical] A visually identical link makes a name holder pay to extend somebody else's registration: Manager decides ownership, expiry and the third-party renewal warning on the ENS-norma...
* \#92300 \[W\&A-Critical] Portal lists an attacker-owned name under "Names you own" in place of the user's own, and transferring it transfers the user's real name instead
* \#90638 \[W\&A-Critical] Renew gate reads one name, renew calldata spends on another
* \#91869 \[W\&A-Critical] Normalization mismatch between the owner and records queries in the Manager profile route displays an attacker's name and address under the victim's ownership, causing stealthy...
* \#92859 \[W\&A-Critical] Unresolved name-scoped resolver role is converted into a root grant, enabling unauthorized record changes for unrelated names
* \#92850 \[W\&A-Critical] Two conflicting ENS name derivations on the Manager's renewal page cause the victim to pay to extend an attacker's registration instead of the name shown on screen
* \#91580 \[W\&A-Critical] Resolver roles editor writes a scope the admin was never shown, defaulting to root
* \#89864 \[W\&A-Critical] ENS Explorer silently resolves a name with no \`addr(60)\` record to its token owner (owner-fallback) and consumes it on irreversible/authority-granting action paths, misdirectin...
* \#92581 \[W\&A-Critical] The Explorer names the ENSv1 registry controller as a name's Owner, so a seller who has already handed over the NFT is still shown as the owner on the official ENS Explorer
* \#92582 \[W\&A-Critical] Set as Primary writes its forward legs to the name the user picked and its reverse leg to a different name, so one click points the victim's wallet at a name an attacker owns
* \#92584 \[W\&A-Critical] Explorer’s resolver role table omits ROLE\_SET\_DATA, so Remove user throws for every account the registration flow provisions and otherwise leaves write access the roles table r...
* \#92591 \[W\&A-Critical] Manager reads the profile with ENSIP-15 and writes the renewal with toLowerCase(), so a crafted link makes the victim's own profile suppress the third-party-renewal warning and...
* \#92593 \[W\&A-Critical] Explorer's Extend flow charges the user to renew a stranger's ENS name instead of the one shown
* \#89516 \[W\&A-Critical] v1→v2 migration assigns the migrated name's resolver to a third party and grants them the role that keeps it
* \#90647 \[W\&A-Critical] v1→v2 migration silently grants a third-party address permanent-in-practice \`ROLE\_SET\_RESOLVER\` over the user's migrated name, with the grantee never displayed and no way to re...
* \#91724 \[W\&A-Critical] Extending a name from the address page renews the ENSIP-15-normalized twin while every screen shows the raw indexer name, so the fee extends someone else's registration
* \#91830 \[W\&A-Critical] Explorer renews a different ENS name than it displays: the victim pays to extend an attacker's name while their own expires
* \#92203 \[W\&A-Critical] Primary-name dialog derives the displayed name and the written name from two different expressions, so the confirmed operation writes a reverse record for a different ENS node,...
* \#89744 \[W\&A-Critical] v1→v2 migration unconditionally grants ROLE\_SET\_RESOLVER on the v2 ETHRegistry to the name's stale v1 registry controller — an attacker-settable address the Manager never displ...
* \#90702 \[W\&A-Critical] v1→v2 migration silently grants a stale V1 controller ROLE\_SET\_RESOLVER on the migrated name, enabling resolution hijack of an acquired name
* \#89279 \[W\&A-Critical] Previous owner permanently controls what a transferred ENS name resolves to
* \#89264 \[W\&A-Critical] Persistent ROOT resolver authorization survives ENS name transfer, allowing former owners to rewrite sold-name records and unrelated resolver resources
* \#92587 \[W\&A-Critical] Explorer’s resolver permission editor keys its checkbox state to the table row index, so re-opening a row position grants the previous occupant’s permissions to a different acc...
* \#92588 \[W\&A-Critical] Transfer ownership signs the token id of getLabel(name) while every screen shows the raw route param, so the name that leaves the victim's wallet is not the name the flow displ...
* \#92352 \[W\&A-Critical] Portal says it removed all roles but sends a one-name revoke
* \#92963 \[W\&A-Critical] Cross-name profile state makes a Description-only save set an attacker address on another ENS name
* \#92602 \[W\&A-Critical] Explorer causes a connected wallet to grant a delegate resolver control over every name instead of one
* \#90691 \[W\&A-Critical] Portal authorizes a raw ENSv2 name but transfers the victim's canonical twin NFT
* \#92605 \[W\&A-Critical] Merged resolver role removal revokes one scope while leaving root record-write authority
* \#90678 \[W\&A-Critical] Explorer: "Remove user" revokes one resolver scope, leaving removed accounts with sibling-name roles and global Upgrade authority
* \#92545 \[W\&A-Critical] Removing a resolver user leaves root upgrade authority intact
* \#92559 \[W\&A-Critical] Explorer reuses a former-owner ENS resolution and encodes the wrong recipient in an ENSv2 NFT transfer
* \#92353 \[W\&A-Critical] Raw Unicode name causes Portal to transfer the victim's different canonical ENS token
* \#93016 \[W\&A-Critical] Manager and Explorer transact the normalized twin of the displayed ENS name, so a renewal pays to extend an attacker's registration and a transfer moves the wrong name — and th...
* \#89418 \[W\&A-Critical] Broken access control in the resolver Remove-user flow leaves a revoked delegate able to redirect the address records of every name on the resolver
* \#90163 \[W\&A-Critical] Manager renewal route proves renewability from the normalized name but renews the raw label, so a user pays to extend an attacker's registration
* \#93061 \[W\&A-Critical] Resolver Role Aggregation Causes “Remove User from All Roles” to Revoke Only One Resource, Leaving the Delegate Able to Redirect ENS Addresses
* \#92667 \[W\&A-Critical] Improper role revocation in Explorer leaves global ROLE\_SET\_ADDR active, enabling ENS payment redirection
* \#92752 \[W\&A-Critical] Removing a resolver user revokes only one resource and leaves root or other-name authority active
* \#92740 \[W\&A-Critical] Missing ENSIP-15 normalization on the Manager's \`/{name}\` route splits one profile page across two ENS nodes, rendering an attacker-controlled address beside the real owner and...
* \#92753 \[W\&A-Critical] Incomplete revocation in the resolver user-removal flow retains resolver-root authority, leading to theft of user funds via payment-address redirection
* \#92755 \[W\&A-Critical] Missing scope validation in ENS Explorer role edits grants resolver-wide address access
* \#93026 \[W\&A-Critical] A name sent with the portal's default plan bricks every third-party subname beneath it, and the name's new holder can re-mint those labels to an address of their choosing
* \#91295 \[W\&A-Critical] IOST-only resolver delegate can make Manager show and copy its value as the SOL receiving address
* \#91982 \[W\&A-Critical] Manager keep-v1 migration of a locked custom-resolver name freezes record control
* \#92345 \[W\&A-Critical] Attacker can register a name's normalized twin to redirect the owner's bulk-renew payment and let their name expire
* \#92762 \[W\&A-Critical] Late ENS normalization makes the Portal detach and transfer a different name than the transfer page authorizes
* \#89395 \[W\&A-Critical] Unsafe owner fallback in the shared address resolver leads to irreversible transfer of ENS names to an unintended address
* \#89474 \[W\&A-Critical] Explorer transfer flow moves a different ENS name than the one displayed, stealing the victim's name
* \#89917 \[W\&A-Critical] The app acts on a different ENS name than the one it displays and just verified you own — an unsolicited airdrop turns "get rid of this junk" into irreversible loss of a valuab...
* \#89585 \[W\&A-Critical] \[Critical] Name transfer resolves its recipient to the v1 registry controller, not the ERC-721 holder -- the Explorer also labels that controller as Owner
* \#91640 \[W\&A-Critical] Bulk renewal signs a different label than the name displayed, so the fee renews an unintended name (displayed-vs-signed mismatch)
* \#91044 \[W\&A-Critical] Explorer authorises a raw ENS name but transfers the normalised name, so a user disposing of a decoy burns their genuine name
* \#91887 \[W\&A-Critical] Resolver-role resource conflation makes “Remove user from all roles” leave global address-write authority
* \#89888 \[W\&A-Critical] Missing ENSIP-15 normalization in Manager registration lets an attacker own the canonical name while a victim buys a hidden lookalike
* \#92640 \[W\&A-Critical] Explorer authorizes record editing by resolver role instead of name ownership — a former owner can edit the new owner's records in-app, and the real owner is locked out (name h...
* \#89507 \[W\&A-Critical] "Remove user" on the resolver roles panel revokes only one resource scope, leaving listed authority in place
* \#90688 \[W\&A-Critical] Incomplete role revocation: Resolver "Remove user" leaves a co-manager's global (root) role in place, so a removed user can still set any name's address record and redirect funds
* \#92949 \[W\&A-Critical] Explorer \`Remove User\` revokes only one resolver scope and leaves global address write authority active
* \#91121 \[W\&A-Critical] Manager profile page resolves one route name against two different ENS nodes, rendering an attacker-controlled receiving address beside the victim's genuine ownership data
* \#89641 \[W\&A-Critical] Explorer invents a recipient address and sends the user's ENS name to it
* \#90379 \[W\&A-Critical] Deleting one resolver-role row rebinds its permission draft to the next account and grants that account an unselected address role
* \#90544 \[W\&A-Critical] Post-confirmation Unicode normalization substitutes an attacker-owned primary name, causing Manager to render an attacker-controlled Optimism recipient.
* \#92746 \[W\&A-Critical] ENS Explorer's “Remove user from all roles” leaves active address-control permission on another name
* \#91524 \[W\&A-Critical] Base58 case folding substitutes the Solana copy target

</details>

<details>

<summary>High</summary>

* \#92196 \[W\&A-High] Portal keys address-record identity by display symbol: selecting an oversized same-symbol alias deletes the owner's Base Sepolia record and resolves the name to a prior owner's add...
* \#91834 \[W\&A-High] Missing resolver-ownership validation in the name-history query leads to persistent forged content injection on any ENS name's history page
* \#92947 \[W\&A-High] Registration checkout presents the rent as the exact total while the wallet signs a permit for \~2.5× that amount
* \#93128 \[W\&A-High] manager HCA quote failure can authorize more USDC than the exact wallet Total shown at checkout
* \#92833 \[W\&A-High] A third party can brick a wrapped V1 ENS name for 99 years through Manager renewal
* \#92413 \[W\&A-High] Unpinned Warp settlementLayers and dest-only 0x03 session policy lead to direct theft of Hybrid Custody Account funds
* \#93021 \[W\&A-High] Swallowed budget-quote failure makes the registration checkout show the rent as an exact total while the machine sizes the funding permit from the full budget, so the user approves...
* \#93029 \[W\&A-High] Manager writes L2 address records to mainnet-derived coin types while the Explorer reads Sepolia-derived ones, so all five L2 rows render the owner's coin-60 address and a sender's...
* \#91001 \[W\&A-High] The registration checkout shows the rent as the total while no funding budget has been fetched, and admits the click on that figure
* \#91924 \[W\&A-High] Manager can display a lower registration total and execute a higher HCA spend without reconfirmation
* \#93046 \[W\&A-High] Unchecked registration subregistry lets an HCA session signer retain control of victim-owned subnames
* \#91648 \[W\&A-High] HCA funding permit is sized from an unbounded orchestrator quote with no ceiling or cross-check
* \#93001 \[W\&A-High] Any contract can write persistent fabricated rows - including a forged "Set address to 0xAttacker" - into any ENS name's History page: the v1 history query has no resolver-provenan...
* \#89462 \[W\&A-High] HCA registration budget omits the resolver deployment, so a first registration is funded short and its commitment cannot be recovered
* \#92461 \[W\&A-High] Portal registration shows $8.01 for name B, charges 160.109598 USDC for name A
* \#92463 \[W\&A-High] Unbounded EIP-2612 permit in \`planHcaIntentFunding\` lets a hostile intent quote drain the connected wallet's USDC on a transaction whose amount the app never displays
* \#92540 \[W\&A-High] The Explorer's role-history panel identifies a name by its label hash alone and never by the registry that holds it
* \#89328 \[W\&A-High] Registration checkout charges more than displayed
* \#92665 \[W\&A-High] Manager does not bind Orchestrator-returned HCA registration calldata to the requested calldata, allowing an attacker-controlled subregistry on a victim-owned root name
* \#91441 \[W\&A-High] An unrelated contract can inject persistent arbitrary text into another name's canonical Portal history
* \#92779 \[W\&A-High] Owner-signed HCA intents are signed from the orchestrator's returned intent with no execution-equivalence check, on the client or on chain
* \#89578 \[W\&A-High] Unreconciled funding budget in Manager HCA registration leads to the user approving a permit for 2.5× the total the checkout displays
* \#89727 \[W\&A-High] Manager checkout does not bind the accepted USDC amount to the later Permit and transfer
* \#93137 \[W\&A-High] Charged-vs-displayed divergence on the HCA registration checkout: on a failed budget quote the screen shows the rent (~~$8) but the funding permit charges the full budget (~~$20.20).
* \#90749 \[W\&A-High] Portal reparses a flat dotted label and grants resolver authority on a different ENS name
* \#93033 \[W\&A-High] apps/portal's address-history page trusts unconsented on-chain ownership: anyone can plant attacker-authored history (and a forged "From" address) on any address's page including a...
* \#92597 \[W\&A-High] HCA registration checkout can display exact rent while a recovered quote authorizes a larger wallet debit
* \#92996 \[W\&A-High] Portal's symbol collision preserves a prior owner's Base address when a canonical row is deleted
* \#92110 \[W\&A-High] Portal reparses a two-dot raw label and authorizes a different four-label ENS name
* \#89329 \[W\&A-High] Uncapped funding permit → full USDC drain on owner-signed HCA actions

</details>

<details>

<summary>Medium</summary>

* \#92825 \[W\&A-Medium] Missing registry scoping in the role History query lets anyone write permanent forged role-change entries into any ENS name's audit trail
* \#92899 \[W\&A-Medium] Stale V1 resolver snapshot during migration reinstalls a removed attacker-operated resolver
* \#89366 \[W\&A-Medium] Missing execution-time owner re-verification in v1→v2 migration leads to ROLE\_SET\_RESOLVER granted to a former manager
* \#91689 \[W\&A-Medium] Portal direct child route and spoofable resolver authentication substitute an attacker contract as the victim's transaction target
* \#91913 \[W\&A-Medium] Incomplete \`KNOWN\_PUBLIC\_RESOLVERS\` skip of locked-record safety wipes ETH/text records on Manager v1→v2 Upgrade
* \#92867 \[W\&A-Medium] Failed HCA reveal is accepted from attacker-created registration state, leaving hidden resolver authority and an incorrect receiving address
* \#92870 \[W\&A-Medium] Case-folded role-name lookup in the Manager profile leads to persistent injection of third-party ENS names on any address's public page
* \#90446 \[W\&A-Medium] No per-user push-channel cap and faulty retry after a partial commit permanently suppress other users' external expiry reminders
* \#92822 \[W\&A-Medium] Explorer copies a preserved ENSv1 resolver into a new ENSv2 subname, letting an ENSv1 party choose the address a user's transfer lands on
* \#92885 \[W\&A-Medium] Stale V1 Manager is encoded as the nested V2 grantRoles recipient after a same-ID refresh
* \#92908 \[W\&A-Medium] Historical resolver provenance check reattaches attacker-upgraded code during profile editing
* \#91765 \[W\&A-Medium] Stale V1 migration state resurrects a revoked manager on V2
* \#89249 \[W\&A-Medium] Stale authorization in V1-to-V2 Manager migration resurrects a revoked manager and lets a former seller hijack the ENS resolver
* \#92913 \[W\&A-Medium] A keep-resolver sale strands the sold name under the seller's resolution control -- and if the seller burns the resolver role first, no one can ever undo it
* \#93130 \[W\&A-Medium] A stale Manager migration plan can restore a former owner's attacker-controlled resolver
* \#89967 \[W\&A-Medium] Manager executes cached V1 migration state, restoring revoked resolver authority and redirecting ENS payments
* \#92951 \[W\&A-Medium] Missing on-chain validation in ENS migration lets a third party hijack the resolver of a user's name
* \#93133 \[W\&A-Medium] Stale V1 migration state restores a revoked manager and attacker resolver in ENSv2
* \#92836 \[W\&A-Medium] Payment-Token Picker Silently Treats a Failed Price Read as $0, Hiding Insufficient-Balance and Skipping the Real Approval Amount
* \#92964 \[W\&A-Medium] Manager migration can restore a revoked V1 manager’s permission to change the resolver
* \#92977 \[W\&A-Medium] Missing registry filter in useRoleHistory lets any user plant fake role grants in another name's Explorer History panel
* \#92976 \[W\&A-Medium] Stale V1 manager state during ENSv2 migration resurrects former-manager resolver authority and enables name hijacking
* \#93132 \[W\&A-Medium] Explorer stamps a verified "Forward match: True / Primary name" badge on an unverified default.reverse name, so any address displays a name it does not own as its confirmed prima...
* \#90056 \[W\&A-Medium] TOCTOU in V1-to-V2 migration restores revoked manager resolver authority
* \#92373 \[W\&A-Medium] Same-ID migration refresh retains removed resolver authority in an executable plan
* \#92398 \[W\&A-Medium] Unscoped, Replay-Unprotected SendGrid Webhook over Duplicate Email Rows Lead To Silent Cross-User Disabling of ENS Email Notifications
* \#92995 \[W\&A-Medium] A Contact-tab Discord edit republishes inherited social handles through Manager's resolver replacement
* \#89381 \[W\&A-Medium] Migration can restore resolver authority to a former V1 manager after reclaim
* \#92097 \[W\&A-Medium] Manager migration resurrects a removed attacker-controlled resolver and re-grants a relinquished manager role from a stale V1 snapshot
* \#91931 \[W\&A-Medium] Manager migration reauthorizes a revoked V1 manager, enabling resolver hijack
* \#91690 \[W\&A-Medium] Manager migration restores a revoked V1 manager with persistent V2 resolver authority
* \#91974 \[W\&A-Medium] SendGrid webhook resolves notification channels by email alone, so a cross-user duplicate is not ownership-bound when provider events are processed
* \#91977 \[W\&A-Medium] Explorer's V2 transfer never revokes third-party roles, so a delegate the seller kept can re-point the buyer's resolver
* \#90944 \[W\&A-Medium] Portal transfer cleanup preserves third-party authority that can hijack the recipient's name
* \#91824 \[W\&A-Medium] V1→V2 migration resurrects a revoked legacy manager with fresh \`ROLE\_SET\_RESOLVER\` after the victim reclaims the V1 node
* \#90957 \[W\&A-Medium] V1-to-V2 migration can reinstall a revoked attacker-controlled resolver after Manager receives corrected V1 state
* \#90239 \[W\&A-Medium] Revoked V1 managers regain persistent V2 resolver control due to stale manager state during migration
* \#91127 \[W\&A-Medium] Unbounded favorite fan-out lets an attacker poison shared expiry batches and suppress cross-user notifications
* \#90254 \[W\&A-Medium] Migration-plan cache key omits V1 authority fields, restoring an attacker resolver
* \#93006 \[W\&A-Medium] \[MEDIUM] Portal registration checkout fails open on a single transient price-read failure — the UI freezes at $0.00 and drops every approval step while the machine charges the li...
* \#89254 \[W\&A-Medium] Cross-User Notification-Channel Disablement via Unscoped \`target\` Predicate
* \#90250 \[W\&A-Medium] Missing V1 Registry owner revalidation re-authorizes a revoked manager during V2 migration, enabling resolver hijacking
* \#89285 \[W\&A-Medium] Fresh V1 owner updates do not invalidate a migration plan, re-authorizing a revoked manager
* \#90981 \[W\&A-Medium] A former owner will redirect a transferred ENS name to an attacker-controlled address for the new owner by retaining PermissionedResolver write authority.
* \#90980 \[W\&A-Medium] Manager migration can grant a revoked V1 manager persistent V2 resolver authority, enabling ENS name hijacking
* \#93019 \[W\&A-Medium] ENS Manager regrants a revoked V1 manager during migration, enabling resolver hijack
* \#90626 \[W\&A-Medium] Manager migration restores a removed V1 manager, enabling resolver hijack and fund theft
* \#90755 \[W\&A-Medium] Cached V1 manager authority is restored as a persistent V2 resolver role during migration
* \#89352 \[W\&A-Medium] V1 to V2 migration silently replays prior-owner resolver records, pointing the victim's migrated name at the attacker
* \#93032 \[W\&A-Medium] Stale V1 manager snapshot restores revoked resolver authority during migration
* \#91600 \[W\&A-Medium] Manager executes a cached migration plan after V1 authority revocation, restoring the revoked manager and resolver
* \#91852 \[W\&A-Medium] Former v1 manager is incorrectly granted V2 resolver authority, allowing ENS redirection and NFT theft
* \#89356 \[W\&A-Medium] TOCTOU in V1-to-V2 manager validation enables ENS resolver hijacking by a revoked manager
* \#93040 \[W\&A-Medium] Stale ENSv1 manager restoration during migration grants a revoked attacker ENSv2 resolver control
* \#93042 \[W\&A-Medium] Migration resurrects a revoked V1 manager from stale indexed state
* \#93051 \[W\&A-Medium] Default clean transfer preserves a prior resolver delegate, allowing the recipient's ENS name to be hijacked
* \#91342 \[W\&A-Medium] Manager resurrects a relinquished V1 manager as a V2 resolver delegate after the pre-wallet state check
* \#89374 \[W\&A-Medium] Stale V1 Registry state reinstalls a revoked attacker resolver during V1-to-V2 migration, enabling ENS name hijacking
* \#93056 \[W\&A-Medium] Migration replays attacker controlled records from a removed cached V1 resolver into the authoritative V2 resolver
* \#92988 \[W\&A-Medium] Missing live V1 owner check re-grants resolver control to a revoked manager during migration
* \#92994 \[W\&A-Medium] Stale V1 migration data restores an ex-manager's resolver and permanent V2 authority after a split handoff
* \#92997 \[W\&A-Medium] Missing factory check in the Permissioned Resolver lookup lets any contract get ENS's "audited and considered secure" badge on its page and OG card
* \#93053 \[W\&A-Medium] Failed per-token price read renders a selectable $0.00 quote, skips the approval leg, and the amount-less renew silently charges the live price against the standing 2x allowance
* \#93007 \[W\&A-Medium] Explorer address page has a visitor's wallet sign setName and take the attacker's name as primary
* \#89445 \[W\&A-Medium] One user's bounce or unsubscribe rewrites every other user's channel with the same target
* \#92578 \[W\&A-Medium] Notification channel state is read and written by target address alone, with no user\_id predicate, over a schema that deliberately allows the same target on multiple accounts
* \#93079 \[W\&A-Medium] A stale Manager migration plan can restore a former owner's attacker-controlled resolver
* \#92177 \[W\&A-Medium] Stale V1 manager snapshot re-grants a former manager V2 resolver authority during migration
* \#92178 \[W\&A-Medium] ENS migration can restore resolver authority to a revoked manager from stale subgraph data
* \#92205 \[W\&A-Medium] Stale V1 authority cache restores attacker resolver control during ENSv2 migration
* \#90807 \[W\&A-Medium] Stale authorization in V1-to-V2 manager restoration re-grants revoked resolver control, enabling ENS name hijack
* \#92156 \[W\&A-Medium] useMigrationGasEstimate preserves an authority-bearing plan after manager revocation, restoring a revoked address during V1-to-V2 migration
* \#90812 \[W\&A-Medium] TOCTOU in Manager migration installs an attacker resolver and restores former V1 manager authority
* \#91615 \[W\&A-Medium] queues/event-ingestion.ts: idempotency key stamped before the delivery fan-out
* \#93096 \[W\&A-Medium] Missing Manager revalidation during V1→V2 migration grants resolver control to a revoked Manager
* \#93106 \[W\&A-Medium] Stale subgraph state in the Manager's v1-to-v2 migration flow re-grants resolver-control authority to an ENSv1 manager the registrant already revoked
* \#92517 \[W\&A-Medium] Same-key record replacement is serialized deletion-last, causing Portal to expose a revoked delegate's address as the Base recipient
* \#92489 \[W\&A-Medium] apps/manager: migration grants ROLE\_SET\_RESOLVER to a revoked V1 manager
* \#92446 \[W\&A-Medium] Stale V1 migration plan restores revoked manager authority and attacker resolver, hijacking an ENS name
* \#92661 \[W\&A-Medium] Missing ownership check in the SendGrid webhook's channel lookup lets any account silently disable another person's verified ENS email notifications, permanently
* \#92466 \[W\&A-Medium] Unbounded ENS \`description\` record is emitted twice through a 6x-expanding HTML escaper, so one attacker-owned name kills the Explorer's Cloudflare Worker isolate for every concu...
* \#91314 \[W\&A-Medium] Explorer builds a user's own name list from an ensjs query whose default relation set includes \`resolvedAddress\`, so any name an attacker points at a victim's address is listed a...
* \#92542 \[W\&A-Medium] Explorer's roles editor revokes \`ROLE\_CAN\_TRANSFER\_ADMIN\` in a "Remove user" whose only warning describes a different consequence, freezing the name for the rest of its registrat...
* \#92618 \[W\&A-Medium] Revoked V1 manager receives persistent ENSv2 resolver control from an already-prepared migration
* \#92623 \[W\&A-Medium] Registry-wide role-event cap hides a retained delegate and enables post-transfer resolver and subregistry hijacking
* \#91414 \[W\&A-Medium] Revoked V1 manager can regain resolver control through a stale V1→V2 migration plan
* \#92625 \[W\&A-Medium] Stale V1 registry state restores a revoked manager and resolver during V1-to-V2 migration
* \#92778 \[W\&A-Medium] Manager accepts an attacker-controlled subregistry as a completed HCA registration after the intended reveal fails
* \#92685 \[W\&A-Medium] Stale V1 resolver state in Manager migration restores an attacker-controlled resolver and hijacks the migrated ENS name
* \#91189 \[W\&A-Medium] Portal hides resolver cleanup when the seller lacks direct ROLE\_SET\_RESOLVER, leaving a delegate able to redirect the new owner's ENS resolution and later transactions
* \#92695 \[W\&A-Medium] setupControlledResolver Intent Underfunding and Session Key Policy Reversion Prevents Controlled Resolver Deployment and Profile Setup in Manager
* \#92263 \[W\&A-Medium] A seller who keeps the resolver on a name transfer keeps root authority over the sold name, and can repoint it at their own address after the sale
* \#92503 \[W\&A-Medium] Stale V1 manager cache re-grants a revoked manager during V2 migration
* \#89627 \[W\&A-Medium] apps/portal: the name-transfer cleanup checklist omits third-party role grants, so a stranger keeps \`setResolver\` on the sold name and repoints its ETH address
* \#92649 \[W\&A-Medium] Improper record deduplication in the Portal silently deletes a reviewed Base address and resolves the ENS name to an attacker-controlled fallback
* \#92718 \[W\&A-Medium] Portal's v1 bulk-renewal table treats "resolves to me" as "owned by me", letting a victim's own connected wallet pay to renew an attacker's name
* \#92723 \[W\&A-Medium] Missing \`ccipRead\` guard in the Explorer's OG-image Worker client lets an attacker-controlled resolver drive unbounded server-side requests, exhausting the Worker's RPC provider ...
* \#92816 \[W\&A-Medium] The expiry-discovery cron's non-unique-timestamp cursor pagination permanently and silently drops names tied at a page-1000 boundary from every expiry notification
* \#92838 \[W\&A-Medium] TOCTOU in Manager v1-to-v2 migration reinstalls a replaced attacker-controlled resolver and hijacks an ENS name
* \#89903 \[W\&A-Medium] Manager migration restores resolver authority to a revoked ENSv1 manager
* \#90002 \[W\&A-Medium] Manager grants V2 \`ROLE\_SET\_RESOLVER\` from unverified V1 manager state, restoring authority to a revoked manager during migration
* \#90314 \[W\&A-Medium] Migration resurrects a revoked V1 manager with V2 resolver authority
* \#90156 \[W\&A-Medium] Missing execution-time manager validation restores a removed V1 manager and lets them hijack a migrated name's resolver
* \#92341 \[W\&A-Medium] Manager restores V2 resolver authority to a V1 manager removed before confirmation
* \#90613 \[W\&A-Medium] Stale V1 manager state during v1→v2 migration resurrects a revoked manager with ROLE\_SET\_RESOLVER, enabling resolver hijacking
* \#92585 \[W\&A-Medium] Explorer certifies any contract as the official, audited ENS Permissioned Resolver from a single storage word the contract writes itself
* \#92596 \[W\&A-Medium] Manager v1 -> v2 migration restores a removed V1 manager as persistent V2 resolver authority, hijacking the name's resolver
* \#93072 \[W\&A-Medium] Portal silently transfers a name with non-detachable third-party resolver authority
* \#92648 \[W\&A-Medium] The Explorer certifies a resolver as “official, audited and considered secure” from a storage word any contract can forge, and the same forged bit replaces the record-edit owners...
* \#92909 \[W\&A-Medium] Migration reuses uncleared resolver state and activates a revoked delegate's attacker address
* \#92934 \[W\&A-Medium] Unauthenticated server-side egress from the ENS Portal Worker via EIP-3668 CCIP-Read leads to ressource exhaustion
* \#92448 \[W\&A-Medium] Revoked ENSv1 manager regains resolver control when the owner migrates
* \#92912 \[W\&A-Medium] Any address can replace the rendered history of any ENS name
* \#92608 \[W\&A-Medium] Explorer checkout shows a $0.00 total, deletes the approval step, and the registrar charges the full rent price the user was never shown
* \#92609 \[W\&A-Medium] Stale V1 controller snapshot grants a revoked controller V2 resolver authority during migration
* \#92611 \[W\&A-Medium] Stale V1 resolver snapshot migrates an attacker-controlled resolver after V1 recovery
* \#93008 \[W\&A-Medium] recipient-controlled transfer failure leaves ENS resolution attacker-controlled and diverts payments
* \#92509 \[W\&A-Medium] Revoked manager regains resolver control during V1 to V2 migration
* \#89616 \[W\&A-Medium] Migration restores a stale former ENSv1 manager, allowing post-reclaim resolver hijacking
* \#89675 \[W\&A-Medium] v1→v2 migration re-grants ROLE\_SET\_RESOLVER to a revoked manager from a stale snapshot
* \#90583 \[W\&A-Medium] Manager trusts stale V1 owner and resolver during .eth migration
* \#92828 \[W\&A-Medium] Stale manager state restores a revoked account that hijacks the migrated ENS resolver

</details>

<details>

<summary>Low</summary>

* \#92905 \[W\&A-Low] Live resolver-access refetch changes "Replace & Continue" into a transaction to the previous owner's resolver
* \#91450 \[W\&A-Low] Unbounded \`links\` rendering blocks Manager navigation for 14 seconds on a gifted name
* \#91446 \[W\&A-Low] Quadratic primary-contact deduplication temporarily locks the victim's Manager tab after an unsolicited name transfer
* \#90307 \[W\&A-Low] Explorer renders an unbounded on-chain uint64 expiry through Temporal without a range check, so one unsolicited subname permanently breaks a victim's names page
* \#92165 \[W\&A-Low] Unauthenticated /verify endpoint binds a victim's email to the attacker's ENS account
* \#92868 \[W\&A-Low] Unauthenticated /notifications/channels/email/verify grants authority by bare token possession — cross-user email-channel hijack and permanent victim lockout
* \#93124 \[W\&A-Low] An unauthenticated attacker can recover a victim's SIWE nonce verbatim and burn it before any verification, locking the victim out of login
* \#92950 \[W\&A-Low] Portal create-subname reuses stale ENS inputs, minting to the wrong owner or an attacker-controlled resolver
* \#93125 \[W\&A-Low] Missing authorization in the notifications channel-delete handler's broadcast-cleanup lets any authenticated user delete another user's SendGrid marketing contact (config-gated).
* \#90264 \[W\&A-Low] Unauthenticated \`POST /wallet/fund\` spends a server-held private key behind a non-atomic KV "lock", allowing concurrent funds to collide on the funder's nonce and to be issued witho...
* \#92966 \[W\&A-Low] Deleting an unverified email channel can remove another user's SendGrid contact
* \#91233 \[W\&A-Low] Manager suppresses a cross-name avatar/header record update and preserves former-owner media control
* \#92408 \[W\&A-Low] Email verification endpoint has no authentication, letting an attacker bind a victim's email address to the attacker's account
* \#91925 \[W\&A-Low] Email verification binds another user's mailbox to the requesting account, unauthenticated
* \#89286 \[W\&A-Low] Unauthenticated POST /wallet/fund Enables Gas-Griefing Against the Faucet's Own Funding Wallet
* \#91813 \[W\&A-Low] Email verification never identifies the requesting wallet, so an attacker binds a victim's mailbox to their own account
* \#89891 \[W\&A-Low] /notifications/channels/email/verify verifies whoever's token you send, not whoever asks
* \#93031 \[W\&A-Low] Missing cross-user authorization on the email-channel verify endpoint lets an attacker bind any third party's email to their own ENS account, disabling that user's own notifications...
* \#93060 \[W\&A-Low] Attacker initiated verification binds a victim email to the attacker
* \#89444 \[W\&A-Low] Any anonymous caller can drain the faucet hot wallet and hang worker invocations
* \#92575 \[W\&A-Low] Any authenticated user can claim a third party's email address as a notification channel, and the victim's own click on the ENS-sent verification link locks them out of using their ...
* \#93089 \[W\&A-Low] SendGrid webhook matches events to the first channel row with that email, so a pre-registered pending duplicate absorbs another user's unsubscribe or bounce
* \#93094 \[W\&A-Low] Notification delivery targets are treated as global identities: provider events and push registrations rewrite other users' notification settings with zero victim interaction
* \#91822 \[W\&A-Low] Editing the recipient during Portal transfer preparation sends the ENS name to the replaced address
* \#93104 \[W\&A-Low] Any authenticated user can bind a stranger's mailbox to their own ENS account with one click, and lock the real owner out of email notifications
* \#91487 \[W\&A-Low] api-worker: user\_channels never enforces one verified owner per email target
* \#93109 \[W\&A-Low] Missing per-user scoping in the email verification rate limit lets any signed-in account lock another user out of adding their email to notifications
* \#92250 \[W\&A-Low] Email verification can be claimed by two accounts because uniqueness is not re-checked during verification
* \#92481 \[W\&A-Low] Email-channel takeover: an unauthenticated verify endpoint finalises a pre-bound victim address
* \#92494 \[W\&A-Low] apps/portal: transfer encodes a cached ENS resolution as the recipient
* \#92544 \[W\&A-Low] Explorer's name page asserts "You are the owner of \<name>" and a "Paid" figure taken verbatim from the URL query string, on any name for any visitor, beside an ownership-free button...
* \#90069 \[W\&A-Low] Missing ownership re-check in email channel verification lets any authenticated user bind another user's email address to their own account as a verified notification channel
* \#91443 \[W\&A-Low] An unsolicited transferred name's one-byte ABI record indefinitely locks Manager's default dashboard
* \#91444 \[W\&A-Low] An unsolicited ENSv2 name with an oversized coin-121 result blocks the victim's default Manager dashboard
* \#89569 \[W\&A-Low] Email notification channels are not unique per address: an attacker can bind a victim's email to their own account, take the address out of the victim's reach, and have ENS mail del...
* \#90651 \[W\&A-Low] Unverified email channel lets any authenticated user delete another user's SendGrid contact, silently unsubscribing them from ENS broadcast email
* \#92792 \[W\&A-Low] Any account can exhaust the email-verification rate limit for an address it does not own, blocking that address's verification for an hour
* \#91631 \[W\&A-Low] Missing token-to-account binding in the unauthenticated email-verification endpoint leads to cross-account mailbox binding, victim lock-out, and an email-membership oracle
* \#92813 \[W\&A-Low] Missing re-validation in the email channel verify handler lets two accounts hold a verified channel for one address, leading to unstoppable notification delivery and destruction of ...
* \#91997 \[W\&A-Low] Manager re-authors a prior owner's social records into the recipient's replacement resolver despite a clear-old-details promise
* \#92893 \[W\&A-Low] Live ENS recipient refetch changes the undisclosed owner of a new subname
* \#92583 \[W\&A-Low] POST /notifications/channels/email/verify ships with no requireAuth and re-checks nothing, so one click on a genuine ENS email transfers a victim's address to an attacker's account ...
* \#92586 \[W\&A-Low] One ENS text record keyed on an Object.prototype member permanently replaces a name’s Manager profile page, and the Edit dialog inside it, with an error screen
* \#92739 \[W\&A-Low] Portal transfer step ID reuse redirects an ENS-name recipient transfer to the former address
* \#92305 \[W\&A-Low] Email-verification rate limiter runs after the rows it should gate and is keyed on the target address, so any account can burn a chosen email's hourly quota and block that person fr...
* \#91256 \[W\&A-Low] Unauthenticated verify endpoint binds a victim's email address to the attacker's ENS account

</details>

<details>

<summary>Insight</summary>

* \#92719 \[W\&A-Insight] A gifted ENS name with a crafted label injects persistent attacker-controlled Markdown into a victim's real Telegram expiry-alert notifications
* \#91851 \[W\&A-Insight] Explorer certifies a victim's wallet as an attacker-owned ENS name — \`/$name/address\` offers "Set primary name" for names the visitor has no relationship with
* \#92837 \[W\&A-Insight] Safe owner removal does not revoke the ENS API session they created
* \#92307 \[W\&A-Insight] DoS on API Worker from insufficient transaction payload validation
* \#90584 \[W\&A-Insight] Unescaped ENS label injected into Markdown-parsed Telegram notifications allows an attacker to place arbitrary hyperlinks in first-party ENS bot messages
* \#90636 \[W\&A-Insight] Unverified default reverse name is displayed as a verified primary name
* \#93004 \[W\&A-Insight] Explorer fabricates a "Forward match: True / Primary name" badge for an unverified, attacker-controlled reverse name
* \#92361 \[W\&A-Insight] Missing transaction-history storage limits allow one authenticated user to exhaust shared database capacity
* \#89599 \[W\&A-Insight] ENS Portal prints the namehash of a different, third-party-owned name as the viewed name's identity
* \#90662 \[W\&A-Insight] Token Info page derives the displayed Namehash from the IDNA/Punycode form of the name, showing the node of a different — and separately registrable — ENS name
* \#93024 \[W\&A-Insight] Stale Worker JWT lets a removed Safe owner modify successor notifications and read their email
* \#89371 \[W\&A-Insight] The Explorer presents an unverified \`default.reverse\` name as a forward-verified "Primary name", letting any address impersonate any name
* \#93059 \[W\&A-Insight] Missing EIP-1271 session revocation lets removed signers retain account access for seven days
* \#91702 \[W\&A-Insight] Telegram expiry notifications render an attacker-chosen link inside the official ENS bot's message
* \#89583 \[W\&A-Insight] \[Critical] Explorer renders an unverified default.reverse name as a confirmed Primary name -- the Forward match column prints True on a branch that never reads forwardMatch
* \#89441 \[W\&A-Insight] Explorer certifies an unverified reverse record as a forward-verified primary name
* \#89389 \[W\&A-Insight] Any wallet can enumerate ENS users by email address, bypass the per-address send limit, and get a victim's own expiry alerts switched off
* \#89461 \[W\&A-Insight] # Missing cleanup on an interrupted migration leaves a permanent registry-wide \`setApprovalForAll\` on the ENS v2 registry, exposing every name the wallet owns to theft
* \#89534 \[W\&A-Insight] Session enable signs attacker-authored policy hidden in an opaque salt, yielding an effectively permanent attacker-keyed session
* \#91728 \[W\&A-Insight] Explorer renders an unverified default.reverse claim as a forward-verified "Primary name" on every L2 row, letting any address be displayed as any ENS name
* \#91732 \[W\&A-Insight] Attacker-chosen ENS text-record key is rendered verbatim into the Explorer homepage activity feed
* \#91658 \[W\&A-Insight] Missing forward-match verification in reverse resolution lets any address display any ENS name as its verified primary name, causing users to send funds to the attacker
* \#89255 \[W\&A-Insight] Attacker-Controlled ENS Name Injected Into Telegram Expiry Notifications Without Escaping
* \#90859 \[W\&A-Insight] The Explorer's reverse-resolution table prints "Forward match: True" and a "Primary name" badge for a name it never forward-verified, letting any address present itself under a ...
* \#92979 \[W\&A-Insight] Telegram authentication accepts future-dated \`auth\_date\` values
* \#92546 \[W\&A-Insight] Explorer's name page asserts "You are the owner of \<name>" and a "Paid" figure taken verbatim from the URL query string, on any name for any visitor, beside an ownership-free bu...
* \#92553 \[W\&A-Insight] Explorer home page "Recent Activity" shows unverified reverse-record names as genuine ENS names (persistent content injection)
* \#89469 \[W\&A-Insight] SSRF self-host protection bypass in the OG image worker leads to temporary denial of service (resource amplification) of the card preview service
* \#92635 \[W\&A-Insight] The Explorer offers “Set primary name” based on a record the attacker controls and never checks name ownership, so one click on a link replaces the victim’s primary name with a ...
* \#92108 \[W\&A-Insight] Unescaped ENS name in api-worker Telegram/Push notification templates allows Markdown injection and a clickable phishing link inside official ENS expiry alerts
* \#92701 \[W\&A-Insight] Missing escaping in the notification worker lets a registered ENS label inject Markdown into official ENS notifications, placing an attacker-chosen link inside messages delivere...
* \#90593 \[W\&A-Insight] The Token-Info page publishes the namehash of a name's punycode A-label, so every non-ASCII ENS name shows the node of a different, separately registerable name
* \#91571 \[W\&A-Insight] Set primary name in the ENS Explorer signs setAddr() with the address from the URL, not the connected wallet, so an attacker can redirect where any ENS name resolves
* \#92796 \[W\&A-Insight] Unverified \`default.reverse\` is falsely labelled “Forward match: True” and “Primary name” on inherited L2 rows
* \#89705 \[W\&A-Insight] Missing recipient-address validation in 'Set primary name' leads to ENS name resolution hijacked to an attacker-controlled address
* \#89736 \[W\&A-Insight] Unverified \`default.reverse\` name rendered as a verified primary name on L2 rows, enabling identity spoofing
* \#89551 \[W\&A-Insight] ENS Explorer renders a forged “Forward match: True / Primary name” verification badge for an attacker’s unverified default.reverse name
* \#91962 \[W\&A-Insight] Hardcoded \`True\` in the Explorer's "Forward match" column certifies any address as the verified primary-name holder of any ENS name
* \#92105 \[W\&A-Insight] Explorer prints "Forward match: True" and "Primary name" for a \`default.reverse\` record ENS resolution reports as a mismatch
* \#92202 \[W\&A-Insight] Inheriting an L1 default.reverse name is treated as equivalent to passing forward verification, so any address is shown as the verified holder of a name it does not own, while t...
* \#92283 \[W\&A-Insight] Notification templates interpolate an ENS label with no escaping: a third party can plant persistent text in another user's Manager inbox with no interaction by that user, and a...
* \#92986 \[W\&A-Insight] ENS Worker allows Telegram Markdown injection
* \#92961 \[W\&A-Insight] Portal falsely marks an unverified inherited reverse name as a verified primary name
* \#92954 \[W\&A-Insight] Unauthorized Profile Image Change. Upload Does Not Verify Signer Ownership
* \#90095 \[W\&A-Insight] One signature on a legitimate app.ens.domains screen redirects a victim's ENS name to the attacker — the app builds the redirect from an attacker-chosen unverified record and ne...
* \#89300 \[W\&A-Insight] Patched \`@rhinestone/sdk\` identity is undocumented in source — two provenance comments assert two different, superseded SHA-256 values
* \#92706 \[W\&A-Insight] Telegram Markdown Injection in Notification Delivery — Unescaped ENS Name Enables Phishing via the Official Notifications Bot
* \#91839 \[W\&A-Insight] An attacker-gifted v1 name modifies the arguments of the migration transaction set the victim signs, adding an approval that would not otherwise be requested and widening a toke...
* \#90646 \[W\&A-Insight] Explorer renders an unverified \`default.reverse\` claim as a verified Primary Name, letting any address assert any ENS name (e.g. \`vitalik.eth\`) with a green "Forward match: True...
* \#92439 \[W\&A-Insight] Unescaped ENS name in Telegram expiry-notification template enables persistent Markdown/link injection reaching third-party users
* \#89464 \[W\&A-Insight] ENS Explorer: reverse-resolution certifies an unverified \`default.reverse\` name as the verified L2 primary name
* \#89513 \[W\&A-Insight] Missing Markdown escaping in notification worker allows attacker author ENS-branded Telegram message content, or silently suppress a victim's expiry warnings
* \#92445 \[W\&A-Insight] Telegram notification Markdown injection via attacker-controlled ENS name
* \#92878 \[W\&A-Insight] Persistent Markdown/content injection into Telegram expiry-notification messages via an unsanitized ENS name label

</details>

## Reports by Type

<details>

<summary>Websites &#x26; Applications</summary>

* \#90165 \[W\&A-Critical] Resolver role removal merges name-scoped permissions and leaves a delegate authorized
* \#91637 \[W\&A-Critical] Manager profile page renders an attacker-controlled receiving address under a genuine name's verified owner identity
* \#91641 \[W\&A-Critical] Explorer resolver-role editor writes grants and revokes to a wrongly-derived EAC resource, so revoking a delegate's permissions silently leaves them in place
* \#92818 \[W\&A-Critical] Missing label validation in the Portal's Configure Registry flow leads to subregistry detachment of a different, real ENS name
* \#92196 \[W\&A-High] Portal keys address-record identity by display symbol: selecting an oversized same-symbol alias deletes the owner's Base Sepolia record and resolves the name to a prior owner's add...
* \#92719 \[W\&A-Insight] A gifted ENS name with a crafted label injects persistent attacker-controlled Markdown into a victim's real Telegram expiry-alert notifications
* \#92112 \[W\&A-Critical] A direct Registry URL applies Registry bit 0 to a resolver and grants address control
* \#92825 \[W\&A-Medium] Missing registry scoping in the role History query lets anyone write permanent forged role-change entries into any ENS name's audit trail
* \#92899 \[W\&A-Medium] Stale V1 resolver snapshot during migration reinstalls a removed attacker-operated resolver
* \#92902 \[W\&A-Critical] Portal treats an empty ETH record as a V1 controller resolution and transfers to the wrong recipient
* \#91851 \[W\&A-Insight] Explorer certifies a victim's wallet as an attacker-owned ENS name — \`/$name/address\` offers "Set primary name" for names the visitor has no relationship with
* \#92905 \[W\&A-Low] Live resolver-access refetch changes "Replace & Continue" into a transaction to the previous owner's resolver
* \#90320 \[W\&A-Critical] Removed resolver user can still redirect ENS names
* \#89366 \[W\&A-Medium] Missing execution-time owner re-verification in v1→v2 migration leads to ROLE\_SET\_RESOLVER granted to a former manager
* \#91689 \[W\&A-Medium] Portal direct child route and spoofable resolver authentication substitute an attacker contract as the victim's transaction target
* \#89548 \[W\&A-Critical] Registration and renewal use un-normalized labels: the name the Manager shows is not the name it writes on chain
* \#92880 \[W\&A-Critical] Explorer transfers a user's ENSv2 name to a separate address the seller selected before the sale
* \#92881 \[W\&A-Critical] ENSv1 registrant/manager conflation redirects ENSv2 name transfers to a non-owner manager
* \#92887 \[W\&A-Critical] Incomplete mixed resolver-role removal leaves a former delegate able to change an ENS address record
* \#92889 \[W\&A-Critical] Manager migration grants a seller-selected address resolver authority over the buyer's ENSv2 name
* \#92892 \[W\&A-Critical] Remove user revokes only UI-recognized role bits and can leave official authority active
* \#92890 \[W\&A-Critical] \[HIGH] Manager authorizes record edits via the resolver's stale internal roles instead of registry ownership — a name's previous owner keeps full write access (incl. the coins\[...
* \#91768 \[W\&A-Critical] "Remove user from all roles" preserves global resolver authority
* \#92837 \[W\&A-Insight] Safe owner removal does not revoke the ENS API session they created
* \#91450 \[W\&A-Low] Unbounded \`links\` rendering blocks Manager navigation for 14 seconds on a gifted name
* \#91446 \[W\&A-Low] Quadratic primary-contact deduplication temporarily locks the victim's Manager tab after an unsolicited name transfer
* \#91717 \[W\&A-Critical] Explorer transfer flow moves a different ENS name than the UI and wallet prompt display, because on-chain identity is derived from the normalized label
* \#89253 \[W\&A-Critical] Wrong-resource revocation in Resolver Roles lets a removed account retain \`ROLE\_SET\_ADDR\` and redirect another ENS name
* \#90307 \[W\&A-Low] Explorer renders an unbounded on-chain uint64 expiry through Temporal without a range check, so one unsolicited subname permanently breaks a victim's names page
* \#91837 \[W\&A-Critical] Manager claims a primary name the user does not own: the reverse registrar receives the normalized name while every displayed surface and the forward addr(60) record use the ra...
* \#92278 \[W\&A-Critical] Both renewal routes prove one ENS name is renewable and then bill the user to renew a different one
* \#91224 \[W\&A-Critical] Explorer renew/transfer encode the on-chain write from a normalizing helper while the ownership gate and displayed transaction use the raw name — a victim renews or transfers a...
* \#90639 \[W\&A-Critical] Transfer authorises the raw label, moves the normalized label's token
* \#91913 \[W\&A-Medium] Incomplete \`KNOWN\_PUBLIC\_RESOLVERS\` skip of locked-record safety wipes ETH/text records on Manager v1→v2 Upgrade
* \#92307 \[W\&A-Insight] DoS on API Worker from insufficient transaction payload validation
* \#89314 \[W\&A-Critical] Manager V1 to V2 migration silently keeps an attacker's resolver and grants the attacker \`ROLE\_SET\_RESOLVER\` on the V2 registry for gifted/secondary-market names
* \#92165 \[W\&A-Low] Unauthenticated /verify endpoint binds a victim's email to the attacker's ENS account
* \#92867 \[W\&A-Medium] Failed HCA reveal is accepted from attacker-created registration state, leaving hidden resolver authority and an incorrect receiving address
* \#92870 \[W\&A-Medium] Case-folded role-name lookup in the Manager profile leads to persistent injection of third-party ENS names on any address's public page
* \#91834 \[W\&A-High] Missing resolver-ownership validation in the name-history query leads to persistent forged content injection on any ENS name's history page
* \#92645 \[W\&A-Critical] Reused resolver-role row state makes a Content Hash edit grant another delegate resolver-wide address control
* \#90446 \[W\&A-Medium] No per-user push-channel cap and faulty retry after a partial commit permanently suppress other users' external expiry reminders
* \#92914 \[W\&A-Critical] Missing ENSIP-15 normalization check on the primary-name render path lets an attacker's address be displayed as another party's name
* \#92658 \[W\&A-Critical] Manager migration grants a former V1 controller resolver authority over the buyer’s V2 name
* \#91855 \[W\&A-Critical] Portal treats an unresolved resolver-role scope as ROOT, enabling cross-name resolution hijacking
* \#92822 \[W\&A-Medium] Explorer copies a preserved ENSv1 resolver into a new ENSv2 subname, letting an ENSv1 party choose the address a user's transfer lands on
* \#92885 \[W\&A-Medium] Stale V1 Manager is encoded as the nested V2 grantRoles recipient after a same-ID refresh
* \#92908 \[W\&A-Medium] Historical resolver provenance check reattaches attacker-upgraded code during profile editing
* \#92868 \[W\&A-Low] Unauthenticated /notifications/channels/email/verify grants authority by bare token possession — cross-user email-channel hijack and permanent victim lockout
* \#90584 \[W\&A-Insight] Unescaped ENS label injected into Markdown-parsed Telegram notifications allows an attacker to place arbitrary hyperlinks in first-party ENS bot messages
* \#91765 \[W\&A-Medium] Stale V1 migration state resurrects a revoked manager on V2
* \#92842 \[W\&A-Critical] A name-scoped role grant gives a limited registrar root resolver authority
* \#92877 \[W\&A-Critical] ENS Explorer’s “Remove user” action can leave global resolver permissions active
* \#89249 \[W\&A-Medium] Stale authorization in V1-to-V2 Manager migration resurrects a revoked manager and lets a former seller hijack the ENS resolver
* \#92555 \[W\&A-Critical] Name transfer never revokes the seller's resolver roles, letting the former owner rewrite the buyer's ETH-address record and redirect resolution to an attacker (name hijack / a...
* \#90161 \[W\&A-Critical] Explorer authorizes a transfer against the name in the URL but moves the token of the normalized name, so disposing of an unsolicited look-alike transfers the victim's real name
* \#93124 \[W\&A-Low] An unauthenticated attacker can recover a victim's SIWE nonce verbatim and burn it before any verification, locking the victim out of login
* \#92913 \[W\&A-Medium] A keep-resolver sale strands the sold name under the seller's resolution control -- and if the seller burns the resolver role first, no one can ever undo it
* \#90636 \[W\&A-Insight] Unverified default reverse name is displayed as a verified primary name
* \#93130 \[W\&A-Medium] A stale Manager migration plan can restore a former owner's attacker-controlled resolver
* \#91095 \[W\&A-Critical] Portal “Remove user from all roles” sends a one-name revoke and leaves root resolver authority usable
* \#93004 \[W\&A-Insight] Explorer fabricates a "Forward match: True / Primary name" badge for an unverified, attacker-controlled reverse name
* \#92947 \[W\&A-High] Registration checkout presents the rent as the exact total while the wallet signs a permit for \~2.5× that amount
* \#93128 \[W\&A-High] manager HCA quote failure can authorize more USDC than the exact wallet Total shown at checkout
* \#92944 \[W\&A-Critical] Recipient resolution returns a different address than every conformant ENS client, sending an irreversible name transfer to whoever registered a look-alike name
* \#92945 \[W\&A-Critical] # \`/renew/$name\` displays the victim's expiry but signs \`renew()\` for the attacker's label the victim pays to extend someone else's name
* \#89967 \[W\&A-Medium] Manager executes cached V1 migration state, restoring revoked resolver authority and redirecting ENS payments
* \#92950 \[W\&A-Low] Portal create-subname reuses stale ENS inputs, minting to the wrong owner or an attacker-controlled resolver
* \#92951 \[W\&A-Medium] Missing on-chain validation in ENS migration lets a third party hijack the resolver of a user's name
* \#93135 \[W\&A-Critical] Unresolved record-part role rows silently grant global resolver authority
* \#93125 \[W\&A-Low] Missing authorization in the notifications channel-delete handler's broadcast-cleanup lets any authenticated user delete another user's SendGrid marketing contact (config-gated).
* \#93127 \[W\&A-Critical] ENS Manager profile page reads ownership from the normalized name and records from the raw URL name, letting an attacker show their own receiving address on a victim's profile
* \#93133 \[W\&A-Medium] Stale V1 migration state restores a revoked manager and attacker resolver in ENSv2
* \#91396 \[W\&A-Critical] An IOST-only delegate can replace Manager's copied Solana recipient without modifying the SOL record
* \#92955 \[W\&A-Critical] Scope substitution in the resolver role editor converts an unresolvable name-scoped role into a resolver-root grant, giving unintended global authority over every name on the r...
* \#93129 \[W\&A-Critical] Stale role state tied to the row position in the resolver Roles sheet quietly adds root ROLE\_SET\_ADDR to a grant, so ENS names resolve to an attacker-controlled address
* \#89465 \[W\&A-Critical] Lookalike-label renewal trap: /renew displays the normalized ENS name but renews the raw label — victim pays to renew an attacker's name, then loses their own
* \#92836 \[W\&A-Medium] Payment-Token Picker Silently Treats a Failed Price Read as $0, Hiding Insufficient-Balance and Skipping the Real Approval Amount
* \#90264 \[W\&A-Low] Unauthenticated \`POST /wallet/fund\` spends a server-held private key behind a non-atomic KV "lock", allowing concurrent funds to collide on the funder's nonce and to be issued witho...
* \#92964 \[W\&A-Medium] Manager migration can restore a revoked V1 manager’s permission to change the resolver
* \#92966 \[W\&A-Low] Deleting an unverified email channel can remove another user's SendGrid contact
* \#92962 \[W\&A-Critical] Resolver role editor revokes the wrong scope
* \#92967 \[W\&A-Critical] The Manager checks name ownership on one node and writes to another
* \#92971 \[W\&A-Critical] Manager falsely confirms a canonical ENS renewal while extending an attacker-owned raw Unicode name
* \#92977 \[W\&A-Medium] Missing registry filter in useRoleHistory lets any user plant fake role grants in another name's Explorer History panel
* \#92976 \[W\&A-Medium] Stale V1 manager state during ENSv2 migration resurrects former-manager resolver authority and enables name hijacking
* \#91233 \[W\&A-Low] Manager suppresses a cross-name avatar/header record update and preserves former-owner media control
* \#92833 \[W\&A-High] A third party can brick a wrapped V1 ENS name for 99 years through Manager renewal
* \#93132 \[W\&A-Medium] Explorer stamps a verified "Forward match: True / Primary name" badge on an unverified default.reverse name, so any address displays a name it does not own as its confirmed prima...
* \#92361 \[W\&A-Insight] Missing transaction-history storage limits allow one authenticated user to exhaust shared database capacity
* \#92990 \[W\&A-Critical] Cross-account receipt reuse skips resolver-role revocation and makes Portal render an attacker-controlled ENS address
* \#90056 \[W\&A-Medium] TOCTOU in V1-to-V2 migration restores revoked manager resolver authority
* \#90071 \[W\&A-Critical] Lossy resolver-role grouping makes "Remove user from all roles" leave global ROLE\_SET\_ADDR active
* \#89599 \[W\&A-Insight] ENS Portal prints the namehash of a different, third-party-owned name as the viewed name's identity
* \#89601 \[W\&A-Critical] Missing ENSIP-15 normalisation in the Manager registration path causes users to pay for and receive a different, permanently unresolvable name than the one the app showed as av...
* \#92373 \[W\&A-Medium] Same-ID migration refresh retains removed resolver authority in an executable plan
* \#89630 \[W\&A-Critical] Renewal payment is redirected to an attacker's look‑alike name
* \#90931 \[W\&A-Critical] Resolver row-index reuse carries a removed user's root Upgrade permission into another collaborator and enables shared-resolver takeover
* \#90693 \[W\&A-Critical] The Manager profile page reads records from the raw URL name and ownership from its ENSIP-15 name, so an attacker-controlled address record is rendered under a real name's owner
* \#92993 \[W\&A-Critical] Explorer Send name checks the name in the URL and signs away the owner's real name
* \#92398 \[W\&A-Medium] Unscoped, Replay-Unprotected SendGrid Webhook over Duplicate Email Rows Lead To Silent Cross-User Disabling of ENS Email Notifications
* \#89467 \[W\&A-Critical] Inconsistent name normalization between the renewal checks and the renewal transaction leads to the user signing a renewal for an attacker-controlled name
* \#92995 \[W\&A-Medium] A Contact-tab Discord edit republishes inherited social handles through Manager's resolver replacement
* \#89381 \[W\&A-Medium] Migration can restore resolver authority to a former V1 manager after reclaim
* \#89483 \[W\&A-Critical] Empty resolvedNames falls back to root scope, so editing one role row signs grantRootRoles over every name on the resolver
* \#91900 \[W\&A-Critical] Explorer resolves a recipient ENS name that has no addr(60) record to the owner of its ENS v2 registry entry, so one ordinary paid v2 registration by a stranger redirects a nam...
* \#89270 \[W\&A-Critical] Manager profile page resolves records from the raw URL name but owner/expiry from the normalized name, letting an attacker display their own address under a victim name's identity
* \#92408 \[W\&A-Low] Email verification endpoint has no authentication, letting an attacker bind a victim's email address to the attacker's account
* \#92097 \[W\&A-Medium] Manager migration resurrects a removed attacker-controlled resolver and re-grants a relinquished manager role from a stale V1 snapshot
* \#92411 \[W\&A-Critical] Incorrect labelhash resolution in Portal leads to deletion of other unintended ENS subnames, and resolving hijacking of unintended
* \#92413 \[W\&A-High] Unpinned Warp settlementLayers and dest-only 0x03 session policy lead to direct theft of Hybrid Custody Account funds
* \#91925 \[W\&A-Low] Email verification binds another user's mailbox to the requesting account, unauthenticated
* \#91931 \[W\&A-Medium] Manager migration reauthorizes a revoked V1 manager, enabling resolver hijack
* \#89286 \[W\&A-Low] Unauthenticated POST /wallet/fund Enables Gas-Griefing Against the Faucet's Own Funding Wallet
* \#89387 \[W\&A-Critical] "Remove user from all roles" leaves root resolver authority active, enabling ENS address redirection
* \#89563 \[W\&A-Critical] ENS Explorer resolver "Remove user" / Save silently leaves the root (all-names) grant intact, so a removed manager keeps write access to every name
* \#91690 \[W\&A-Medium] Manager migration restores a revoked V1 manager with persistent V2 resolver authority
* \#91098 \[W\&A-Critical] Renew page shows the correct name but signs a different one — invisible unicode causes wrong-recipient renewal
* \#92510 \[W\&A-Critical] Portal authorizes an attacker-gifted raw-NFD name but transfers the victim's distinct NFC ERC-1155 name token
* \#91974 \[W\&A-Medium] SendGrid webhook resolves notification channels by email alone, so a cross-user duplicate is not ownership-bound when provider events are processed
* \#91977 \[W\&A-Medium] Explorer's V2 transfer never revokes third-party roles, so a delegate the seller kept can re-point the buyer's resolver
* \#90944 \[W\&A-Medium] Portal transfer cleanup preserves third-party authority that can hijack the recipient's name
* \#92111 \[W\&A-Critical] Portal authorizes a fullwidth raw ENSv2 name but transfers the victim's ASCII canonical twin NFT
* \#90951 \[W\&A-Critical] Portal "Remove user from all roles" submits incomplete revocation calldata and leaves the delegate authorized
* \#89611 \[W\&A-Critical] Resolver “Remove user” revokes only one name and leaves global address/upgrade authority active
* \#89761 \[W\&A-Critical] Removing a resolver user revokes only one displayed ENS name, allowing redirected incoming payments
* \#89990 \[W\&A-Critical] Resolver roles page revokes a single name while claiming removal of all roles — removed users keep live record-write authority on other names and ROOT
* \#90784 \[W\&A-Critical] Critical: Portal account-only role grouping collapses resolver scopes, leaving an offboarded collaborator able to redirect ENS resolution
* \#92860 \[W\&A-Critical] Missing grantee validation in the v1→v2 migration's "manager restoration" grants an attacker ROLE\_SET\_RESOLVER over a victim's migrated ENS name, hijacking where the name resolves
* \#89336 \[W\&A-Critical] V1-to-V2 migration grants stale owners resolver control
* \#90498 \[W\&A-Critical] Resolver role table merges different names into one row but signs calldata for only one name
* \#90725 \[W\&A-Critical] Encoded-labelhash confusion in Portal deletion unregisters a different victim-owned subname and enables registrar takeover
* \#90622 \[W\&A-Critical] Lossy resolver-role aggregation causes “Remove user from all roles” to retain global permissions, enabling ENS record hijacking
* \#92222 \[W\&A-Critical] Portal “Remove user” leaves root text authority, allowing a former delegate to overwrite another owner's profile
* \#92094 \[W\&A-Critical] \[Critical] Explorer role removal preserves global ROLE\_SET\_ADDR, enabling post-offboarding ENS address diversion
* \#90637 \[W\&A-Critical] Removing a resolver user leaves every root-scoped role in place
* \#91757 \[W\&A-Critical] Resolver role aggregation mis-scopes removal and preserves global authority
* \#91824 \[W\&A-Medium] V1→V2 migration resurrects a revoked legacy manager with fresh \`ROLE\_SET\_RESOLVER\` after the victim reclaims the V1 node
* \#92465 \[W\&A-Critical] Owner/controller role confusion in the Explorer's address-resolution fallback (resolveEnsOwner) leads to irreversible transfer of the user's ENS name to an unintended recipient
* \#90957 \[W\&A-Medium] V1-to-V2 migration can reinstall a revoked attacker-controlled resolver after Manager receives corrected V1 state
* \#91602 \[W\&A-Critical] Incorrect privilege assignment in the v1 to v2 migration leads to resolver hijack and loss of funds
* \#90239 \[W\&A-Medium] Revoked V1 managers regain persistent V2 resolver control due to stale manager state during migration
* \#92416 \[W\&A-Critical] Mixed-case transfer route sends the lowercase sibling ENS NFT
* \#91127 \[W\&A-Medium] Unbounded favorite fan-out lets an attacker poison shared expiry batches and suppress cross-user notifications
* \#90894 \[W\&A-Critical] apps/portal resolver roles UI: "Remove user" revokes only the name scope while confirming removal from all roles; the root grant stays active
* \#90254 \[W\&A-Medium] Migration-plan cache key omits V1 authority fields, restoring an attacker resolver
* \#90896 \[W\&A-Critical] apps/portal: "Remove user" confirms removal from all roles; the approved transaction revokes one name scope
* \#91097 \[W\&A-Critical] Explorer's “Remove user from all roles” signs a one-resource revoke, leaving a former delegate able to redirect another ENS name
* \#91333 \[W\&A-Critical] Portal's “Remove user from all roles” submits a one-resource revocation and leaves the removed delegate's root resolver authority active
* \#91449 \[W\&A-Critical] Broken Access Control: Multi-Resource Role Aggregation in Resolver Roles Management Causes Incomplete Revocation and Persistent Root-Level Access
* \#91848 \[W\&A-Critical] Editing a record-scoped resolver delegate silently submits \`grantRootRoles\` — authority over every name on the resolver
* \#91921 \[W\&A-Critical] Granular Resolver Role Escalation to Root
* \#93006 \[W\&A-Medium] \[MEDIUM] Portal registration checkout fails open on a single transient price-read failure — the UI freezes at $0.00 and drops every approval step while the machine charges the li...
* \#92128 \[W\&A-Critical] Inconsistent ENS name normalization in the Manager profile page renders a trusted name's owner beside an attacker-controlled receiving address, leading to theft of user funds
* \#92429 \[W\&A-Critical] The Explorer's "Remove user" filters the root scope out of the resource it revokes, so an account the operator removed keeps write authority over the address record of every na...
* \#89254 \[W\&A-Medium] Cross-User Notification-Channel Disablement via Unscoped \`target\` Predicate
* \#92239 \[W\&A-Critical] One ticked permission on a record-scoped row makes the Explorer's Roles page sign grantRootRoles — authority over every name the resolver serves — with no scope shown on any sc...
* \#92257 \[W\&A-Critical] Resolver role editor upgrades a name-scoped grant to resolver root, submitting a broader transaction than the operator selected
* \#93010 \[W\&A-Critical] Profile page reads ownership and records from two different names, so a look-alike name displays an attacker-controlled receiving address
* \#93012 \[W\&A-Critical] Explorer transfer flow checks ownership of one name and signs transactions for a different one
* \#90250 \[W\&A-Medium] Missing V1 Registry owner revalidation re-authorizes a revoked manager during V2 migration, enabling resolver hijacking
* \#91813 \[W\&A-Low] Email verification never identifies the requesting wallet, so an attacker binds a victim's mailbox to their own account
* \#89285 \[W\&A-Medium] Fresh V1 owner updates do not invalidate a migration plan, re-authorizing a revoked manager
* \#93018 \[W\&A-Critical] Raw-cased profile URL renders the real owner over an attacker resolver's address
* \#90662 \[W\&A-Insight] Token Info page derives the displayed Namehash from the IDNA/Punycode form of the name, showing the node of a different — and separately registrable — ENS name
* \#89293 \[W\&A-Critical] Explorer resolver "Remove user" confirms a full revocation and signs a single-resource one, leaving the target holding root \`ROLE\_SET\_ADDR\`
* \#90981 \[W\&A-Medium] A former owner will redirect a transferred ENS name to an attacker-controlled address for the new owner by retaining PermissionedResolver write authority.
* \#90980 \[W\&A-Medium] Manager migration can grant a revoked V1 manager persistent V2 resolver authority, enabling ENS name hijacking
* \#93019 \[W\&A-Medium] ENS Manager regrants a revoked V1 manager during migration, enabling resolver hijack
* \#90626 \[W\&A-Medium] Manager migration restores a removed V1 manager, enabling resolver hijack and fund theft
* \#93021 \[W\&A-High] Swallowed budget-quote failure makes the registration checkout show the rent as an exact total while the machine sizes the funding permit from the full budget, so the user approves...
* \#89891 \[W\&A-Low] /notifications/channels/email/verify verifies whoever's token you send, not whoever asks
* \#93024 \[W\&A-Insight] Stale Worker JWT lets a removed Safe owner modify successor notifications and read their email
* \#90755 \[W\&A-Medium] Cached V1 manager authority is restored as a persistent V2 resolver role during migration
* \#93029 \[W\&A-High] Manager writes L2 address records to mainnet-derived coin types while the Explorer reads Sepolia-derived ones, so all five L2 rows render the owner's coin-60 address and a sender's...
* \#89398 \[W\&A-Critical] Missing address-record validation in resolveAddressOrName causes irreversible ENS name transfers to the wrong recipient
* \#93030 \[W\&A-Critical] Broken scope derivation in the Portal resolver roles editor leads to resolving ENS names to an attacker-controlled address
* \#89352 \[W\&A-Medium] V1 to V2 migration silently replays prior-owner resolver records, pointing the victim's migrated name at the attacker
* \#93031 \[W\&A-Low] Missing cross-user authorization on the email-channel verify endpoint lets an attacker bind any third party's email to their own ENS account, disabling that user's own notifications...
* \#93032 \[W\&A-Medium] Stale V1 manager snapshot restores revoked resolver authority during migration
* \#91600 \[W\&A-Medium] Manager executes a cached migration plan after V1 authority revocation, restoring the revoked manager and resolver
* \#91852 \[W\&A-Medium] Former v1 manager is incorrectly granted V2 resolver authority, allowing ENS redirection and NFT theft
* \#89356 \[W\&A-Medium] TOCTOU in V1-to-V2 manager validation enables ENS resolver hijacking by a revoked manager
* \#92155 \[W\&A-Critical] Renewal flow checks and displays one ENS name but signs renew() calldata for a different one, so the user pays to extend a name they were never shown
* \#91645 \[W\&A-Critical] Wrong-field ENS ownership resolution in the Explorer's recipient input sends name transfers to a v1 name's former controller instead of its owner
* \#90206 \[W\&A-Critical] The transfer flow authorises against one ENS name and signs the transfer of a different one
* \#91001 \[W\&A-High] The registration checkout shows the rent as the total while no funding budget has been fetched, and admits the click on that figure
* \#91924 \[W\&A-High] Manager can display a lower registration total and execute a higher HCA spend without reconfirmation
* \#90770 \[W\&A-Critical] "Remove user from all roles" revokes one resolver scope and leaves address-control authority active
* \#89408 \[W\&A-Critical] The Manager profile page asks the attacker's resolver for the victim's node, and renders the answer under the victim's real ownership and expiry
* \#93040 \[W\&A-Medium] Stale ENSv1 manager restoration during migration grants a revoked attacker ENSv2 resolver control
* \#93042 \[W\&A-Medium] Migration resurrects a revoked V1 manager from stale indexed state
* \#93043 \[W\&A-Critical] ERC1271 signature-mode confusion lets a bounded HCA session drain the account's refund token
* \#93045 \[W\&A-Critical] Explorer resolves a recipient ENS name to the ENS v1 registry *controller*, so a retained-controller name silently redirects name transfers, subname ownership and role grants t...
* \#93046 \[W\&A-High] Unchecked registration subregistry lets an HCA session signer retain control of victim-owned subnames
* \#91004 \[W\&A-Critical] The Explorer transfer flow checks ownership of the name in the URL and moves its normalized sibling
* \#90094 \[W\&A-Critical] Renewing a name from the portal confirm screen builds the transaction for a different on-chain name than the one it displays — the payment renews \`normalize(name)\`, not the nam...
* \#91337 \[W\&A-Critical] v1 to v2 migration silently grants the V1 registry controller resolver authority over the buyer's name
* \#91335 \[W\&A-Critical] Portal applies the Registry role schema to a PermissionedResolver: one ordinary "Registrar" Add User grant becomes global Set Address authority
* \#92028 \[W\&A-Critical] Manager renew signs a transaction for a look-alike name while the UI shows the victim’s real name
* \#89278 \[W\&A-Critical] Portal displays raw ALICE.eth but transfers the distinct canonical alice.eth NFT
* \#93051 \[W\&A-Medium] Default clean transfer preserves a prior resolver delegate, allowing the recipient's ENS name to be hijacked
* \#89363 \[W\&A-Critical] ENS v1→v2 migration grants resolver authority over the migrated name to an unverified, never-displayed third party
* \#90245 \[W\&A-Critical] Explorer "Remove user" confirms removal from all roles but only revokes the name scope; root roles keep resolver-wide addr-write
* \#89593 \[W\&A-Critical] Incorrect scope inference in the resolver roles editor leaves a removed user's resolver-wide grant live, up to ROLE\_UPGRADE
* \#90190 \[W\&A-Critical] Manager bulk renewal displays the raw label but renews the normalized one, so a user's rent payment extends a name they do not own
* \#91342 \[W\&A-Medium] Manager resurrects a relinquished V1 manager as a V2 resolver delegate after the pre-wallet state check
* \#89371 \[W\&A-Insight] The Explorer presents an unverified \`default.reverse\` name as a forward-verified "Primary name", letting any address impersonate any name
* \#89374 \[W\&A-Medium] Stale V1 Registry state reinstalls a revoked attacker resolver during V1-to-V2 migration, enabling ENS name hijacking
* \#91014 \[W\&A-Critical] Scoped HCA sessions are downgraded to owner mode, bypassing refund authorization and transferring HCA tokens
* \#92042 \[W\&A-Critical] ENSv2 migration grants ROLE\_SET\_RESOLVER on migrated names to a subgraph-sourced "manager" address it never verifies on-chain or shows the user (resolver hijack)
* \#92043 \[W\&A-Critical] Explorer renewal sends \`renew()\` the ENSIP-15-normalized label while every rendered string is the raw on-chain label: the user pays full rent, their own name gains zero seconds...
* \#91648 \[W\&A-High] HCA funding permit is sized from an unbounded orchestrator quote with no ceiling or cross-check
* \#92121 \[W\&A-Critical] Manager renewal signs a different ENS registration from the one the page describes: the route parameter is ENSIP-15 normalized for the expiry it displays and only lower-cased f...
* \#93059 \[W\&A-Insight] Missing EIP-1271 session revocation lets removed signers retain account access for seven days
* \#91698 \[W\&A-Critical] Crafted /renew URL renews an attacker's look-alike name while the page displays the victim's own name and on-chain expiry
* \#92775 \[W\&A-Critical] Manager renders a lookalike ENS name with the victim's identity above the attacker's receiving address
* \#92820 \[W\&A-Critical] Explorer: "Remove user" / "Save" on a resolver operator revokes a single scope while the row aggregates root and name scopes, so root-level roles survive a "remove this user fr...
* \#93056 \[W\&A-Medium] Migration replays attacker controlled records from a removed cached V1 resolver into the authoritative V2 resolver
* \#91702 \[W\&A-Insight] Telegram expiry notifications render an attacker-chosen link inside the official ENS bot's message
* \#89460 \[W\&A-Critical] Every ENS registration overwrites the previous one's transaction history
* \#93060 \[W\&A-Low] Attacker initiated verification binds a victim email to the attacker
* \#92978 \[W\&A-Critical] Portal transfer flow authorises ownership of one ENS node and then executes irreversible writes against a different one
* \#92988 \[W\&A-Medium] Missing live V1 owner check re-grants resolver control to a revoked manager during migration
* \#92994 \[W\&A-Medium] Stale V1 migration data restores an ex-manager's resolver and permanent V2 authority after a split handoff
* \#93044 \[W\&A-Critical] The v1→v2 migration automatically grants \`ROLE\_SET\_RESOLVER\` on the newly migrated v2 name to the legacy v1 registry controller — an address that is, by construction, never the...
* \#93049 \[W\&A-Critical] Missing resolver contract-type validation turns record-role delegation into registry-wide name theft
* \#93001 \[W\&A-High] Any contract can write persistent fabricated rows - including a forged "Set address to 0xAttacker" - into any ENS name's History page: the v1 history query has no resolver-provenan...
* \#92997 \[W\&A-Medium] Missing factory check in the Permissioned Resolver lookup lets any contract get ENS's "audited and considered secure" badge on its page and OG card
* \#93071 \[W\&A-Critical] Portal reuses a completed resolver step after recipient correction, leaving the transferred name resolving to the prior recipient
* \#89583 \[W\&A-Insight] \[Critical] Explorer renders an unverified default.reverse name as a confirmed Primary name -- the Forward match column prints True on a branch that never reads forwardMatch
* \#93053 \[W\&A-Medium] Failed per-token price read renders a selectable $0.00 quote, skips the approval leg, and the amount-less renew silently charges the live price against the standing 2x allowance
* \#89431 \[W\&A-Critical] Stale transaction success falsely completes later role revocations, leaving resolver authority active
* \#89433 \[W\&A-Critical] Transfer flow never resets the transaction manager, so a stale step actor from an abandoned attempt is matched by id to the next attempt and the steps before it are skipped whi...
* \#93007 \[W\&A-Medium] Explorer address page has a visitor's wallet sign setName and take the attacker's name as primary
* \#93073 \[W\&A-Critical] Explorer "Remove user" on a resolver role holder with root plus named roles only revokes one named scope, leaving the root role active
* \#91485 \[W\&A-Critical] Missing ENS normalization in the Manager renewal flow leads to a victim extending the attacker's name with their own funds
* \#89420 \[W\&A-Critical] Resolver roles sidebar collapses an account's multiple role resources into one write scope, so "Remove user" displays one scope and writes another, leaving removed accounts wit...
* \#89441 \[W\&A-Insight] Explorer certifies an unverified reverse record as a forward-verified primary name
* \#89444 \[W\&A-Low] Any anonymous caller can drain the faucet hot wallet and hang worker invocations
* \#89445 \[W\&A-Medium] One user's bounce or unsubscribe rewrites every other user's channel with the same target
* \#92565 \[W\&A-Critical] The Explorer's Extend flow renders the raw indexer name on every surface but signs the ENSIP-15-normalized label, so a renewal reached from the address names table pays to exte...
* \#92564 \[W\&A-Critical] apps/portal: Remove user revokes only the root grant it promised to clear entirely
* \#92567 \[W\&A-Critical] Bulk renew displays the raw name from the indexer but signs the normalized label, so a victim's stablecoins extend a registration they do not own
* \#92568 \[W\&A-Critical] Profile editing authorizes the normalized name but encodes the raw route param, so a crafted URL makes the victim sign a record write to an attacker's node on an attacker's res...
* \#89389 \[W\&A-Insight] Any wallet can enumerate ENS users by email address, bypass the per-address send limit, and get a victim's own expiry alerts switched off
* \#92570 \[W\&A-Critical] Set primary name writes its forward legs against the raw name and signs the reverse leg against the normalized name, so one click points the victim's reverse record at a name t...
* \#92572 \[W\&A-Critical] ENSv2 registration applies no ENSIP-15 normalization at any step, so a crafted URL prices, displays and registers a label the user cannot see and that ENS does not consider par...
* \#92575 \[W\&A-Low] Any authenticated user can claim a third party's email address as a notification channel, and the victim's own click on the ENS-sent verification link locks them out of using their ...
* \#92578 \[W\&A-Medium] Notification channel state is read and written by target address alone, with no user\_id predicate, over a schema that deliberately allows the same target on multiple accounts
* \#93079 \[W\&A-Medium] A stale Manager migration plan can restore a former owner's attacker-controlled resolver
* \#92177 \[W\&A-Medium] Stale V1 manager snapshot re-grants a former manager V2 resolver authority during migration
* \#92178 \[W\&A-Medium] ENS migration can restore resolver authority to a revoked manager from stale subgraph data
* \#90794 \[W\&A-Critical] Explorer confirms an uppercase ENSv2 transfer but signs the distinct lowercase token ID, causing direct NFT theft
* \#93081 \[W\&A-Critical] Resolver-roles sidebar coerces an un-invertible resource hash to the empty string, which the save path reads as ROOT, so an operator scoping a grant to one name silently grants...
* \#93080 \[W\&A-Critical] Improper resolver-role grouping causes incomplete revocation and attacker-controlled ENS resolution
* \#92205 \[W\&A-Medium] Stale V1 authority cache restores attacker resolver control during ENSv2 migration
* \#89461 \[W\&A-Insight] # Missing cleanup on an interrupted migration leaves a permanent registry-wide \`setApprovalForAll\` on the ENS v2 registry, exposing every name the wallet owns to theft
* \#89462 \[W\&A-High] HCA registration budget omits the resolver deployment, so a first registration is funded short and its commitment cannot be recovered
* \#90255 \[W\&A-Critical] apps/portal resolver roles UI: "Remove user" revokes only the name scope while confirming removal from all roles; the root grant stays active
* \#91150 \[W\&A-Critical] Explorer replaces the displayed ENSv2 asset's ERC-1155 tokenId before the connected-wallet transfer
* \#90807 \[W\&A-Medium] Stale authorization in V1-to-V2 manager restoration re-grants revoked resolver control, enabling ENS name hijack
* \#92064 \[W\&A-Critical] Incorrect EAC resource scoping in the Resolver role editor leads to an unintended global SET\_ADDR grant
* \#92156 \[W\&A-Medium] useMigrationGasEstimate preserves an authority-bearing plan after manager revocation, restoring a revoked address during V1-to-V2 migration
* \#90812 \[W\&A-Medium] TOCTOU in Manager migration installs an attacker resolver and restores former V1 manager authority
* \#90820 \[W\&A-Critical] The Manager checks name ownership with one canonicalization function and builds the renewal transaction with another, so one invisible character in a URL makes a user pay to re...
* \#91722 \[W\&A-Critical] Choosing a primary name claims the ENSIP-15-normalized twin of the displayed row, handing the victim's on-chain identity to whoever owns that twin
* \#91299 \[W\&A-Critical] Unmapped resolver role becomes global and lets a delegate redirect other ENS names
* \#93084 \[W\&A-Critical] Renewal flow signs a paid transaction against a different ENS name than the one it resolves, gates and displays
* \#89534 \[W\&A-Insight] Session enable signs attacker-authored policy hidden in an opaque salt, yielding an effectively permanent attacker-keyed session
* \#92452 \[W\&A-Critical] Explorer collapses per-name resolver roles by account, so Remove User leaves a delegate able to redirect another ENS name to an attacker-controlled address
* \#90843 \[W\&A-Critical] Portal turns an unresolved name-scoped resolver role into ROOT, granting resolver-wide control over unrelated names
* \#91731 \[W\&A-Critical] Renewal screen displays an expiry date computed from a different ENS name than the one the transaction renews
* \#91728 \[W\&A-Insight] Explorer renders an unverified default.reverse claim as a forward-verified "Primary name" on every L2 row, letting any address be displayed as any ENS name
* \#91732 \[W\&A-Insight] Attacker-chosen ENS text-record key is rendered verbatim into the Explorer homepage activity feed
* \#91615 \[W\&A-Medium] queues/event-ingestion.ts: idempotency key stamped before the delivery fan-out
* \#92461 \[W\&A-High] Portal registration shows $8.01 for name B, charges 160.109598 USDC for name A
* \#93090 \[W\&A-Critical] Explorer "Send name" resolves a recipient ENS name with no address record to the name's registry owner instead of failing
* \#92463 \[W\&A-High] Unbounded EIP-2612 permit in \`planHcaIntentFunding\` lets a hostile intent quote drain the connected wallet's USDC on a transaction whose amount the app never displays
* \#92464 \[W\&A-Critical] Resolver Role Aggregation Causes “Remove User from All Roles” to Revoke Only One Resource, Leaving the Delegate Able to Redirect ENS Addresses
* \#91658 \[W\&A-Insight] Missing forward-match verification in reverse resolution lets any address display any ENS name as its verified primary name, causing users to send funds to the attacker
* \#93089 \[W\&A-Low] SendGrid webhook matches events to the first channel row with that email, so a pre-registered pending duplicate absorbs another user's unsubscribe or bounce
* \#90852 \[W\&A-Critical] The Explorer's irreversible name transfer resolves a recipient name with no address record to its registry controller, silently delivering the name to the previous owner while ...
* \#93093 \[W\&A-Critical] A pending role grant can be relabelled with a trusted recipient while retaining attacker calldata
* \#93094 \[W\&A-Low] Notification delivery targets are treated as global identities: provider events and push registrations rewrite other users' notification settings with zero victim interaction
* \#89255 \[W\&A-Insight] Attacker-Controlled ENS Name Injected Into Telegram Expiry Notifications Without Escaping
* \#93096 \[W\&A-Medium] Missing Manager revalidation during V1→V2 migration grants resolver control to a revoked Manager
* \#91688 \[W\&A-Critical] Manager authorizes a canonical-name victim to write to an attacker-owned raw-name resolver
* \#93103 \[W\&A-Critical] Deleting a resolver delegate can grant an unselected address-record role to the next delegate
* \#90859 \[W\&A-Insight] The Explorer's reverse-resolution table prints "Forward match: True" and a "Primary name" badge for a name it never forward-verified, letting any address present itself under a ...
* \#90459 \[W\&A-Critical] Incomplete Resolver Role Revocation Leaves Attacker Control Over Another ENS Name and Enables Direct Fund Theft
* \#91345 \[W\&A-Critical] \`resolveAddressOrName\` treats an unresolved V1 Manager as a transfer recipient, sending a V2 name NFT to an unintended address
* \#91822 \[W\&A-Low] Editing the recipient during Portal transfer preparation sends the ENS name to the replaced address
* \#93104 \[W\&A-Low] Any authenticated user can bind a stranger's mailbox to their own ENS account with one click, and lock the real owner out of email notifications
* \#91348 \[W\&A-Critical] Public Safe ERC-1271 transcript can be redeemed first to steal an ENS seven-day JWT
* \#91306 \[W\&A-Critical] Explorer's resolver-roles editor builds every transaction against a scope it never displays, silently granting an account authority over every name a resolver serves
* \#93106 \[W\&A-Medium] Stale subgraph state in the Manager's v1-to-v2 migration flow re-grants resolver-control authority to an ENSv1 manager the registrant already revoked
* \#91181 \[W\&A-Critical] Normalization desync in the Manager renewal route validates one name and renews another, letting a link make a victim pay to extend an attacker's name
* \#91062 \[W\&A-Critical] Scope collapse in the resolver-roles editor causes role writes to target the resolver ROOT resource, granting authority over every name it serves and making "Remove user" repor...
* \#91487 \[W\&A-Low] api-worker: user\_channels never enforces one verified owner per email target
* \#93109 \[W\&A-Low] Missing per-user scoping in the email verification rate limit lets any signed-in account lock another user out of adding their email to notifications
* \#92250 \[W\&A-Low] Email verification can be claimed by two accounts because uniqueness is not re-checked during verification
* \#92474 \[W\&A-Critical] Improper resource binding in Portal “Remove user” leaves root resolver authority untouched
* \#91318 \[W\&A-Critical] Portal's “remove all roles” action revokes only one name, leaving a delegate's global resolver authority active
* \#89952 \[W\&A-Critical] An omitted non-root role mapping makes Edit user roles encode a root-scoped grant
* \#89614 \[W\&A-Critical] Cross-resource role aggregation makes "Remove user" revoke only one name and leaves delegated resolver control
* \#92481 \[W\&A-Low] Email-channel takeover: an unauthenticated verify endpoint finalises a pre-bound victim address
* \#90883 \[W\&A-Critical] The Manager renewal flow reads one name and pays for another, so a name's owner funds an attacker's registration while their own name is untouched
* \#89960 \[W\&A-Critical] Raw-name ownership check can transfer a different normalized ENSv2 token
* \#92494 \[W\&A-Low] apps/portal: transfer encodes a cached ENS resolution as the recipient
* \#93111 \[W\&A-Critical] ENS falsely shows a victim as owner of an attacker's v1 name via unconsented setOwner, suppressing the third-party renewal warning so the victim pays to renew the attacker's name
* \#92517 \[W\&A-Medium] Same-key record replacement is serialized deletion-last, causing Portal to expose a revoked delegate's address as the Base recipient
* \#93112 \[W\&A-Critical] Unresolved resolver role resource becomes a root grant
* \#92489 \[W\&A-Medium] apps/manager: migration grants ROLE\_SET\_RESOLVER to a revoked V1 manager
* \#93115 \[W\&A-Critical] Manager renew route: the victim pays to renew an attacker's name and their own lapses
* \#93116 \[W\&A-Critical] After abandoning Alpha and switching Portal to Bravo, the surviving registration actor later submits \`register(Alpha, ...)\` while Portal displays Bravo
* \#93118 \[W\&A-Critical] Missing ENSIP-15 normalization on the register and renew write paths lets an attacker redirect a victim's renewal payment to a name the attacker owns
* \#91569 \[W\&A-Critical] Inconsistent ENS name normalization displays and copies an attacker controlled address as the victim profile's main receiving address
* \#92840 \[W\&A-Critical] Explorer's renewal builder substitutes the name it signs — the user pays to renew a different ENS name than the one displayed
* \#92535 \[W\&A-Critical] Portal row reuse can grant one resolver delegate another account's \`UPGRADE\` role
* \#93119 \[W\&A-Critical] Transfer flow authorizes one ENS name and signs transactions for a different one
* \#90337 \[W\&A-Critical] Explorer removes only one grouped role scope, allowing removed users to redirect ENS transfers
* \#92522 \[W\&A-Critical] Unconditional owner fallback in Explorer name resolution leads to the app resolving a record-less ENS name to its owner, an attacker-controlled address, pre-filled into the nam...
* \#92225 \[W\&A-Critical] Resolver “Remove user” leaves root write authority active, allowing a removed delegate to redirect ENS resolution
* \#89360 \[W\&A-Critical] The \`/$name\` profile route resolves one URL parameter as two ENS names, rendering an attacker's receiving address under the victim's genuine owner and expiry
* \#93121 \[W\&A-Critical] Renewal flow signs a paid transaction against a different ENS name than the one it resolves, gates and displays
* \#89535 \[W\&A-Critical] Manager profile page renders an attacker-chosen ETH address next to a name's genuine owner
* \#92446 \[W\&A-Medium] Stale V1 migration plan restores revoked manager authority and attacker resolver, hijacking an ENS name
* \#93099 \[W\&A-Critical] Resolver roles: the sidebar displays one permission scope and signs a different one
* \#90904 \[W\&A-Critical] Mis-scoped resolver role removal leaves a removed account able to repoint any name the resolver serves
* \#91566 \[W\&A-Critical] Explorer “Remove user from all roles” revokes one name resource and leaves root address authority live
* \#92540 \[W\&A-High] The Explorer's role-history panel identifies a name by its label hash alone and never by the registry that holds it
* \#92979 \[W\&A-Insight] Telegram authentication accepts future-dated \`auth\_date\` values
* \#89930 \[W\&A-Critical] Manager profile displays a trusted name's owner but an attacker-controlled receiving address (owner-node vs records-node normalization split)
* \#90907 \[W\&A-Critical] The migration flow silently grants a third party permanent ROLE\_SET\_RESOLVER over the user's name, and destroys the only revocation path in the same transaction
* \#92544 \[W\&A-Low] Explorer's name page asserts "You are the owner of \<name>" and a "Paid" figure taken verbatim from the URL query string, on any name for any visitor, beside an ownership-free button...
* \#92546 \[W\&A-Insight] Explorer's name page asserts "You are the owner of \<name>" and a "Paid" figure taken verbatim from the URL query string, on any name for any visitor, beside an ownership-free bu...
* \#92661 \[W\&A-Medium] Missing ownership check in the SendGrid webhook's channel lookup lets any account silently disable another person's verified ENS email notifications, permanently
* \#92483 \[W\&A-Critical] Migrate to v2, hand your name's resolver to a third party the app never validated
* \#92553 \[W\&A-Insight] Explorer home page "Recent Activity" shows unverified reverse-record names as genuine ENS names (persistent content injection)
* \#92466 \[W\&A-Medium] Unbounded ENS \`description\` record is emitted twice through a 6x-expanding HTML escaper, so one attacker-owned name kills the Explorer's Cloudflare Worker isolate for every concu...
* \#89328 \[W\&A-High] Registration checkout charges more than displayed
* \#92665 \[W\&A-High] Manager does not bind Orchestrator-returned HCA registration calldata to the requested calldata, allowing an attacker-controlled subregistry on a victim-owned root name
* \#91314 \[W\&A-Medium] Explorer builds a user's own name list from an ensjs query whose default relation set includes \`resolvedAddress\`, so any name an attacker points at a victim's address is listed a...
* \#92542 \[W\&A-Medium] Explorer's roles editor revokes \`ROLE\_CAN\_TRANSFER\_ADMIN\` in a "Remove user" whose only warning describes a different consequence, freezing the name for the rest of its registrat...
* \#89728 \[W\&A-Critical] Explorer app fabricates a 'Resolved:' for recordless ENS names - silently substitutes the name's owner, sending name transfers and registry-wide role grants to an attacker's ad...
* \#92738 \[W\&A-Critical] Resolver “Remove user from all roles” revokes only one scope, leaving global address-hijack authority active
* \#92764 \[W\&A-Critical] Encoded-label recipient names select an attacker-owned literal label but query the honest hash-only node, redirecting Portal name transfers
* \#92616 \[W\&A-Critical] Explorer resolves a typed ENS name to a previous owner and builds the wrong recipient transaction
* \#92671 \[W\&A-Critical] ENS Manager registers one name and then points the buyer's primary name at a registration owned by someone else
* \#92618 \[W\&A-Medium] Revoked V1 manager receives persistent ENSv2 resolver control from an already-prepared migration
* \#90069 \[W\&A-Low] Missing ownership re-check in email channel verification lets any authenticated user bind another user's email address to their own account as a verified notification channel
* \#92622 \[W\&A-Critical] An unmappable name-scoped resolver role becomes a resolver-wide root grant signed by the victim
* \#92623 \[W\&A-Medium] Registry-wide role-event cap hides a retained delegate and enables post-transfer resolver and subregistry hijacking
* \#92771 \[W\&A-Critical] Portal treats a factory-verified UserRegistry as a PermissionedResolver, so “Set Pubkey” grants registry-wide UNREGISTER
* \#91414 \[W\&A-Medium] Revoked V1 manager can regain resolver control through a stale V1→V2 migration plan
* \#91441 \[W\&A-High] An unrelated contract can inject persistent arbitrary text into another name's canonical Portal history
* \#92625 \[W\&A-Medium] Stale V1 registry state restores a revoked manager and resolver during V1-to-V2 migration
* \#92773 \[W\&A-Critical] Unresolved resolver scopes turn scoped Set Address edits into resolver-wide grants
* \#91443 \[W\&A-Low] An unsolicited transferred name's one-byte ABI record indefinitely locks Manager's default dashboard
* \#91444 \[W\&A-Low] An unsolicited ENSv2 name with an oversized coin-121 result blocks the victim's default Manager dashboard
* \#89569 \[W\&A-Low] Email notification channels are not unique per address: an attacker can bind a victim's email to their own account, take the address out of the victim's reach, and have ENS mail del...
* \#90651 \[W\&A-Low] Unverified email channel lets any authenticated user delete another user's SendGrid contact, silently unsubscribing them from ENS broadcast email
* \#89469 \[W\&A-Insight] SSRF self-host protection bypass in the OG image worker leads to temporary denial of service (resource amplification) of the card preview service
* \#92778 \[W\&A-Medium] Manager accepts an attacker-controlled subregistry as a completed HCA registration after the intended reveal fails
* \#92779 \[W\&A-High] Owner-signed HCA intents are signed from the orchestrator's returned intent with no execution-equivalence check, on the client or on chain
* \#92782 \[W\&A-Critical] registration and renewal write an on-chain node that no compliant client resolves
* \#90363 \[W\&A-Critical] Editing a Fine-Grained Resolver Role Silently Grants Root-Wide Authority
* \#92632 \[W\&A-Critical] A resolver role row whose scope the UI cannot resolve is saved as a ROOT grant, so one click on a single account’s row signs grantRootRoles over every name on the resolver
* \#92685 \[W\&A-Medium] Stale V1 resolver state in Manager migration restores an attacker-controlled resolver and hijacks the migrated ENS name
* \#92792 \[W\&A-Low] Any account can exhaust the email-verification rate limit for an address it does not own, blocking that address's verification for an hour
* \#92635 \[W\&A-Insight] The Explorer offers “Set primary name” based on a record the attacker controls and never checks name ownership, so one click on a link replaces the victim’s primary name with a ...
* \#91631 \[W\&A-Low] Missing token-to-account binding in the unauthenticated email-verification endpoint leads to cross-account mailbox binding, victim lock-out, and an email-membership oracle
* \#91189 \[W\&A-Medium] Portal hides resolver cleanup when the seller lacks direct ROLE\_SET\_RESOLVER, leaving a delegate able to redirect the new owner's ENS resolution and later transactions
* \#92695 \[W\&A-Medium] setupControlledResolver Intent Underfunding and Session Key Policy Reversion Prevents Controlled Resolver Deployment and Profile Setup in Manager
* \#91063 \[W\&A-Critical] Owner fallback in Portal's ENS name resolution pre-fills an unpublished address, sending role grants and name transfers to an unintended recipient
* \#92263 \[W\&A-Medium] A seller who keeps the resolver on a name transfer keeps root authority over the sold name, and can repoint it at their own address after the sale
* \#92527 \[W\&A-Critical] The Explorer's resolver-roles editor writes a scope the administrator never chose: narrow grants are signed as resolver-wide (ROOT), and "remove from all roles" leaves ROOT sta...
* \#92813 \[W\&A-Low] Missing re-validation in the email channel verify handler lets two accounts hold a verified channel for one address, leading to unstoppable notification delivery and destruction of ...
* \#92503 \[W\&A-Medium] Stale V1 manager cache re-grants a revoked manager during V2 migration
* \#92108 \[W\&A-Insight] Unescaped ENS name in api-worker Telegram/Push notification templates allows Markdown injection and a clickable phishing link inside official ENS expiry alerts
* \#90874 \[W\&A-Critical] Portal authorizes a literal bracket-label decoy but changes the resolver of the embedded-hash ENS name
* \#92699 \[W\&A-Critical] A role scope the app cannot resolve is granted on every name instead
* \#92701 \[W\&A-Insight] Missing escaping in the notification worker lets a registered ENS label inject Markdown into official ENS notifications, placing an attacker-chosen link inside messages delivere...
* \#92475 \[W\&A-Critical] A profile URL carrying one invisible codepoint renders the victim's verified owner beside an address published by a name the attacker registered
* \#89627 \[W\&A-Medium] apps/portal: the name-transfer cleanup checklist omits third-party role grants, so a stranger keeps \`setResolver\` on the sold name and repoints its ETH address
* \#92925 \[W\&A-Critical] The Manager profile page normalizes the name for the owner query but not for the records query, resolving an ENS name to an attacker-controlled address
* \#90593 \[W\&A-Insight] The Token-Info page publishes the namehash of a name's punycode A-label, so every non-ASCII ENS name shows the node of a different, separately registerable name
* \#92310 \[W\&A-Critical] A recipient name with no address record resolves to the name's manager, and the transfer goes there
* \#91571 \[W\&A-Insight] Set primary name in the ENS Explorer signs setAddr() with the address from the URL, not the connected wallet, so an attacker can redirect where any ENS name resolves
* \#91909 \[W\&A-Critical] Grouped resolver-role removal signs a single-resource revoke, leaving a removed delegate able to replace the ENS address
* \#92794 \[W\&A-Critical] Manager's bulk-renew dialog substitutes a normalized lookalike for the victim's real registered label, letting a victim's own connected wallet pay to renew an attacker's name
* \#92716 \[W\&A-Critical] Index-based row identity in Portal's resolver-roles table lets an admin's stale draft silently grant a global resolver role to an uninvolved account
* \#92713 \[W\&A-Critical] A phishing link renews the attacker's own junk registration with the victim's money while Manager displays the victim's real name
* \#92552 \[W\&A-Critical] Portal “Remove user from all roles” revokes only one resolver scope; the surviving scope permits a profile-description write that remains after role cleanup
* \#92649 \[W\&A-Medium] Improper record deduplication in the Portal silently deletes a reviewed Base address and resolves the ENS name to an attacker-controlled fallback
* \#92796 \[W\&A-Insight] Unverified \`default.reverse\` is falsely labelled “Forward match: True” and “Primary name” on inherited L2 rows
* \#92714 \[W\&A-Critical] Resolver "Remove User" Silently No-Ops Root-Scoped Roles — Enables Persistent ENS Name-Resolution Hijack
* \#92718 \[W\&A-Medium] Portal's v1 bulk-renewal table treats "resolves to me" as "owned by me", letting a victim's own connected wallet pay to renew an attacker's name
* \#92624 \[W\&A-Critical] Two same-named withEthSuffix helpers in one post-registration sequence, one raw and one ENSIP-15, make the registration flow set the victim's primary name to a registration the...
* \#92626 \[W\&A-Critical] Manager bulk renewal renders the raw indexer name on every row and signs the normalized label, so the victim's stablecoins extend a registration that appears nowhere in the dialog
* \#92806 \[W\&A-Critical] Resolver role removal silently leaves ROLE\_SET\_DATA authority on-chain
* \#91838 \[W\&A-Critical] Migration grants a stale v1 controller a permanent ROLE\_SET\_RESOLVER on the migrated v2 name, letting a third party repoint what the victim's name resolves to
* \#92761 \[W\&A-Critical] Inconsistent ENS normalization makes Manager submit an attacker-owned label in a victim-approved renewal
* \#92174 \[W\&A-Critical] Registration and migration grant resolver-global (not name-scoped) root roles that "Detach the resolver" never revokes, so a name's seller keeps redirecting the buyer's \`addr(6...
* \#89456 \[W\&A-Critical] Renewal route prices and displays the normalized name but signs the raw label, so a crafted URL renews an attacker's lookalike name with the victim's funds
* \#92651 \[W\&A-Critical] Explorer transfer resolves an ENS recipient to the name's v1 controller rather than its owner, sending the user's name to a party that does not own the name they addressed
* \#92723 \[W\&A-Medium] Missing \`ccipRead\` guard in the Explorer's OG-image Worker client lets an attacker-controlled resolver drive unbounded server-side requests, exhausting the Worker's RPC provider ...
* \#92817 \[W\&A-Critical] Missing label validation in the Portal's Change Resolver flow leads to resolver hijack of a different, real ENS name
* \#92816 \[W\&A-Medium] The expiry-discovery cron's non-unique-timestamp cursor pagination permanently and silently drops names tied at a page-1000 boundary from every expiry notification
* \#89705 \[W\&A-Insight] Missing recipient-address validation in 'Set primary name' leads to ENS name resolution hijacked to an attacker-controlled address
* \#91635 \[W\&A-Critical] Incorrect resolution on the Manager profile page displays an attacker-controlled receiving address under a legitimate ENS name's owner identity
* \#91681 \[W\&A-Critical] Manager displays the victim's NFC expiry but renews an attacker-owned raw NFD ENS token
* \#91967 \[W\&A-Critical] Ownership check on the wrong name in the Explorer transfer flow leads to hijacking of a victim's ENS name, leaving it inaccessible after the flow
* \#91080 \[W\&A-Critical] A visually identical link makes a name holder pay to extend somebody else's registration: Manager decides ownership, expiry and the third-party renewal warning on the ENS-norma...
* \#91997 \[W\&A-Low] Manager re-authors a prior owner's social records into the recipient's replacement resolver despite a clear-old-details promise
* \#92300 \[W\&A-Critical] Portal lists an attacker-owned name under "Names you own" in place of the user's own, and transferring it transfers the user's real name instead
* \#89736 \[W\&A-Insight] Unverified \`default.reverse\` name rendered as a verified primary name on L2 rows, enabling identity spoofing
* \#92838 \[W\&A-Medium] TOCTOU in Manager v1-to-v2 migration reinstalls a replaced attacker-controlled resolver and hijacks an ENS name
* \#89551 \[W\&A-Insight] ENS Explorer renders a forged “Forward match: True / Primary name” verification badge for an attacker’s unverified default.reverse name
* \#89578 \[W\&A-High] Unreconciled funding budget in Manager HCA registration leads to the user approving a permit for 2.5× the total the checkout displays
* \#89903 \[W\&A-Medium] Manager migration restores resolver authority to a revoked ENSv1 manager
* \#90638 \[W\&A-Critical] Renew gate reads one name, renew calldata spends on another
* \#90002 \[W\&A-Medium] Manager grants V2 \`ROLE\_SET\_RESOLVER\` from unverified V1 manager state, restoring authority to a revoked manager during migration
* \#90314 \[W\&A-Medium] Migration resurrects a revoked V1 manager with V2 resolver authority
* \#90156 \[W\&A-Medium] Missing execution-time manager validation restores a removed V1 manager and lets them hijack a migrated name's resolver
* \#91869 \[W\&A-Critical] Normalization mismatch between the owner and records queries in the Manager profile route displays an attacker's name and address under the victim's ownership, causing stealthy...
* \#91962 \[W\&A-Insight] Hardcoded \`True\` in the Explorer's "Forward match" column certifies any address as the verified primary-name holder of any ENS name
* \#92105 \[W\&A-Insight] Explorer prints "Forward match: True" and "Primary name" for a \`default.reverse\` record ENS resolution reports as a mismatch
* \#92202 \[W\&A-Insight] Inheriting an L1 default.reverse name is treated as equivalent to passing forward verification, so any address is shown as the verified holder of a name it does not own, while t...
* \#92283 \[W\&A-Insight] Notification templates interpolate an ENS label with no escaping: a third party can plant persistent text in another user's Manager inbox with no interaction by that user, and a...
* \#92341 \[W\&A-Medium] Manager restores V2 resolver authority to a V1 manager removed before confirmation
* \#92893 \[W\&A-Low] Live ENS recipient refetch changes the undisclosed owner of a new subname
* \#92859 \[W\&A-Critical] Unresolved name-scoped resolver role is converted into a root grant, enabling unauthorized record changes for unrelated names
* \#92850 \[W\&A-Critical] Two conflicting ENS name derivations on the Manager's renewal page cause the victim to pay to extend an attacker's registration instead of the name shown on screen
* \#91580 \[W\&A-Critical] Resolver roles editor writes a scope the admin was never shown, defaulting to root
* \#89864 \[W\&A-Critical] ENS Explorer silently resolves a name with no \`addr(60)\` record to its token owner (owner-fallback) and consumes it on irreversible/authority-granting action paths, misdirectin...
* \#90613 \[W\&A-Medium] Stale V1 manager state during v1→v2 migration resurrects a revoked manager with ROLE\_SET\_RESOLVER, enabling resolver hijacking
* \#92581 \[W\&A-Critical] The Explorer names the ENSv1 registry controller as a name's Owner, so a seller who has already handed over the NFT is still shown as the owner on the official ENS Explorer
* \#92583 \[W\&A-Low] POST /notifications/channels/email/verify ships with no requireAuth and re-checks nothing, so one click on a genuine ENS email transfers a victim's address to an attacker's account ...
* \#92582 \[W\&A-Critical] Set as Primary writes its forward legs to the name the user picked and its reverse leg to a different name, so one click points the victim's wallet at a name an attacker owns
* \#92584 \[W\&A-Critical] Explorer’s resolver role table omits ROLE\_SET\_DATA, so Remove user throws for every account the registration flow provisions and otherwise leaves write access the roles table r...
* \#92591 \[W\&A-Critical] Manager reads the profile with ENSIP-15 and writes the renewal with toLowerCase(), so a crafted link makes the victim's own profile suppress the third-party-renewal warning and...
* \#92585 \[W\&A-Medium] Explorer certifies any contract as the official, audited ENS Permissioned Resolver from a single storage word the contract writes itself
* \#92593 \[W\&A-Critical] Explorer's Extend flow charges the user to renew a stranger's ENS name instead of the one shown
* \#92596 \[W\&A-Medium] Manager v1 -> v2 migration restores a removed V1 manager as persistent V2 resolver authority, hijacking the name's resolver
* \#92986 \[W\&A-Insight] ENS Worker allows Telegram Markdown injection
* \#89516 \[W\&A-Critical] v1→v2 migration assigns the migrated name's resolver to a third party and grants them the role that keeps it
* \#90647 \[W\&A-Critical] v1→v2 migration silently grants a third-party address permanent-in-practice \`ROLE\_SET\_RESOLVER\` over the user's migrated name, with the grantee never displayed and no way to re...
* \#91724 \[W\&A-Critical] Extending a name from the address page renews the ENSIP-15-normalized twin while every screen shows the raw indexer name, so the fee extends someone else's registration
* \#91830 \[W\&A-Critical] Explorer renews a different ENS name than it displays: the victim pays to extend an attacker's name while their own expires
* \#92203 \[W\&A-Critical] Primary-name dialog derives the displayed name and the written name from two different expressions, so the confirmed operation writes a reverse record for a different ENS node,...
* \#93072 \[W\&A-Medium] Portal silently transfers a name with non-detachable third-party resolver authority
* \#92648 \[W\&A-Medium] The Explorer certifies a resolver as “official, audited and considered secure” from a storage word any contract can forge, and the same forged bit replaces the record-edit owners...
* \#89744 \[W\&A-Critical] v1→v2 migration unconditionally grants ROLE\_SET\_RESOLVER on the v2 ETHRegistry to the name's stale v1 registry controller — an attacker-settable address the Manager never displ...
* \#90702 \[W\&A-Critical] v1→v2 migration silently grants a stale V1 controller ROLE\_SET\_RESOLVER on the migrated name, enabling resolution hijack of an acquired name
* \#92909 \[W\&A-Medium] Migration reuses uncleared resolver state and activates a revoked delegate's attacker address
* \#89279 \[W\&A-Critical] Previous owner permanently controls what a transferred ENS name resolves to
* \#89727 \[W\&A-High] Manager checkout does not bind the accepted USDC amount to the later Permit and transfer
* \#89264 \[W\&A-Critical] Persistent ROOT resolver authorization survives ENS name transfer, allowing former owners to rewrite sold-name records and unrelated resolver resources
* \#92586 \[W\&A-Low] One ENS text record keyed on an Object.prototype member permanently replaces a name’s Manager profile page, and the Edit dialog inside it, with an error screen
* \#92587 \[W\&A-Critical] Explorer’s resolver permission editor keys its checkbox state to the table row index, so re-opening a row position grants the previous occupant’s permissions to a different acc...
* \#92588 \[W\&A-Critical] Transfer ownership signs the token id of getLabel(name) while every screen shows the raw route param, so the name that leaves the victim's wallet is not the name the flow displ...
* \#92934 \[W\&A-Medium] Unauthenticated server-side egress from the ENS Portal Worker via EIP-3668 CCIP-Read leads to ressource exhaustion
* \#93137 \[W\&A-High] Charged-vs-displayed divergence on the HCA registration checkout: on a failed budget quote the screen shows the rent (~~$8) but the funding permit charges the full budget (~~$20.20).
* \#92448 \[W\&A-Medium] Revoked ENSv1 manager regains resolver control when the owner migrates
* \#92352 \[W\&A-Critical] Portal says it removed all roles but sends a one-name revoke
* \#92963 \[W\&A-Critical] Cross-name profile state makes a Description-only save set an attacker address on another ENS name
* \#92602 \[W\&A-Critical] Explorer causes a connected wallet to grant a delegate resolver control over every name instead of one
* \#90691 \[W\&A-Critical] Portal authorizes a raw ENSv2 name but transfers the victim's canonical twin NFT
* \#92912 \[W\&A-Medium] Any address can replace the rendered history of any ENS name
* \#92961 \[W\&A-Insight] Portal falsely marks an unverified inherited reverse name as a verified primary name
* \#92605 \[W\&A-Critical] Merged resolver role removal revokes one scope while leaving root record-write authority
* \#92608 \[W\&A-Medium] Explorer checkout shows a $0.00 total, deletes the approval step, and the registrar charges the full rent price the user was never shown
* \#92609 \[W\&A-Medium] Stale V1 controller snapshot grants a revoked controller V2 resolver authority during migration
* \#90678 \[W\&A-Critical] Explorer: "Remove user" revokes one resolver scope, leaving removed accounts with sibling-name roles and global Upgrade authority
* \#92545 \[W\&A-Critical] Removing a resolver user leaves root upgrade authority intact
* \#92559 \[W\&A-Critical] Explorer reuses a former-owner ENS resolution and encodes the wrong recipient in an ENSv2 NFT transfer
* \#92611 \[W\&A-Medium] Stale V1 resolver snapshot migrates an attacker-controlled resolver after V1 recovery
* \#92353 \[W\&A-Critical] Raw Unicode name causes Portal to transfer the victim's different canonical ENS token
* \#93016 \[W\&A-Critical] Manager and Explorer transact the normalized twin of the displayed ENS name, so a renewal pays to extend an attacker's registration and a transfer moves the wrong name — and th...
* \#90749 \[W\&A-High] Portal reparses a flat dotted label and grants resolver authority on a different ENS name
* \#92954 \[W\&A-Insight] Unauthorized Profile Image Change. Upload Does Not Verify Signer Ownership
* \#93033 \[W\&A-High] apps/portal's address-history page trusts unconsented on-chain ownership: anyone can plant attacker-authored history (and a forged "From" address) on any address's page including a...
* \#90095 \[W\&A-Insight] One signature on a legitimate app.ens.domains screen redirects a victim's ENS name to the attacker — the app builds the redirect from an attacker-chosen unverified record and ne...
* \#89418 \[W\&A-Critical] Broken access control in the resolver Remove-user flow leaves a revoked delegate able to redirect the address records of every name on the resolver
* \#90163 \[W\&A-Critical] Manager renewal route proves renewability from the normalized name but renews the raw label, so a user pays to extend an attacker's registration
* \#93061 \[W\&A-Critical] Resolver Role Aggregation Causes “Remove User from All Roles” to Revoke Only One Resource, Leaving the Delegate Able to Redirect ENS Addresses
* \#89300 \[W\&A-Insight] Patched \`@rhinestone/sdk\` identity is undocumented in source — two provenance comments assert two different, superseded SHA-256 values
* \#92667 \[W\&A-Critical] Improper role revocation in Explorer leaves global ROLE\_SET\_ADDR active, enabling ENS payment redirection
* \#92752 \[W\&A-Critical] Removing a resolver user revokes only one resource and leaves root or other-name authority active
* \#92740 \[W\&A-Critical] Missing ENSIP-15 normalization on the Manager's \`/{name}\` route splits one profile page across two ENS nodes, rendering an attacker-controlled address beside the real owner and...
* \#92706 \[W\&A-Insight] Telegram Markdown Injection in Notification Delivery — Unescaped ENS Name Enables Phishing via the Official Notifications Bot
* \#92753 \[W\&A-Critical] Incomplete revocation in the resolver user-removal flow retains resolver-root authority, leading to theft of user funds via payment-address redirection
* \#92739 \[W\&A-Low] Portal transfer step ID reuse redirects an ENS-name recipient transfer to the former address
* \#92755 \[W\&A-Critical] Missing scope validation in ENS Explorer role edits grants resolver-wide address access
* \#93008 \[W\&A-Medium] recipient-controlled transfer failure leaves ENS resolution attacker-controlled and diverts payments
* \#93026 \[W\&A-Critical] A name sent with the portal's default plan bricks every third-party subname beneath it, and the name's new holder can re-mint those labels to an address of their choosing
* \#91295 \[W\&A-Critical] IOST-only resolver delegate can make Manager show and copy its value as the SOL receiving address
* \#91982 \[W\&A-Critical] Manager keep-v1 migration of a locked custom-resolver name freezes record control
* \#92345 \[W\&A-Critical] Attacker can register a name's normalized twin to redirect the owner's bulk-renew payment and let their name expire
* \#92305 \[W\&A-Low] Email-verification rate limiter runs after the rows it should gate and is keyed on the target address, so any account can burn a chosen email's hourly quota and block that person fr...
* \#92597 \[W\&A-High] HCA registration checkout can display exact rent while a recovered quote authorizes a larger wallet debit
* \#92509 \[W\&A-Medium] Revoked manager regains resolver control during V1 to V2 migration
* \#92762 \[W\&A-Critical] Late ENS normalization makes the Portal detach and transfer a different name than the transfer page authorizes
* \#91839 \[W\&A-Insight] An attacker-gifted v1 name modifies the arguments of the migration transaction set the victim signs, adding an approval that would not otherwise be requested and widening a toke...
* \#89395 \[W\&A-Critical] Unsafe owner fallback in the shared address resolver leads to irreversible transfer of ENS names to an unintended address
* \#89616 \[W\&A-Medium] Migration restores a stale former ENSv1 manager, allowing post-reclaim resolver hijacking
* \#89474 \[W\&A-Critical] Explorer transfer flow moves a different ENS name than the one displayed, stealing the victim's name
* \#89917 \[W\&A-Critical] The app acts on a different ENS name than the one it displays and just verified you own — an unsolicited airdrop turns "get rid of this junk" into irreversible loss of a valuab...
* \#89675 \[W\&A-Medium] v1→v2 migration re-grants ROLE\_SET\_RESOLVER to a revoked manager from a stale snapshot
* \#89585 \[W\&A-Critical] \[Critical] Name transfer resolves its recipient to the v1 registry controller, not the ERC-721 holder -- the Explorer also labels that controller as Owner
* \#91256 \[W\&A-Low] Unauthenticated verify endpoint binds a victim's email address to the attacker's ENS account
* \#91640 \[W\&A-Critical] Bulk renewal signs a different label than the name displayed, so the fee renews an unintended name (displayed-vs-signed mismatch)
* \#92996 \[W\&A-High] Portal's symbol collision preserves a prior owner's Base address when a canonical row is deleted
* \#92110 \[W\&A-High] Portal reparses a two-dot raw label and authorizes a different four-label ENS name
* \#91044 \[W\&A-Critical] Explorer authorises a raw ENS name but transfers the normalised name, so a user disposing of a decoy burns their genuine name
* \#91887 \[W\&A-Critical] Resolver-role resource conflation makes “Remove user from all roles” leave global address-write authority
* \#89888 \[W\&A-Critical] Missing ENSIP-15 normalization in Manager registration lets an attacker own the canonical name while a victim buys a hidden lookalike
* \#92640 \[W\&A-Critical] Explorer authorizes record editing by resolver role instead of name ownership — a former owner can edit the new owner's records in-app, and the real owner is locked out (name h...
* \#89507 \[W\&A-Critical] "Remove user" on the resolver roles panel revokes only one resource scope, leaving listed authority in place
* \#90583 \[W\&A-Medium] Manager trusts stale V1 owner and resolver during .eth migration
* \#90646 \[W\&A-Insight] Explorer renders an unverified \`default.reverse\` claim as a verified Primary Name, letting any address assert any ENS name (e.g. \`vitalik.eth\`) with a green "Forward match: True...
* \#90688 \[W\&A-Critical] Incomplete role revocation: Resolver "Remove user" leaves a co-manager's global (root) role in place, so a removed user can still set any name's address record and redirect funds
* \#89329 \[W\&A-High] Uncapped funding permit → full USDC drain on owner-signed HCA actions
* \#92949 \[W\&A-Critical] Explorer \`Remove User\` revokes only one resolver scope and leaves global address write authority active
* \#92439 \[W\&A-Insight] Unescaped ENS name in Telegram expiry-notification template enables persistent Markdown/link injection reaching third-party users
* \#91121 \[W\&A-Critical] Manager profile page resolves one route name against two different ENS nodes, rendering an attacker-controlled receiving address beside the victim's genuine ownership data
* \#89641 \[W\&A-Critical] Explorer invents a recipient address and sends the user's ENS name to it
* \#89464 \[W\&A-Insight] ENS Explorer: reverse-resolution certifies an unverified \`default.reverse\` name as the verified L2 primary name
* \#89513 \[W\&A-Insight] Missing Markdown escaping in notification worker allows attacker author ENS-branded Telegram message content, or silently suppress a victim's expiry warnings
* \#90379 \[W\&A-Critical] Deleting one resolver-role row rebinds its permission draft to the next account and grants that account an unselected address role
* \#90544 \[W\&A-Critical] Post-confirmation Unicode normalization substitutes an attacker-owned primary name, causing Manager to render an attacker-controlled Optimism recipient.
* \#92746 \[W\&A-Critical] ENS Explorer's “Remove user from all roles” leaves active address-control permission on another name
* \#92445 \[W\&A-Insight] Telegram notification Markdown injection via attacker-controlled ENS name
* \#91524 \[W\&A-Critical] Base58 case folding substitutes the Solana copy target
* \#92878 \[W\&A-Insight] Persistent Markdown/content injection into Telegram expiry-notification messages via an unsanitized ENS name label
* \#92828 \[W\&A-Medium] Stale manager state restores a revoked account that hijacks the migrated ENS resolver

</details>
