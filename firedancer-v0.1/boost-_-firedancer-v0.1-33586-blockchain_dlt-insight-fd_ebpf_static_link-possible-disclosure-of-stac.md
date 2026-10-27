# Boost \_ Firedancer v0.1 33586 - \[Blockchain\_DLT - Insight] fd\_ebpf\_static\_link - possible disclosure

Submitted on Wed Jul 24 2024 00:05:29 GMT-0400 (Atlantic Standard Time) by @gln for [Boost | Firedancer v0.1](https://immunefi.com/bounty/firedancer-boost/)

Report ID: #33586

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/firedancer-io/firedancer/tree/e60d9a6206efaceac65a5a2c3a9e387a79d1d096>

Impacts:

* Process to process RCE between sandboxed tiles

## Description

## Brief/Intro

The function fd\_ebpf\_static\_link() is being used to parse and load BPF programs which are basically hooks for XDP packet processing.

The code is being used by fd\_net tile.

## Vulnerability Details

There is an uninitialized memory issue in fd\_ebpf\_static\_link() function which could lead to disclosure of some parts of stack memory.

Let's look at this function:

```
fd_ebpf_link_opts_t *
fd_ebpf_static_link( fd_ebpf_link_opts_t * const opts,
                     void *                const elf,
                     ulong                 const elf_sz ) {

# define FD_ELF_REQUIRE(c) do { if( FD_UNLIKELY( !(c) ) ) { FD_LOG_WARNING(( "FAIL: %s", #c )); return NULL; } } while(0)

  ...
  ...

  for( uint i=0; i < eh->e_shnum; i++ ) {
    ...
  }

  FD_ELF_REQUIRE( prog_shndx    >=0 );
  FD_ELF_REQUIRE( rel_prog_shndx>=0 );
  FD_ELF_REQUIRE( symtab_shndx  >=0 );
  FD_ELF_REQUIRE( strtab_shndx  >=0 );

  /* Load bytecode */
  ...
  /* Load symbol table */
  ...
  ulong sym_cnt = symtab->sh_size / sizeof(fd_elf64_sym);
  fd_elf64_sym const * sym = (fd_elf64_sym *)( (ulong)elf + symtab->sh_offset );
  FD_ELF_REQUIRE( sym_cnt <= FD_EBPF_MAX_SYM_CNT );

  /* Load string table */
  ...
  ulong rel_cnt = rel_prog->sh_size / sizeof(fd_elf64_rel);
  fd_elf64_rel const * rel = (fd_elf64_rel *)( (ulong)elf + rel_prog->sh_offset );

  /* Create symbol mapping table */

1.  fd_ebpf_known_sym_t sym_mapping[ FD_EBPF_MAX_SYM_CNT ];

  /* Walk symbol table */

2. for( ulong i=0; i<sym_cnt; i++ ) {
    char const * sym_name = fd_elf_read_cstr( elf, elf_sz, strtab->sh_offset + sym[ i ].st_name, 128UL );
    if( !sym_name ) continue;

    /* TODO: O(n^2) complexity -- fine for now as factors are small */

    for( ulong j=0; j<opts->sym_cnt; j++ ) {
      if( 0==strcmp( sym_name, opts->sym[ j ].name ) ) {
        sym_mapping[ i ] = (fd_ebpf_known_sym_t) {
          .known = 1,
          .value = (ulong)(uint)opts->sym[ j ].value
        };
      }
    }
  }

  /* Apply relocations */

  for( ulong i=0; i<rel_cnt; i++ ) {
    FD_ELF_REQUIRE( rel[ i ].r_offset     < prog->sh_size );
    FD_ELF_REQUIRE( rel[ i ].r_offset+8UL <=prog->sh_size );

    ulong r_sym  = FD_ELF64_R_SYM(  rel[ i ].r_info );
    ulong r_type = FD_ELF64_R_TYPE( rel[ i ].r_info );
    FD_ELF_REQUIRE( r_sym < sym_cnt );

3.  ulong S = sym_mapping[ r_sym ].value;

    /* TODO another bounds check? */

    switch( r_type ) {
    case FD_ELF_R_BPF_64_64: {
      ulong r_lo_off = prog->sh_offset + rel[ i ].r_offset +  4UL;
      ulong r_hi_off = prog->sh_offset + rel[ i ].r_offset + 12UL;
      ...
     
      FD_STORE( ulong, insn+0, insn0_post );
      FD_STORE( ulong, insn+8, insn1_post );

      break;
    }
    default:
      FD_LOG_WARNING(( "reloc %lu: Unsupported relocation type %#lx", i, r_type ));
      return NULL;
    }
  }

  opts->bpf    = (void *)( (ulong)elf + prog->sh_offset );
  opts->bpf_sz = fd_ulong_align_dn( prog->sh_size, 8UL );

  return opts;

# undef FD_ELF_REQUIRE
}
```

1. Note that sym\_mapping array is not initialized
2. During execution of loop on line 2. it is possible that sym\_mapping still will not be initialized
3. On line 3. unitialized value from sym\_mapping array will be used and after some modifications written back to a buffer containing bpf program.

As a result it is possible to obtain the access to parts of stack memory.

## Impact Details

Could be used to bypass ASLR/DEP protections from fd\_net tile.

## Proof of concept

## Proof of Concept

How to reproduce:

1. get the archive from provided gist link, it contains slightly modified fd\_ebpf.c file and testcase.

You need to run base64 decoder to unpack it:

```
$ base64 -d gist.txt > repro.tgz
$ tar zxf repro.tgz

```

That is we crash if we notice the use of unitialized values from sym\_mapping array.

2. copy new fd\_ebpf.c over src/waltz/ebpf/fd\_ebpf.c file and build fuzzers
3. run fuzz\_ebpf fuzzer and observe it crash:

```
./fuzz_ebpf test.bin


Running: test1.bin
Setting mapping value for index 5
Last index is 6 
Aplying relocs for index 4
Value s = 4142434451525354
Usage of unitialized value from sym_mapping detected!

```
