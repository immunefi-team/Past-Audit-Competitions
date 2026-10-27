# Boost \_ Firedancer v0.1 34290 - \[Blockchain\_DLT - Medium] bank tile overflow

Submitted on Thu Aug 08 2024 11:40:14 GMT-0400 (Atlantic Standard Time) by @gln for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #34290

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Process to process RCE between sandboxed tiles

## Description

## Brief/Intro

Bank tile incorrectly pre-allocates memory for storing incoming transactions.

As a result buffer overflow will occur when it tries to process microblock containing several txns.

## Vulnerability Details

The bank tile calls fd\_bank\_abi\_txn\_init() function to process incoming transactions.

Let's look at the code of this function <https://github.com/firedancer-io/firedancer/blob/main/src/disco/bank/fd\\_bank\\_abi.c#L270>

```
int
fd_bank_abi_txn_init( fd_bank_abi_txn_t * out_txn,
                      uchar *             out_sidecar,
                      void const *        bank,
                      fd_blake3_t *       blake3,
                      uchar *             payload,
                      ulong               payload_sz,
                      fd_txn_t *          txn,
                      int                 is_simple_vote ) {
  out_txn->signatures_cnt = txn->signature_cnt;
  out_txn->signatures_cap = txn->signature_cnt;
  out_txn->signatures     = (void*)(payload + txn->signature_off);

  fd_blake3_init( blake3 );
  fd_blake3_append( blake3, "solana-tx-message-v1", 20UL );
  fd_blake3_append( blake3, payload + txn->message_off, payload_sz - txn->message_off );
  fd_blake3_fini( blake3, out_txn->message_hash );

  out_txn->is_simple_vote_tx = !!is_simple_vote;

  if( FD_LIKELY( txn->transaction_version==FD_TXN_VLEGACY ) ) {
    sanitized_txn_abi_legacy_message1_t * legacy = &out_txn->message.legacy;
    sanitized_txn_abi_legacy_message0_t * message = &legacy->message.owned;

    out_txn->message.tag = 0UL;

    legacy->is_writable_account_cache_cnt = txn->acct_addr_cnt;
    legacy->is_writable_account_cache_cap = txn->acct_addr_cnt;
    legacy->is_writable_account_cache     = out_sidecar;
    int _is_upgradeable_loader_present = is_upgradeable_loader_present( txn, payload, NULL );
    for( ushort i=0; i<txn->acct_addr_cnt; i++ ) {
      int is_writable = fd_txn_is_writable( txn, i ) &&
                        /* Agave does this check, but we don't need to here because pack
                           rejects these transactions before they make it to the bank.

                           !fd_bank_abi_builtin_keys_and_sysvars_tbl_contains( (const fd_acct_addr_t*)(payload + txn->acct_addr_off + i*32UL) ) */
                        (!is_key_called_as_program( txn, i ) || _is_upgradeable_loader_present);
      legacy->is_writable_account_cache[ i ] = !!is_writable;
    }
    out_sidecar += txn->acct_addr_cnt;
    out_sidecar = (void*)fd_ulong_align_up( (ulong)out_sidecar, 8UL );

    message->account_keys_cnt = txn->acct_addr_cnt;
    message->account_keys_cap = txn->acct_addr_cnt;
    message->account_keys     = (void*)(payload + txn->acct_addr_off);

    message->instructions_cnt = txn->instr_cnt;
    message->instructions_cap = txn->instr_cnt;
    message->instructions     = (void*)out_sidecar;
    for( ulong i=0; i<txn->instr_cnt; i++ ) {
      fd_txn_instr_t * instr = &txn->instr[ i ];
      sanitized_txn_abi_compiled_instruction_t * out_instr = &message->instructions[ i ];

      out_instr->accounts_cnt = instr->acct_cnt;
      out_instr->accounts_cap = instr->acct_cnt;
      out_instr->accounts     = payload + instr->acct_off;

      out_instr->data_cnt = instr->data_sz;
      out_instr->data_cap = instr->data_sz;
      out_instr->data     = payload + instr->data_off;

      out_instr->program_id_index = instr->program_id;
    }
    out_sidecar += txn->instr_cnt*sizeof(sanitized_txn_abi_compiled_instruction_t);

    fd_memcpy( message->recent_blockhash, payload + txn->recent_blockhash_off, 32UL );
    message->header.num_required_signatures        = txn->signature_cnt;
    message->header.num_readonly_signed_accounts   = txn->readonly_signed_cnt;
    message->header.num_readonly_unsigned_accounts = txn->readonly_unsigned_cnt;
    return FD_BANK_ABI_TXN_INIT_SUCCESS;
    ...
}

```

