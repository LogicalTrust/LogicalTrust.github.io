---
layout: post
title: "[EN] A-Z: B - Pain in the as"
published: true
date: 2019-09-23
author: MichalMateusz
---

## What's *as*?
*as* is a pretty standard assembler widely used in the UNIX world. Probably no further explanation is needed. It reads assembler code and produces compiled objects, no magic in here.

We decided to stick with the GNU implementation, which probably is the most spread nowadays. In case of GNU it's actually a part of the *binutils* package which was our choice for the letter 'B'. Further *as(1)* references point to the GNU one.

It's a very common tool, likely you have got it installed in your operating system. Even if not, there's a big probability that it was compiled partially using the *as(1)* utility.

## Why *as*?

Remember the classical [Trusting trust concept by Ken Thompson](https://dl.acm.org/citation.cfm?id=358210)? We've thought that it would be super useful to find a bunch of bugs that could let us to execute shell from *as*. But if you ever compiled source codes that you haven't read (you didn't do that, right?), malicious code could already change your compiler, so you don't know if your *as(1)* is still *as(1)*. What a crazy time to be alive!

## Our approach

We don't know if you've ever seen the *as* code. It's not easy to analyze and read for many reasons. And yeah, Linus' law states that "*given enough eyeballs, all bugs are shallow*". So we decided to try use old, good dumb fuzzing. We've built small corpus of *.s* files basing on the open source projects (i.a. Linux Kernel, binutils's test suite, \*BSD systems) and put it through cherished radamsa mutator ([did you know that it's written in Scheme?](https://gitlab.com/akihe/radamsa/blob/develop/rad/mutations.scm#L1143)). *as(1)* provides support for many platforms including exotic ones like z8k and rs6000. We've targeted them as well to cover as much code as we can. Tests were done on binutils built with the Address Sanitizer.

## So you say dumb fuzzing was enough?

Hell yeah! We found a bunch of bugs.

{% highlight x %}
# clone the repository
git clone --depth 1 git://sourceware.org/git/binutils-gdb.git

# build in the main directory
binutils: LDFLAGS="-lasan" LDADD="-lasan" CFLAGS="-fsanitize=address -ggdb -O0" ./configure --enable-targets=all
make -j4

# build as for different architectures
cd gas
archs="alpha arm bfd go32 i386 ia64 mcore mips ppc rs6000 sh stgo32 tic30 tic4x tic54x tic80 x86_64 z80 z8k"

for arch in $archs; do
    echo $arch ;
    make clean ;
    LDFLAGS="-lasan" LDADD="-lasan" CFLAGS="-fsanitize=address -ggdb -O0" ./configure --prefix=`pwd`/bin/ --target=$arch-elf ;
    make -j4 ;
    cp as-new as-$arch ;
done
{% endhighlight %}

Tests were executed in a loop:
{% highlight x %}
for i in *.s ; do ~/src/radamsa/bin/radamsa $i | timelimit -T 1 ~/src/binutils-gdb/bin/bin/as ; done
{% endhighlight %}

We've found more than 22 distinct crash places, you can find them in the [submission](https://sourceware.org/bugzilla/show_bug.cgi?id=24538)
.
- inget_any_string_home_mmm_fuzz_binutils_git_binutils-gdb_gas_macro.c:385
- inget_debugseg_name_home_mmm_fuzz_binutils_git_binutils-gdb_gas_dw2gencfi.c:240
- inget_filenum_home_mmm_fuzz_binutils_git_binutils-gdb_gas_dwarf2dbg.c:740
- inget_filenum_home_mmm_fuzz_binutils_git_binutils-gdb_gas_dwarf2dbg.c:743
- inhash_lookup_home_mmm_fuzz_binutils_git_binutils-gdb_gas_hash.c:181
- ini386_intel_simplify_registerconfig_tc-i386-intel.c:289
- ini386_output_nopsconfig_tc-i386.c:1302
- ini386_output_nopsconfig_tc-i386.c:1319
- inia64_find_matching_opcode_home_mmm_fuzz_binutils_git_binutils-gdb_opcodes_ia64-opc.c:627
- inignore_rest_of_line_home_mmm_fuzz_binutils_git_binutils-gdb_gas_read.c:3763
- inmake_invalid_floating_point_numberconfig_atof-ieee.c:146
- inmd_assembleconfig_tc-sh.c:2535
- inmips_lookup_insnconfig_tc-mips.c:14183
- innext_line_shows_parallelconfig_tc-tic54x.c:4198
- ins_arm_eabi_attributeconfig_tc-arm.c:4662
- intic54x_macro_startconfig_tc-tic54x.c:2501
- intic54x_start_labelconfig_tc-tic54x.c:5349
- intic54x_start_line_hookconfig_tc-tic54x.c:4738
- intic54x_undefined_symbolconfig_tc-tic54x.c:5027


Type of bugs found:
- 9 global-buffer-overflow
- 7 heap-buffer-overflow
- 2 stack-buffer-overflow
- 1 heap-use-after-free

Most of the problems were caused by integer jugglery, which shows how tough C can be. Just one example:
{% highlight x %}
[...]
	.intel_syntax noprefix
	vaesdeclast	zmm6, zmm5, ZMMWORD PTR [esp+esi*8-123456]	 # AVX512F,VAES
	vaesdeclast	zmm6, zmm5, ZMMWORD PTR [160491810783548811822244002774]	 # AVX512F,VAES
[...]
{% endhighlight %}

Debug session:
{% highlight x %}
[...]
 Starting program: /home/shm/src/binutils-gdb/bin/bin/as-i386 < example.s
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
{standard input}: Assembler messages:
{standard input}:3: Error: bignum invalid
=================================================================
==1619==ERROR: AddressSanitizer: global-buffer-overflow on address 0x000001363f98 at pc 0x0000004a8868 bp 0x7fffffffdfc0 sp 0x7fffffffdfb0
READ of size 8 at 0x000001363f98 thread T0
    #0 0x4a8867 in i386_intel_simplify_register config/tc-i386-intel.c:289
    #1 0x4a9864 in i386_intel_simplify config/tc-i386-intel.c:500
    #2 0x4a8b98 in i386_intel_simplify_symbol config/tc-i386-intel.c:322
    #3 0x4a8e04 in i386_intel_simplify config/tc-i386-intel.c:355
    #4 0x4a8b98 in i386_intel_simplify_symbol config/tc-i386-intel.c:322
    #5 0x4a90fc in i386_intel_simplify config/tc-i386-intel.c:398
    #6 0x4a9e87 in i386_intel_operand config/tc-i386-intel.c:577
    #7 0x4876f1 in parse_operands config/tc-i386.c:4760
    #8 0x484d42 in md_assemble config/tc-i386.c:4089
    #9 0x445c21 in assemble_one /home/shm/src/binutils-gdb/gas/read.c:711
    #10 0x447357 in read_a_source_file /home/shm/src/binutils-gdb/gas/read.c:1179
    #11 0x409f94 in perform_an_assembly_pass /home/shm/src/binutils-gdb/gas/as.c:1197
    #12 0x40a4d0 in main /home/shm/src/binutils-gdb/gas/as.c:1350
    #13 0x7ffff68bc82f in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x2082f)
    #14 0x4034a8 in _start (/home/shm/src/binutils-gdb/bin/bin/as-i386+0x4034a8)
(...)
 #7  0x00000000004a8868 in i386_intel_simplify_register (e=0x621000015960) at config/tc-i386-intel.c:289

289		   && (i386_regtab[reg_num].reg_type.bitfield.xmmword
 reg_num is set by e->X_md - 1;

 (gdb) print *e
 $2 = {X_add_symbol = 0x0, X_op_symbol = 0x0, X_add_number = 0, X_op = O_constant, X_unsigned = 0, X_extrabit = 0, X_md = 65535}

 (gdb) print reg_num
 $1 = 65534

 thus i386_regtab[reg_num] is accessing table far after its end:

 (gdb) print i386_regtab_size 
 $3 = 281
[...]
{% endhighlight %}
As usually, we left exploitability of found issues to the reader.  In some cases we weren't able to figure out what's wrong with the code, as some parts of it are in old school PERLish flavoured C. :) But all bugs were fixed quickly by the developers, particularly we'd like to thank Nick Clifton for taking care of it (it was fixed within a few days, which is truly impressive!)

## The end of the world is near?

Can you imagine that somebody used full-chain *as(1)*, kernel, persistence exploit against you? (if so, welcome to the paranoid club). Probably nobody backdoored your *as(1)* implementation and nobody will as there are thousand easier ways to hurt you.

The brutal truth behind this article is that the idea for the title was first, a heck of a long time before the research, so we needed to examine *as(1)* to make it happen. :) We hope you enjoy it, and that we left your *as(1)* at least a bit more secure. See you in the next issue!
