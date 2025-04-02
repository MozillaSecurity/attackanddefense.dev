---
layout: post
title: "WebAssembly and Back Again: Fine-Grained Sandboxing in Firefox 95"
date: 2021-12-06
categories: 
  - "firefox-internals"
  - "hardening"
  - "security-internals"
---

In Firefox 95, we're shipping a novel sandboxing technology called [RLBox](https://plsyssec.github.io/rlbox_sandboxing_api/sphinx/) — developed in collaboration with researchers at the University of California San Diego and the University of Texas — that makes it easy and efficient to isolate subcomponents to make the browser more secure. This technology opens up new opportunities beyond what's been possible with traditional process-based sandboxing, and we look forward to expanding its usage and (hopefully) seeing it adopted in other browsers and software projects.

This technique, which uses WebAssembly to isolate potentially-buggy code, builds on the [prototype](https://hacks.mozilla.org/2020/02/securing-firefox-with-webassembly/) we shipped last year to Mac and Linux users. Now, we’re bringing that technology to all supported Firefox platforms (desktop and mobile), and isolating five different modules: [Graphite](https://scripts.sil.org/cms/scripts/page.php?site_id=projects&item_id=graphite_home), [Hunspell](http://hunspell.github.io/), [Ogg](https://xiph.org/ogg/), [Expat](https://libexpat.github.io/) and [Woff2](https://github.com/google/woff2) \[1\].

Going forward, we can treat these modules as untrusted code, and — assuming we did it right — even a zero-day vulnerability in any of them should pose no threat to Firefox. Accordingly, we’ve updated our [bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/#exploit-mitigation-bounty) to pay researchers for bypassing the sandbox even without a vulnerability in the isolated library.

## **The Limits of Process Sandboxing**

All major browsers run Web content in its own sandboxed process, in theory preventing it from exploiting a browser vulnerability to compromise your computer. On desktop operating systems, Firefox also isolates each site in its own process in order to protect sites from each other.

Unfortunately, threat actors routinely attack users by chaining together two vulnerabilities — one to compromise the sandboxed process containing the malicious site, and another to escape the sandbox \[2\]. To keep our users secure against the most well-funded adversaries, we need multiple layers of protection.

Having already isolated things along trust boundaries, the next logical step is to isolate across functional boundaries. Historically, this has meant hoisting a subcomponent into its own process. For example, Firefox runs audio and video codecs in a dedicated, locked-down process with a limited interface to the rest of the system. However, there are some serious limitations to this approach. First, it requires decoupling the code and making it asynchronous, which is usually time-consuming and may impose a performance cost. Second, processes have a fixed memory overhead, and adding more of them increases the memory footprint of the application.

For all of these reasons, nobody would seriously consider hoisting something like the XML parser into its own process. To isolate at that level of granularity, we need a different approach.

## **Isolating with RLBox**

This is where RLBox comes in. Rather than hoisting the code into a separate process, we instead compile it into WebAssembly and then compile that WebAssembly into native code. This doesn’t result in us shipping any .wasm files in Firefox, since the WebAssembly step is only an intermediate representation in our build process.

However, the transformation places two key restrictions on the target code: it can’t jump to unexpected parts of the rest of the program, and it can’t access memory outside of a specified region. Together, these restrictions [make it safe to share an address space](http://www.cse.psu.edu/~gxt29/papers/sfi-final.pdf) ([including the stack](https://arxiv.org/abs/2105.00033)) between trusted and untrusted code, allowing us to run them in the same process largely as we were doing before. This, in turn, makes it easy to apply without major refactoring: the programmer only needs to sanitize any values that come from the sandbox (since they could be maliciously-crafted), a task which RLBox makes easy with a [tainting layer](https://hacks.mozilla.org/2020/02/securing-firefox-with-webassembly/).

The first step in this transformation is straightforward: we use [Clang](https://clang.llvm.org/) to compile Firefox, and Clang knows how to emit WebAssembly, so we simply need to switch the output format for the given module from native code to wasm. For the second step, our prototype implementation used [Cranelift](https://github.com/bytecodealliance/wasmtime/tree/main/cranelift). Cranelift is excellent, but a second native code generator added complexity — and we realized that it would be simpler to just map the WebAssembly back into something that our existing build system could ingest.

We accomplished this with [wasm2c](https://github.com/WebAssembly/wabt/tree/main/wasm2c), which performs a straightforward translation of WebAssembly into equivalent C code, which we can then feed back into Clang along with the rest of the Firefox source code. This approach is very simple, and automatically enables a number of important features that we support for regular Firefox code: profile-guided optimization, inlining across sandbox boundaries, crash reporting, debugger support, source-code indexing, and likely other things that we have yet to appreciate.

## **Next Steps**

RLBox is a big win for us on several fronts: it protects our users from accidental defects as well as supply-chain attacks, and it reduces the need for us to scramble when such issues are disclosed upstream. As such, we intend to continue applying to more components going forward. Some components are not a good fit for this approach — either because they depend too much on sharing memory with the rest of the program, or because they’re too performance-sensitive to accept the modest overhead incurred — but we’ve identified a number of other good candidates.

Moreover, we hope to see this technology make its way into other browsers and software projects to make the ecosystem safer. [RLBox](https://github.com/PLSysSec/rlbox_sandboxing_api) is a standalone project that’s designed to be very modular and easy-to-use, and the team behind it would welcome other use-cases.

Speaking of the team: I’d like to thank [Shravan Narayan](https://shravanrn.com/), [Deian Stefan](https://cseweb.ucsd.edu/~dstefan/), and [Hovav Shacham](https://www.cs.utexas.edu/~hovav/) for their tireless work in bringing this work from research concept to production. Shipping to hundreds of millions of users is hard, and they did some seriously impressive work.

Read more about RLBox and this announcement on the [UC San Diego Jacobs School of Engineering website](https://jacobsschool.ucsd.edu/news/release/3374).

\---

\[1\] Cross-platform sandboxing for Graphite, Hunspell, and Ogg is shipping in Firefox 95, while Expat and Woff2 will ship in Firefox 96.

\[2\] By using a syscall to to exploit a vulnerability in the OS, or by using an IPC message to exploit a vulnerability in a process hosting more-privileged parts of the browser.

\---