Transaction is being processed and parsed data is written into 'out\_sidecar' and 'out\_txn' buffers.

Let's see how pointers to these buffer are handled inside bank tile:

```
static inline void
after_frag( void *             _ctx,
            ulong              in_idx,
            ulong              seq,
            ulong *            opt_sig,
            ulong *            opt_chunk,
            ulong *            opt_sz,
            ulong *            opt_tsorig,
            int *              opt_filter,
            fd_mux_context_t * mux ) {
  ...
  fd_bank_ctx_t * ctx = (fd_bank_ctx_t *)_ctx;

  uchar * dst = (uchar *)fd_chunk_to_laddr( ctx->out_mem, ctx->out_chunk );
  ulong txn_cnt = (*opt_sz-sizeof(fd_microblock_bank_trailer_t))/sizeof(fd_txn_p_t);

  ulong sanitized_txn_cnt = 0UL;
  ulong sidecar_footprint_bytes = 0UL;
  for( ulong i=0UL; i<txn_cnt; i++ ) {
    fd_txn_p_t * txn = (fd_txn_p_t *)( dst + (i*sizeof(fd_txn_p_t)) );

1.    void * abi_txn = ctx->txn_abi_mem + (sanitized_txn_cnt*FD_BANK_ABI_TXN_FOOTPRINT);
2.    void * abi_txn_sidecar = ctx->txn_sidecar_mem + sidecar_footprint_bytes;

    int result = fd_bank_abi_txn_init( abi_txn, abi_txn_sidecar, ctx->_bank, ctx->blake3, txn->payload, txn->payload_sz, TXN(txn), !!(txn->flags & FD_TXN_P_FLAGS_IS_SIMPLE_VOTE) );
    ...
    fd_txn_t * txn1 = TXN(txn);
    sidecar_footprint_bytes += FD_BANK_ABI_TXN_FOOTPRINT_SIDECAR( txn1->acct_addr_cnt, txn1->addr_table_adtl_cnt, txn1->instr_cnt, txn1->addr_table_lookup_cnt );
    sanitized_txn_cnt++;
  }
```

Lines #1, #2 - on each loop iteration both pointers are being incremented.

Initially , both txn\_abi\_mem and txn\_sidecar\_mem bufferrs were allocated like this:

```
static void
unprivileged_init( fd_topo_t *      topo,
                   fd_topo_tile_t * tile,
                   void *           scratch ) {
  FD_SCRATCH_ALLOC_INIT( l, scratch );
  fd_bank_ctx_t * ctx = FD_SCRATCH_ALLOC_APPEND( l, alignof( fd_bank_ctx_t ), sizeof( fd_bank_ctx_t ) );
  ...
  ctx->txn_abi_mem = FD_SCRATCH_ALLOC_APPEND( l, FD_BANK_ABI_TXN_ALIGN, MAX_TXN_PER_MICROBLOCK*FD_BANK_ABI_TXN_FOOTPRINT );
3.  ctx->txn_sidecar_mem = FD_SCRATCH_ALLOC_APPEND( l, FD_BANK_ABI_TXN_ALIGN, FD_BANK_ABI_TXN_FOOTPRINT_SIDECAR_MAX );
```

The issue is that on line #3 txn\_sidecar\_mem was allocated to store single transaction of max size (FD\_BANK\_ABI\_TXN\_FOOTPRINT\_SIDECAR\_MAX).

To simplify things we will focus on legacy txns.

The sizes of txn\_abi\_mem and txn\_sidecar\_mem buffers are 7688 and 18096 bytes.

The maximum number of txns in microblock is 31.

So, when fd\_bank\_abi\_txn\_init() will try to process several transactions and store them in these buffers, heap overflow will occur.

To trigger the issue I modified fuzz\_txn\_parse fuzzer, basically after parsing txn and making sure it is valid the fuzzer tries to store it in pre-allocated buffers.

The logic is exactly the same as in fd\_bank\_abi\_txn\_init() function.

## Impact Details

Buffer overflow in bank tile during txn processing. Possibility of inter-tile RCE.

## Proof of concept

## Proof of Concept

How to reproduce:

1. get archive by using provided gist link
2. decode and unpack it

```
$ base64 -d poc.txt > arch.tgz
```

3. copy provided fuzzer over fuzz\_txn\_parse.c fuzzer and build firedancer with 'make -j fuzz-test'
4. run fuzz\_txn\_parse fuzzer with included test case:

