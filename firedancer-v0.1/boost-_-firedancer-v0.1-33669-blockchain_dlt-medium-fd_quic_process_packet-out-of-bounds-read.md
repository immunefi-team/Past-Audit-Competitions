# Boost \_ Firedancer v0.1 33669 - \[Blockchain\_DLT - Medium] fd\_quic\_process\_packet out of bounds read

Submitted on Fri Jul 26 2024 00:53:15 GMT-0400 (Atlantic Standard Time) by @gln for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #33669

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Process to process RCE between sandboxed tiles

## Description

## Brief/Intro

QUIC tile fails to validate data received from Net tile.

If udp packet length is too short, out of bounds read might be triggered.

## Vulnerability Details

Lets look at the code <https://github.com/firedancer-io/firedancer/blob/main/src/waltz/quic/fd\\_quic.c#L2782>

```

void
fd_quic_process_packet( fd_quic_t * quic,
                        uchar *     data,
                        ulong       data_sz ) {

  fd_quic_state_t * state = fd_quic_get_state( quic );

  ulong rc = 0;

  /* holds the remainder of the packet*/
  uchar * cur_ptr = data;
  ulong   cur_sz  = data_sz;

  if( FD_UNLIKELY( data_sz > 0xffffu ) ) {
    /* sanity check */
    return;
  }

  fd_quic_pkt_t pkt = { .datagram_sz = (uint)data_sz };

  pkt.rcv_time = state->now;

  rc = fd_quic_decode_eth( pkt.eth, cur_ptr, cur_sz );
  if( FD_UNLIKELY( rc == FD_QUIC_PARSE_FAIL ) ) {
    /* TODO count failure, log-debug failure */
    return;
  }

  if( FD_UNLIKELY( pkt.eth->net_type != FD_ETH_HDR_TYPE_IP ) ) {
    FD_DEBUG( FD_LOG_DEBUG(( "Invalid ethertype: %4.4x", pkt.eth->net_type )) );
    return;
  }

  cur_ptr += rc;
  cur_sz  -= rc;

  rc = fd_quic_decode_ip4( pkt.ip4, cur_ptr, cur_sz );
  if( FD_UNLIKELY( rc == FD_QUIC_PARSE_FAIL ) ) {
    /* TODO count failure, log-debug failure */
    return;
  }

  if( FD_UNLIKELY( pkt.ip4->protocol != FD_IP4_HDR_PROTOCOL_UDP ) ) {
    return;
  }

  if( FD_UNLIKELY( pkt.ip4->net_tot_len > cur_sz ) ) {
    return;
  }

  cur_ptr += rc;
  cur_sz  -= rc;

1.  rc = fd_quic_decode_udp( pkt.udp, cur_ptr, cur_sz );
  if( FD_UNLIKELY( rc == FD_QUIC_PARSE_FAIL ) ) {
    return;
  }

  if( FD_UNLIKELY( pkt.udp->net_len > cur_sz ) ) {
    return;
  }

  cur_ptr += rc;
2.  cur_sz   = pkt.udp->net_len - rc; 
  ...

  int long_pkt = !!( (uint)cur_ptr[0] & 0x80u );

  uint version = 0;

  if( long_pkt ) {
    version = DECODE_UINT32( cur_ptr + 1 );

    ...

3.  while(1) {
      if( FD_UNLIKELY( cur_sz < FD_QUIC_SHORTEST_PKT ) ) return;

      int short_pkt = !( (uint)cur_ptr[0] & 0x80u );

      if( FD_UNLIKELY( short_pkt ) ) break;

      uint cur_version = DECODE_UINT32( cur_ptr + 1 );
      ...
      ...
```

1. Note that fd\_quic\_decode\_udp() returns 8 (size of udp header)
2. If pkt.udp->net\_len is less than 8, cur\_sz will be set to a negative value (large positive)
3. while loop will go out of bounds when trying to read quic packets

## Impact Details

Might be possible to leak parts of memory from QUIC tile.

Denial of service is possible as well.

## Proof of concept

## Proof of Concept

Most simple way to reproduce this issue is to use fuzz\_quic\_wire fuzzer with memory sanitizer.

How to reproduce:

1. build fuzz tests with memory sanitizer, edit config/extra/with-asan.mk and change variables to:

```
CPPFLAGS+=-fsanitize=memory

LDFLAGS+=-fsanitize=memory

```

2. build with command:

```

EXTRAS="asan fuzz" make -j fuzz-test
```

3. get proof of concept by using provided gist link
4. unpack and decode it:

```
$ base64 -d test.txt > test.bin
```

5. run fuzz\_quic\_wire test:

```
$ ./fuzz_quic_wire test.bin


Running: test.bin
==325630==WARNING: MemorySanitizer: use-of-uninitialized-value
    #0 0x5fc853ce6dae in fd_quic_process_packet /src/waltz/quic/fd_quic.c:2889:7
    #1 0x5fc853cbc939 in send_udp_packet /src/waltz/quic/tests/fuzz_quic_wire.c:97:3
    #2 0x5fc853cbc939 in LLVMFuzzerTestOneInput /src/waltz/quic/tests/fuzz_quic_wire.c:189:3
    #3 0x5fc853c1a553 in fuzzer::Fuzzer::ExecuteCallback(unsigned char const*, unsigned long) (/build/linux/clang/x86_64/fuzz-test/fuzz_quic_wire/fuzz_quic_wire+0x80553) (BuildId: dbb992a413a9e6c41b1271a0480fad6bf3051f9f)
    #4 0x5fc853c042cf in fuzzer::RunOneTest(fuzzer::Fuzzer*, char const*, unsigned long) (/build/linux/clang/x86_64/fuzz-test/fuzz_quic_wire/fuzz_quic_wire+0x6a2cf) (BuildId: dbb992a413a9e6c41b1271a0480fad6bf3051f9f)
    #5 0x5fc853c0a026 in fuzzer::FuzzerDriver(int*, char***, int (*)(unsigned char const*, unsigned long)) (/build/linux/clang/x86_64/fuzz-test/fuzz_quic_wire/fuzz_quic_wire+0x70026) (BuildId: dbb992a413a9e6c41b1271a0480fad6bf3051f9f)
    #6 0x5fc853c33e42 in main (/build/linux/clang/x86_64/fuzz-test/fuzz_quic_wire/fuzz_quic_wire+0x99e42) (BuildId: dbb992a413a9e6c41b1271a0480fad6bf3051f9f)
    #7 0x7e7a43629d8f in __libc_start_call_main csu/../sysdeps/nptl/libc_start_call_main.h:58:16
    #8 0x7e7a43629e3f in __libc_start_main csu/../csu/libc-start.c:392:3
    #9 0x5fc853bfeb94 in _start (/build/linux/clang/x86_64/fuzz-test/fuzz_quic_wire/fuzz_quic_wire+0x64b94) (BuildId: dbb992a413a9e6c41b1271a0480fad6bf3051f9f)

SUMMARY: MemorySanitizer: use-of-uninitialized-value /src/waltz/quic/fd_quic.c:2889:7 in fd_quic_process_packet

```
