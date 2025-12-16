---
layout: post
title:  "Attempting Cross Translation Unit Taint Analysis for Firefox"
date:   2025-12-16 10:06:37 +0100
author: Balázs Benics, Tom Ritter, Frederik Braun
---

## Preface

Browser security is a cutting edge frontier for exploit mitigations,
addressing bug classes holistically, and identifying vulnerabilities. Not
everything we try works, and we think it's important to document our
shortcomings in addition to our successes. A responsible project uses all
available tools to find bugs and vulnerabilities before you ship. Besides
many other tools and techniques, Firefox uses Clang Tidy and the Clang
Static Analyzer, including many customized checks for enforcing the coding
conventions in the project. To extend these tools, Mozilla contacted Balázs,
as one of the maintainers of the Clang Static Analyzer, to help address
problems encountered when exploring Cross Translation Unit (CTU) Static
Analysis. Ultimately, we weren't able to make as much headway with this
project as we hoped, but we wanted to contribute our experience to the
community and hopefully inspire future work. Be warned, this is a highly
technical blog post.

The following sections describe some fundamental concepts, such as taint
analysis, CTU, the Clang Static Analyzer engine. This will be followed by
the problem statement and the solution. Finally, some closing words.

Disclaimer: The work described here was sponsored by Mozilla.

## Static Analysis Fundamentals

### Taint analysis

Vulnerabilities often root from using untrusted data in some way. Data from
such sources is called "tainted" in static analysis, and "taint analysis" is the
technique that tracks how such "tainted" values propagate or "flow"
throughout the program.

In short, "Taint sources" introduce a flow, such as reading from a socket. If
a tainted value reaches a "taint sink" then we should report an error. These
"sources" and "sinks" are often configurable.