```
$./fuzz_txn_parse bug2.bin

Max size 18096
Processing tx 0, footprint_bytes=0
Processing tx 1, footprint_bytes=6304
Processing tx 2, footprint_bytes=12608
Processing tx 3, footprint_bytes=18912
=================================================================
==265260==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x629000004be0 at pc 0x64c9aed8ca98 bp 0x7ffe20d73230 sp 0x7ffe20d73228
WRITE of size 1 at 0x629000004be0 thread T0
    #0 0x64c9aed8ca97 in LLVMFuzzerTestOneInput src/ballet/txn/fuzz_txn_parse.c:292:48
    #1 0x64c9aecb2543 in fuzzer::Fuzzer::ExecuteCallback(unsigned char const*, unsigned long) (build/linux/clang/x86_64/fuzz-test/fuzz_txn_parse+0x5a543) (BuildId: 6f80b66d1c683ddf319ed6b58a068270d87210e6)
    #2 0x64c9aec9c2bf in fuzzer::RunOneTest(fuzzer::Fuzzer*, char const*, unsigned long) (build/linux/clang/x86_64/fuzz-test/fuzz_txn_parse+0x442bf) (BuildId: 6f80b66d1c683ddf319ed6b58a068270d87210e6)
    #3 0x64c9aeca2016 in fuzzer::FuzzerDriver(int*, char***, int (*)(unsigned char const*, unsigned long)) (build/linux/clang/x86_64/fuzz-test/fuzz_txn_parse+0x4a016) (BuildId: 6f80b66d1c683ddf319ed6b58a068270d87210e6)
    #4 0x64c9aeccbe32 in main (build/linux/clang/x86_64/fuzz-test/fuzz_txn_parse+0x73e32) (BuildId: 6f80b66d1c683ddf319ed6b58a068270d87210e6)
    #5 0x762ce5e29d8f in __libc_start_call_main csu/../sysdeps/nptl/libc_start_call_main.h:58:16
    #6 0x762ce5e29e3f in __libc_start_main csu/../csu/libc-start.c:392:3
    #7 0x64c9aec96b84 in _start (build/linux/clang/x86_64/fuzz-test/fuzz_txn_parse+0x3eb84) (BuildId: 6f80b66d1c683ddf319ed6b58a068270d87210e6)

0x629000004be0 is located 816 bytes to the right of 18096-byte region [0x629000000200,0x6290000048b0)
allocated by thread T0 here:
    #0 0x64c9aed4ebbe in malloc (build/linux/clang/x86_64/fuzz-test/fuzz_txn_parse+0xf6bbe) (BuildId: 6f80b66d1c683ddf319ed6b58a068270d87210e6)
    #1 0x64c9aed8c260 in LLVMFuzzerTestOneInput src/ballet/txn/fuzz_txn_parse.c:264:26
    #2 0x64c9aecb2543 in fuzzer::Fuzzer::ExecuteCallback(unsigned char const*, unsigned long) (build/linux/clang/x86_64/fuzz-test/fuzz_txn_parse+0x5a543) (BuildId: 6f80b66d1c683ddf319ed6b58a068270d87210e6)
    #3 0x64c9aec9c2bf in fuzzer::RunOneTest(fuzzer::Fuzzer*, char const*, unsigned long) (build/linux/clang/x86_64/fuzz-test/fuzz_txn_parse+0x442bf) (BuildId: 6f80b66d1c683ddf319ed6b58a068270d87210e6)
    #4 0x64c9aeca2016 in fuzzer::FuzzerDriver(int*, char***, int (*)(unsigned char const*, unsigned long)) (build/linux/clang/x86_64/fuzz-test/fuzz_txn_parse+0x4a016) (BuildId: 6f80b66d1c683ddf319ed6b58a068270d87210e6)
    #5 0x64c9aeccbe32 in main (build/linux/clang/x86_64/fuzz-test/fuzz_txn_parse+0x73e32) (BuildId: 6f80b66d1c683ddf319ed6b58a068270d87210e6)
    #6 0x762ce5e29d8f in __libc_start_call_main csu/../sysdeps/nptl/libc_start_call_main.h:58:16

SUMMARY: AddressSanitizer: heap-buffer-overflow src/ballet/txn/fuzz_txn_parse.c:292:48 in LLVMFuzzerTestOneInput
Shadow bytes around the buggy address:
  0x0c527fff8920: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c527fff8930: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c527fff8940: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c527fff8950: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c527fff8960: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
=>0x0c527fff8970: fa fa fa fa fa fa fa fa fa fa fa fa[fa]fa fa fa
  0x0c527fff8980: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c527fff8990: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c527fff89a0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c527fff89b0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c527fff89c0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==265260==ABORTING

```
