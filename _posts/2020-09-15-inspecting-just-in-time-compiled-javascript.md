---
title: "Inspecting Just-in-Time Compiled JavaScript"
date: 2020-09-15
categories: 
  - "firefox-internals"
---

The security implications of Just-in-Time (JIT) Compilers in browsers have been getting attention for the [past](http://www.semantiscope.com/research/BHDC2010/BHDC-2010-Paper.pdf) [decade](https://www.nccgroup.com/us/about-us/resources/jit/) and the [references](https://labs.f-secure.com/blog/exploiting-cve-2019-17026-a-firefox-jit-bug/) [to](https://i.blackhat.com/asia-19/Fri-March-29/bh-asia-Li-Using-the-JIT-Vulnerability-to-Pwning-Microsoft-Edge.pdf) [more](https://www.blackhat.com/us-19/briefings/schedule/#the-most-secure-browser-pwning-chrome-from--to--16274) [recent](https://googleprojectzero.blogspot.com/2020/09/jitsploitation-one.html) [resources](https://www.jaybosamiya.com/blog/2019/01/02/krautflare/) [is](https://blog.ret2.io/2018/06/05/pwn2own-2018-exploit-development/) [too](https://github.com/tunz/js-vuln-db) [great](https://sensepost.com/blog/2020/intro-to-chromes-v8-from-an-exploit-development-angle/) [to](https://googleprojectzero.blogspot.com/2018/05/bypassing-mitigations-by-attacking-jit.html) [enumerate](https://www.youtube.com/watch?v=A1FWvfkTgY4). While it’s not the _only_ class of flaw in a browser, it is a common one; and diving deeply into it has a higher barrier to entry than, say, [UXSS injection in the UI](https://blog.mozilla.org/attack-and-defense/2019/12/02/help-test-firefoxs-built-in-html-sanitizer-to-protect-against-uxss-bugs/). This post is about lowering that barrier to entry.

If you want to understand what is happening under the hood in the JIT engine, you can read the source. But that’s kind of a tall order given that the folder [js/](https://searchfox.org/mozilla-central/source/js) contains 500,000+ lines of code. Sometimes it’s easier to treat a target as a black box until you find something you want to dig into deeper. To aid in that endeavor, we’ve landed a feature in the js shell that allows you to get the assembly output of a Javascript function the JIT has processed. Disassembly is supported with the [zydis disassembly library](https://zydis.re/) ([our in-tree version](https://searchfox.org/mozilla-central/source/js/src/zydis)).

To use the new feature; you’ll need to run the js interpreter. You can download the jsshell for any Nightly version of Firefox from [our FTP server](https://ftp.mozilla.org/pub/firefox/nightly/latest-mozilla-central/) - for example [here's the latest Linux x64 jsshell](https://ftp.mozilla.org/pub/firefox/nightly/latest-mozilla-central/jsshell-linux-x86_64.zip). Helpfully, these links always point to the latest version available, historical versions can [also be downloaded](https://ftp.mozilla.org/pub/firefox/nightly/2020/09/).

You can also [build the js shell from source](https://firefox-source-docs.mozilla.org/js/build.html) (which can be done separately from [building Firefox](https://firefox-source-docs.mozilla.org/setup/index.html), but doing the full browser build can also create the shell.)  If building from source, in your [.mozconfig](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Build_Instructions/Configuring_Build_Options), you’ll want to following to get the tools and output you want but also emulate the shell as the Javascript engine is released to users:

`ac_add_options --enable-application=js ac_add_options --enable-js-shell ac_add_options --enable-jitspew ac_add_options --disable-debug ac_add_options --enable-optimize  # If you want to experiment with the debug and optimize flags, # you can build Firefox to different object directories # (and avoid an entire recompilation) mk_add_options MOZ_OBJDIR=@TOPSRCDIR@/obj-nodebug-opt # mk_add_options MOZ_OBJDIR=@TOPSRCDIR@/obj-debug-noopt`

After building the shell or Firefox, fire up \`obj-dir/dist/bin/js\[.exe\]\` and try the following script:

```
function add(x, y) { x = 0+x; y = 0+y; return x+y; }
for(i=0; i<500; i++) { add(2, i); }
print(disnative(add))

```

![](/images/Screen-Shot-2020-09-09-at-4.04.29-PM.png)

You’ll be greeted by an initial line indicating which backend is being used. The possible values and their meanings are:

- Wasm - A WebAssembly function0
- Asmjs - An asm.js module or exported function
- Baseline - indicates the Baseline JIT, a first-pass JIT Engine that collects type information (that can be used by Ion during a subsequent compilation).
- Ion - indicates IonMonkey, a powerful optimizing JIT that performs aggressive compiler optimizations at the cost of additional compile time.

0 The WASM function itself might be Baseline WASM or compiled with an optimizing compiler Cranelift on Nightly; Ion otherwise - it’s not easily enumerated which the assembly function is, but identifying baseline or not becomes easier once you’ve looked at the assembly output a few times.

After running a function [100 times](https://searchfox.org/mozilla-central/rev/2b250967a66886398e5e798371484fd018d88a22/modules/libpref/init/all.js#1079), we will trigger the Baseline compiler; after [1000 times](https://searchfox.org/mozilla-central/rev/2b250967a66886398e5e798371484fd018d88a22/modules/libpref/init/all.js#1085) we will trigger Ion, and after [100,000 times](https://searchfox.org/mozilla-central/rev/2b250967a66886398e5e798371484fd018d88a22/modules/libpref/init/all.js#1086) the full, more expensive, Ion compilation.

For more information about the differences and internals of the JIT Engines, we can point to the following articles:

- [The Baseline Compiler Has Landed (2013)](https://blog.mozilla.org/javascript/2013/04/05/the-baseline-compiler-has-landed/)
- [IonMonkey: Optimizing Away (2014)](https://blog.mozilla.org/javascript/2014/07/15/ionmonkey-optimizing-away/)
- [IonMonkey: Evil on your behalf (2016)](https://blog.mozilla.org/javascript/2016/07/05/ionmonkey-evil-on-your-behalf/)
- [The Baseline Interpreter (2019)](https://hacks.mozilla.org/2019/08/the-baseline-interpreter-a-faster-js-interpreter-in-firefox-70/)
- [The Compiler Compiler Youtube Series (2020)](https://www.youtube.com/playlist?list=PLo3w8EB99pqJVPhmYbYdInBvAGarDavh-)
- [The SpiderMonkey Blog (2019-2020)](https://mozilla-spidermonkey.github.io/blog/)
- [A cartoon intro to WebAssembly (2017)](https://hacks.mozilla.org/2017/02/a-cartoon-intro-to-webassembly/)
- [Making WebAssembly even faster (2018)](https://hacks.mozilla.org/2018/01/making-webassembly-even-faster-firefoxs-new-streaming-and-tiering-compiler/)
- [Calls between JavaScript and WebAssembly are finally fast (2018)](https://hacks.mozilla.org/2018/10/calls-between-javascript-and-webassembly-are-finally-fast-%f0%9f%8e%89/)

Let’s dive into the output we just generated.  Here’s the output of the above script:

Binary Ninja CloudASCII

<iframe id="BinjaCloudEmbed1" src="https://cloud.binary.ninja/embed/cdab8294-7865-453b-a154-29cf3bcf3ee8" width="100%" height="800px"></iframe>

```
; backend=baseline
00000000   jmp 0x0000000000000028                           
00000005   mov $0x7F8A23923000, %rcx                           
0000000F   movq 0x170(%rcx), %rcx                           
00000016   movq %rsp, 0xD0(%rcx)                           
0000001D   movq $0x00, 0xD8(%rcx)                                 
00000028   push %rbp                                 

00000029   mov %rsp, %rbp                |                 
0000002C   sub $0x48, %rsp               | Allocating & initializing                  
00000030   movl $0x00, -0x10(%rbp)       | BaselineFrame structure on                         
00000037   movq 0x18(%rbp), %rcx         | stack.                        
0000003B   and $-0x04, %rcx              | (BaselineCompilerCodeGen::
0000003F   movq 0x28(%rcx), %rcx         |        emitInitFrameFields)                     
00000043   movq %rcx, -0x30(%rbp)        |                         
        
00000047   mov $0x7F8A239237E0, %r11     |                     
00000051   cmpq %rsp, (%r11)             |             
00000054   jbe 0x000000000000006C        |                  
0000005A   mov %rbp, %rbx                | Stackoverflow check         
0000005D   sub $0x48, %rbx               | (BaselineCodeGen::
00000061   push %rbx                     |              emitStackCheck)          
00000062   push $0x5821                  |        
00000067   call 0xFFFFFFFFFFFE1680       |                   

0000006C   mov $0x7F8A226CE0D8, %r11                           
00000076   addq $0x01, (%r11)                           

0000007A   mov $0x7F8A227F6E00, %rax     |                     
00000084   movl 0xC0(%rax), %ecx         |                 
0000008A   add $0x01, %ecx               |           
0000008D   movl %ecx, 0xC0(%rax)         |                 
00000093   cmp $0x3E8, %ecx              |            
00000099   jl 0x00000000000000CC         | Check if we should tier up to
0000009F   movq 0x88(%rax), %rax         | Ion code. 0x38 (1000) is the
000000A6   cmp $0x02, %rax               | threshold. After that check,          
000000AA   jz 0x00000000000000CC         | it checks 'are we already
000000B0   cmp $0x01, %rax               | compiling' and 'is Ion
000000B4   jz 0x00000000000000CC         | compilation impossible'                
000000BA   mov %rbp, %rcx                |          
000000BD   sub $0x48, %rcx               |           
000000C1   push %rcx                     |     
000000C2   push $0x5821                  |        
000000C7   call 0xFFFFFFFFFFFE34B0       |                   

000000CC   movq 0x28(%rbp), %rcx         |                 
000000D0   mov $0x7F8A227F6ED0, %r11     |                     
000000DA   movq (%r11), %rdi             |             
000000DD   callq (%rdi)                  |  
000000DF   movq 0x30(%rbp), %rcx         | Type Inference Type Monitors
000000E3   mov $0x7F8A227F6EE0, %r11     | for |this| and each arg.    
000000ED   movq (%r11), %rdi             | (This overhead is one of the           
000000F0   callq (%rdi)                  |  reasons we're doing
000000F2   movq 0x38(%rbp), %rcx         |  WARP - see below.)                                       
000000F6   mov $0x7F8A227F6EF0, %r11     |                     
00000100   movq (%r11), %rdi             |             
00000103   callq (%rdi)                  |        

00000105   movq 0x30(%rbp), %rbx         |                 
00000109   mov $0xFFF8800000000000, %rcx |                         
00000113   mov $0x7F8A227F6F00, %r11     | Load Int32Value(0) + arg1 and                    
0000011D   movq (%r11), %rdi             | calling an Inline Cache stub            
00000120   callq (%rdi)                  |        
00000122   movq %rcx, 0x30(%rbp)         |                 

00000126   movq 0x38(%rbp), %rbx         |                 
0000012A   mov $0xFFF8800000000000, %rcx | Load Int32Value(0) + arg2 and                        
00000134   mov $0x7F8A227F6F10, %r11     | calling an Inline Cache stub                               
0000013E   movq (%r11), %rdi             |             
00000141   callq (%rdi)                  |        
00000143   movq %rcx, 0x38(%rbp)         |                  

00000147   movq 0x38(%rbp), %rbx         |                 
0000014B   movq 0x30(%rbp), %rcx         |                 
0000014F   mov $0x7F8A227F6F20, %r11     |                     
00000159   movq (%r11), %rdi             | Final Add Inline Cache call
0000015C   callq (%rdi)                  | followed by epilogue code and
0000015E   jmp 0x0000000000000163        | return       
00000163   mov %rbp, %rsp                |          
00000166   pop %rbp                      |    
00000167   jmp 0x0000000000000171        |                  
0000016C   jmp 0xFFFFFFFFFFFE69E0                           
00000171   ret                           
00000172   ud2                           

```

<script type="text/javascript">function showBN1() { document.getElementById('BNCloud1').style.display = 'block'; document.getElementById('Ascii1').style.display = 'none'; document.getElementById('BNCloud1Btn').className += " active"; document.getElementById('Ascii1Btn').className = document.getElementById('Ascii1Btn').className.replace(" active", ""); } function showAscii1() { document.getElementById('BNCloud1').style.display = 'none'; document.getElementById('Ascii1').style.display = 'block'; document.getElementById('BNCloud1Btn').className = document.getElementById('BNCloud1Btn').className.replace(" active", ""); document.getElementById('Ascii1Btn').className += " active"; }</script>

So that’s the Baseline code. It’s the more simplistic JIT in Firefox. What about IonMonkey - its faster, more aggressive big brother?

If we preface our script with setJitCompilerOption("ion.warmup.trigger", 4); then we will induce the Ion compiler to trigger earlier instead of the aforementioned [1000 invocations](https://searchfox.org/mozilla-central/rev/2b250967a66886398e5e798371484fd018d88a22/modules/libpref/init/all.js#1085). You can also set setJitCompilerOption("ion.full.warmup.trigger", 4); to trigger the more aggressive tier for Ion compilation that otherwise kicks in after [100,000 invocations](https://searchfox.org/mozilla-central/rev/2b250967a66886398e5e798371484fd018d88a22/modules/libpref/init/all.js#1086). After triggering the ‘full’ layer, the output will look like:

Binary Ninja CloudASCII

<iframe id="BinjaCloudEmbed2" src="https://cloud.binary.ninja/embed/41afad0b-ea62-4e76-90d8-1c1faf7f1503" width="100%" height="800px"></iframe>

```
; backend=ion
00000000    movq 0x20(%rsp), %rax         |
00000005    shr $0x2F, %rax               |
00000009    cmp $0x1FFF3, %eax            |
0000000E    jnz 0x0000000000000078        |
00000014    movq 0x28(%rsp), %rax         |
00000019    shr $0x2F, %rax               | Type Guards
0000001D    cmp $0x1FFF1, %eax            | for this variable,
00000022    jnz 0x0000000000000078        | arg1, & arg2
00000028    movq 0x30(%rsp), %rax         | 
0000002D    shr $0x2F, %rax               |
00000031    cmp $0x1FFF1, %eax            |
00000036    jnz 0x0000000000000078        |
0000003C    jmp 0x0000000000000041        |

00000041    movl 0x28(%rsp), %eax         |
00000045    movl 0x30(%rsp), %ecx         | Addition
00000049    add %ecx, %eax                |

0000004B    jo 0x000000000000007F         | Overflow Check

00000051    mov $0xFFF8800000000000, %rcx | Box int32 into
0000005B    or %rax, %rcx                 | Int32Value
0000005E    ret
0000005F    nop
00000060    nop
00000061    nop
00000062    nop
00000063    nop
00000064    nop
00000065    nop
00000066    nop
00000067    mov $0x7F8A23903FC0, %r11     |
00000071    push %r11                     |
00000073    jmp 0xFFFFFFFFFFFDED40        |
00000078    push $0x00                    |
0000007A    jmp 0x000000000000008D        |
0000007F    sub %ecx, %eax                | Out-of-line
00000081    jmp 0x0000000000000086        | error handling
00000086    push $0x0D                    | code
00000088    jmp 0x000000000000008D        |
0000008D    push $0x00                    |
0000008F    jmp 0xFFFFFFFFFFFDEC60        |
00000094    ud2                           |

```

<script type="text/javascript">function showBN2() { document.getElementById('BNCloud2').style.display = 'block'; document.getElementById('Ascii2').style.display = 'none'; document.getElementById('BNCloud2Btn').className += " active"; document.getElementById('Ascii2Btn').className = document.getElementById('Ascii2Btn').className.replace(" active", ""); } function showAscii2() { document.getElementById('BNCloud2').style.display = 'none'; document.getElementById('Ascii2').style.display = 'block'; document.getElementById('BNCloud2Btn').className = document.getElementById('BNCloud2Btn').className.replace(" active", ""); document.getElementById('Ascii2Btn').className += " active"; }</script>

There are some other things worth noting.

You can control the behavior of the JITs using environment variables, such as JIT\_OPTION\_fullDebugChecks=false (this will avoid running all the debug checks even in the debug build.)  The full list of JIT Options with documentation is available in [JitOptions.cpp](https://searchfox.org/mozilla-central/source/js/src/jit/JitOptions.cpp).

There are also a variety of command-line flags that can be used in place of environment variables or setJitCompilerOption. For instance \--baseline-eager and \--ion-eager will trigger JIT compilation immediately instead of requiring multiple compilations. (ion-eager triggers ‘full’ compilation, so avoid it if you want the non-full behavior.) \--no-threads or \--ion-offthread-compile=off will disable off-thread compilation that can make it harder to write reliable tests because it adds non-determinism. no-threads turns off all the background threads and implies ion-offthread-compile=off.

Finally, we have a new in-development frontend for Ion: WarpBuilder. You can learn more about WarpBuilder over in the [spidermonkey](https://mozilla-spidermonkey.github.io/blog/2020/03/12/newsletter-3.html) [newsletter](https://mozilla-spidermonkey.github.io/blog/2020/05/11/newsletter-4.html) [or the Bugzilla bug](https://bugzilla.mozilla.org/show_bug.cgi?id=1613592). Enabling warp (by passing \--warp to the js shell executable) significantly reduces the assembly generated, partly because we're simplifying how type information is collected and updated.

If you’ve got other tricks or techniques you use to help you navigate our JIT(s), be sure to [reply to our tweet](https://twitter.com/attackndefense/status/1305859067496275970) so others can find them!