A [YAML configuration](https://clang.llvm.org/docs/analyzer/user-docs/TaintAnalysisConfiguration.html) file can be used with the Clang Static Analyzer
configuring the taint rules.

### Cross Translation Unit (CTU) analysis

The steps involved in bugs or vulnerabilities might cross file boundaries.
Conventional static analysis tools that operate on a translation-unit basis
would not find the issue. Luckily, the Clang Static Analyzer offers [CTU
mode](https://clang.llvm.org/docs/analyzer/user-docs/CrossTranslationUnit.html)
that loads the relevant pieces of the required translation units to
enhance the contextual view of the analysis target, thus increasing the
covered execution paths. Running CTU needs a bit of setup, but luckily
tools like [scan-build](https://clang.llvm.org/docs/analyzer/user-docs/CommandLineUsage.html#scan-build)
or [CodeChecker](https://github.com/Ericsson/codechecker) have built-in support.

### Path-sensitive analysis

The Clang Static Analyzer implements a path-sensitive symbolic execution.
[Here is an excellent talk](https://www.youtube.com/watch?v=g0Mqx1niUi0)
but let us give a refresher here.

Basically, it interprets the abstract syntax tree (AST) of the analyzed C/C++
program and builds up program facts statement by statement as it
simulates different execution paths of the program. If it sees an `if`
statement, it splits into two execution paths: one where the condition is
assumed to be false, and another one where it's assumed to be true. Loops
are handled slightly differently, but that’s not the point of this post today.

When the engine sees a function call, it will jump to the definition of the
callee (if available) and continue the analysis there with the arguments we
had at the call-site. We call this "inlining" in the Clang Static Analyzer. This
makes this engine inter-procedural, in other words, reason across
functions. Of course, this only works if it knows the callee. This means that
without knowing the pointee of a function pointer or the dynamic type of a
polymorphic object (that has virtual functions), it cannot "inline" the callee,
which in turn means that the engine must conservatively relax the program
facts it gathered so far because they might be changed by the callee.

For example, if we have some allocated memory, and we pass that pointer
to such a function, then the engine must assume that the pointer was
potentially released, and not raise leak warnings after this point.

The conclusion here is that following the control-flow is critical, and virtual
functions limit our ability to reason about this if we don’t know the dynamic
type of objects.

## So, taint analysis for Firefox?

### Firefox has a lot of virtual functions!

We discussed that control-flow is critical for taint analysis, and virtual
functions ruin the control-flow. A browser has almost every code pattern
you can imagine, and it so happens that many of the motivating use cases
for this analysis involve virtual functions that also happen to cross file
boundaries.

### Once upon a time...

It all started by Tom creating a couple of GitHub issues, like 
[#114270](https://github.com/llvm/llvm-project/issues/114270)
(which prompted a couple smaller fixes that are not the subject of this port),
and [#62663](https://github.com/llvm/llvm-project/issues/62663).

This latter one was blocked by not being able to follow the callees of virtual
functions, kicking off this whole subject and the prototype.

### Plotting against virtual functions

#### The idea
Let’s just look at the AST and build the inheritance graph. After that, if we
see a virtual call to `data()`, we could check who overrides this method.

Let’s say only class `A` and `B` overrides this method in the translation unit.
This means we could split the path into 2 and assume that on one path we
call `A::data()` and on the other one `B::data()`.

```c++
// class A... class B deriving from Base
void func(Base *p) {
  p->data(); // ‘p’ might point to an object A or B here.
}
```

This looks nice and simple, and the core of the idea is solid. However, there
are a couple of problems:

 1. One translation unit (TU) might define a class `Derived`, overriding
`data()`, and then pass a `Base` pointer to the other translation unit.
And when that TU is analyzed, it shouldn’t be sure that only class `A`
and `B` overrides `data()` just because it didn’t see `Derived` from the
other TU.
This is the problem with inheritance, which is an "open-set" relation.
One cannot be sure to see the whole inheritance graph all at once.

 2. It’s not only that `Derived` might be in a different TU, but it might be in
a 3rd party library, and dynamically loaded at runtime. In this case,
assuming a finite set of callees for a virtual function would be wrong.

#### Refining the idea

Fixing problem (2) is easy, as we should just assume that the list of
potential callees always has an extra unknown callee, to have an execution
path where the call is conservatively evaluated and do the invalidations -
just like before.

Fixing problem (1) is more challenging because we need whole-program
analysis. We need to create the inheritance graphs of each TU and then
merge them into a unified graph. Once we’ve built that, we can run the
Clang Static Analyzer and start reasoning about the overriders of virtual
functions in the whole project.
Consequently, in the example we discussed before, we would know that
class `A`, `B` and (crucially) `Derived` overrides `data()`. So after the call,
we would have 4 execution paths: `A`, `B`, `Derived` and the last path is for
the unknown case (like potentially dynamically loading some library that
overrides this method).

#### It sounds great, but does it work?

It does! The analysis gives a list of the potential overriders of a virtual
function. The Clang Static Analyzer was modified to do the path splits we
discussed and remember the dynamic type constraints we learn on the
way. There is one catch though.

Some taint flows cross file boundaries, and the Clang Static Analyzer has
CTU to counter this, right?

CTU uses the "[ASTImporter](https://clang.llvm.org/docs/LibASTImporter.html)",
which is known to have infinite recursion,
crashes and incomplete implementation in terms of what constructs it can
import. There are plenty of examples, but one we encountered
was [#123093](https://github.com/llvm/llvm-project/issues/123093).

Usually fixing one of these is time consuming and needs a deep
understanding of the ASTImporter. And even if you fix one of these, there
will be plenty of others to follow.

This patch for "devirtualizing" virtual function calls didn’t really help with the
reliability of the ASTImporter. As the interesting taint flows cross file
boundaries, the benefits of this new feature are unfortunately limited by the
ASTImporter for Firefox.

### Is it available in the Clang Static Analyzer already?

Unfortunately no, and as the contract was over, it is unlikely that these
patches would merge upstream without others splitting up the patches and
doing the labor of proposing them upstream. Note that this whole program
analysis is a brand new feature and it was just a quick prototype to check
the viability.

Upstreaming would likely also need some wider consensus about the
design.

Apparently, whole-project analyses could be important for other domains
besides bug-finding, such as code rewriting tools, which was the motivation
for a 
[recently posted RFC](https://discourse.llvm.org/t/rfc-scalable-static-analysis-framework/88678). 
The proposed framework in that RFC could
potentially also work for the use-case described in this blog post, but it’s
important to highlight that this prototype was built before that RFC and
framework, consequently it’s not using that.

Balázs shared that working on the prototype was really motivating at first,
but as he started to hit the bugs in the ASTImporter - effectively blocking
the prototype - development slowed down. All in all, the prototype proved
that using project-level information, such as "overriders", could enable
better control-flow modeling, but CTU analysis as we have it in Clang today
will show its weaknesses when trying to resolve those calls. Without
resolving these virtual calls, we can’t track taint flows across file boundaries
in the Clang Static Analyzer.

### What does this mean for Firefox?

Not much, unfortunately. If the ASTImporter would work as expected, then
finalizing the prototype would meaningfully improve taint analysis on code
using virtual functions.

You can find the source code at Balázs’ GitHub repo as
[steakhal/llvm-project/devirtualize-for-each-overrider](https://github.com/steakhal/llvm-project/tree/devirtualize-for-each-overrider),
which served well for
exploring and rapid prototyping but it is far from production quality.

### Bonus: We need to talk about the ASTImporter
From the cases Balázs looked at, it seems like qualified names, such as
`std` in `std::unique_ptr` for example, will trigger the import of a `std`
[DeclContext](https://clang.llvm.org/doxygen/classclang_1_1DeclContext.html),
which in turn triggers the import of all the declarations within
that lexical declaration context. In other words, we start importing a lot
more things than strictly necessary to make the `std::` qualification work.
This in turn increases the chances of hitting something that causes a crash
or just fails to import, poisoning the original AST we wanted to import into.
This is likely not how it should work, and might be a good subject to discuss
in the future.

Note that the ASTImporter can be configured to do so-called "[minimal
imports](https://clang.llvm.org/docs/LibASTImporter.html#api)" which is
probably what we should have for the Clang Static
Analyzer, however, 
[this is not set](https://github.com/llvm/llvm-project/blob/4a5692d6b3a6276ef6a8b6a62ef187a16dd3f983/clang/lib/CrossTU/CrossTranslationUnit.cpp#L799-L801),
and setting it would lead to even more crashes. Balázs didn’t investigate this
further, but it might be something to explore in the future.