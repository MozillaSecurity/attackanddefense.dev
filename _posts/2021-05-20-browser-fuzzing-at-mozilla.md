---
layout: post
title: "Browser fuzzing at Mozilla"
author: Tyson Smith, Jesse Schwartzentruber, Sylvestre Ledru
date: 2021-05-20
tags: 
  - "automation"
  - "fuzzing"
  - "testing"
---

 

## Introduction

Mozilla has been [fuzzing Firefox](https://firefox-source-docs.mozilla.org/tools/fuzzing/index.html) and its underlying components for a while. It has proven to be one of the most efficient ways to identify quality and security issues. In general, we apply fuzzing on [different levels](https://firefox-source-docs.mozilla.org/tools/fuzzing/index.html#levels-of-fuzzing-in-firefox-gecko): there is fuzzing the browser as a whole, but a significant amount of time is also spent on [fuzzing isolated code](https://firefox-source-docs.mozilla.org/tools/fuzzing/fuzzing_interface.html) (e.g. with [libFuzzer](https://www.llvm.org/docs/LibFuzzer.html)) or whole components such as the [JS engine](https://firefox-source-docs.mozilla.org/js/build.html) using separate shells. In this blog post, we will talk specifically about _browser_ fuzzing only, and go into detail on the pipeline we’ve developed. This single pipeline is the result of years of work that the fuzzing team has put into aggregating our browser fuzzing efforts to provide consistently actionable issues to developers and to ease integration of internal and external fuzzing tools as they become available.

![Diagram showing interaction of systems used in Mozilla's browser fuzzing workflow](/images/Fuzzing-pipeline1.svg)

## Build instrumentation

To be as effective as possible we make use of different methods of detecting errors. These include sanitizers such as [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) (with LeakSanitizer), [ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html), and [UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html), as well as using debug builds that enable assertions and other runtime checks. We also make use of debuggers such as [rr](https://rr-project.org/) and [Valgrind](https://valgrind.org/). Each of these tools provides a different lens to help uncover specific bug types, but many are incompatible with each other or require their own custom build to function or provide optimal results. Besides providing debugging and error detection, some tools cannot work without build instrumentation, such as code coverage and [libFuzzer](https://www.llvm.org/docs/LibFuzzer.html). Each operating system and architecture combination requires a unique build and may only support a subset of these tools.

Last, each variation has multiple active branches including Release, Beta, Nightly, and Extended Support Release (ESR). The [Firefox CI](https://firefox-ci-tc.services.mozilla.com/tasks/index/gecko.v2.mozilla-central.latest.firefox) Taskcluster instance builds each of these periodically.

## Downloading builds

Taskcluster makes it easy to find and download the latest build to test. We discussed above the number of variants created by different instrumentation types, and we need to fuzz them in automation. Because of the large number of combinations of builds, artifacts, architectures, operating systems, and unpacking each, downloading is a non-trivial task.

To help reduce the complexity of build management, we developed a tool called [fuzzfetch](https://github.com/MozillaSecurity/fuzzfetch). Fuzzfetch makes it easy to specify the required build parameters and it will download and unpack the build. It also supports downloading specified revisions to make it useful with bisection tools.

## How we generate the test cases

As the goal of this blog post is to explain the whole pipeline, we won’t spend much time explaining fuzzers. If you are interested, please read ["Fuzzing Firefox with WebIDL"](https://hacks.mozilla.org/2020/04/fuzzing-with-webidl/) and the [in-tree documentation](https://firefox-source-docs.mozilla.org/tools/fuzzing/index.html). We use a combination of publicly available and custom-built fuzzers to generate test cases.

## How we execute, report, and scale

For fuzzers that target the browser, [Grizzly](https://github.com/MozillaSecurity/grizzly) manages and runs test cases and monitors for results. [Creating an adapter](https://github.com/MozillaSecurity/grizzly/wiki/Writing-an-Adapter) allows us to easily run existing fuzzers in Grizzly.

![Simplified Python code for a Grizzly adaptor using an external fuzzer.](/images/example2.svg)

To make full use of available resources on any given machine, we run multiple instances of Grizzly in parallel.

For each fuzzer, we create containers to encapsulate the configuration required to run it. These exist in the [Orion monorepo](https://github.com/MozillaSecurity/orion). Each fuzzer has a configuration with deployment specifics and resource allocation depending on the priority of the fuzzer. Taskcluster continuously deploys these configurations to distribute work and manage fuzzing nodes.

Grizzly Target handles the detection of issues such as hangs, crashes, and other defects. Target is an interface between Grizzly and the browser. Detected issues are automatically packaged and reported to a [FuzzManager](https://github.com/MozillaSecurity/FuzzManager) server. The FuzzManager server provides automation and a UI for triaging the results.

Other more targeted fuzzers use [JS shell](https://firefox-source-docs.mozilla.org/js/build.html) and [libFuzzer](https://www.llvm.org/docs/LibFuzzer.html) based targets use the [fuzzing interface](https://firefox-source-docs.mozilla.org/tools/fuzzing/fuzzing_interface.html). Many third-party libraries are also fuzzed in [OSS-Fuzz](https://google.github.io/oss-fuzz/). These deserve mention but are outside of the scope of this post.

## Managing results

Running multiple fuzzers against various targets at scale generates a large amount of data. These crashes are not suitable for direct entry into a bug tracking system like [Bugzilla](https://bugzilla.mozilla.org/home). We have tools to manage this data and get it ready to report.

The [FuzzManager](https://github.com/MozillaSecurity/FuzzManager) client library filters out crash variations and duplicate results before they leave the fuzzing node. Unique results are reported to a FuzzManager server. The FuzzManager web interface allows for the creation of signatures that help group reports together in buckets to aid the client in detecting duplicate results.

Fuzzers commonly generate test cases that are hundreds or even thousands of lines long. FuzzManager buckets are automatically scanned to queue reduction tasks in Taskcluster. These reduction tasks use [Grizzly Reduce](https://github.com/MozillaSecurity/grizzly/wiki/grizzly-reduce) and [Lithium](https://github.com/MozillaSecurity/lithium) to apply different reduction strategies, often removing the majority of the unnecessary data. Each bucket is continually processed until a successful reduction is complete. Then an engineer can do a final inspection of the minimized test case and attach it to a bug report. The final result is often used as a crash test in the Firefox test suite.

![Animation showing an example testcase reduction using Grizzly](/images/reduce-3.svg)

Code coverage of the fuzzer is also measured periodically. FuzzManager is used again to collect code coverage data and generate coverage reports.

## Creating optimal bug reports

Our goal is to create actionable bug reports to get issues fixed as soon as possible while minimizing overhead for developers.

We do this by providing:

- crash information such as logs and a stack trace
- build and environment information
- reduced test case
- [Pernosco](https://www.pernos.co/) session
- regression range (bisections via [Bugmon](https://github.com/MozillaSecurity/bugmon))
- verification via Bugmon

[Grizzly Replay](https://github.com/MozillaSecurity/grizzly/wiki/Grizzly-Replay) is a tool that forms the basic execution engine for Bugmon and [Grizzly Reduce](https://github.com/MozillaSecurity/grizzly/wiki/grizzly-reduce), and makes it easy to collect [rr](https://rr-project.org/) traces to submit to Pernosco. It makes re-running browser test cases easy both in automation and for manual use. It simplifies working with stubborn test cases and test cases that trigger multiple results.

As mentioned, we have also been making use of Pernosco. Pernosco is a tool that provides a web interface for rr traces and makes them available to developers without the need for direct access to the execution environment. It is an amazing tool developed by a company of the same name which significantly helps to debug massively parallel applications. It is also very helpful when test cases are too unreliable to reduce or attach to bug reports. Creating an rr trace and uploading it can make stalled bug reports actionable.

The combination of Grizzly and Pernosco have had the added benefit of making infrequent, hard to reproduce issues, actionable. A test case for a very inconsistent issue can be run hundreds or thousands of times until the desired crash occurs under rr. The trace is automatically collected and ready to be submitted to Pernosco and fixed by a developer, instead of being passed over because it was not actionable.

## How we interact with developers

To request new features get a proper assessment, the fuzzing team can be reached at [fuzzing@mozilla.com](mailto:fuzzing@mozilla.com) or on [Matrix](https://chat.mozilla.org/#/room/#fuzzing:mozilla.org). This is also a great way to get in touch for any reason. We are happy to help you with any fuzzing related questions or ideas. We will also reach out when we receive information about new initiatives and features that we think will require attention. Once fuzzing of a component begins, we communicate mainly via Bugzilla. As mentioned, we strive to open actionable issues or enhance existing issues logged by others.

[Bugmon](https://hacks.mozilla.org/2020/12/analyzing-bugzilla-testcases-with-bugmon) is used to automatically bisect regression ranges. This notifies the appropriate people as quickly as possible and verifies bugs once they are marked as FIXED. Closing a bug automatically removes it from FuzzManager, so if a similar bug finds its way into the code base, it can be identified again.

Some issues found during fuzzing will prevent us from effectively fuzzing a feature or build variant. These are known as [fuzz-blockers](https://firefox-source-docs.mozilla.org/tools/fuzzing/index.html#fuzz-blockers), and they come in a few different forms. These issues may seem benign from a product perspective, but they can block fuzzers from targeting important code paths or even prevent fuzzing a target altogether. Prioritizing these issues appropriately and getting them fixed quickly is very helpful and much appreciated by the fuzzing team.

[PrefPicker](https://github.com/MozillaSecurity/prefpicker) manages the set of Firefox preferences used for fuzzing. When adding features behind a pref, consider adding it to the PrefPicker fuzzing template to have it enabled during fuzzing. Periodic audits of the PrefPicker [fuzzing template](https://github.com/MozillaSecurity/prefpicker/blob/master/prefpicker/templates/browser-fuzzing.yml) can help ensure areas are not missed and resources are used as effectively as possible.

## Measuring success

As in other fields, measurement is a key part of evaluating success. We leverage the meta bug feature of Bugzilla to help us keep track of the issues identified by fuzzers. We strive to have a meta bug per fuzzer and for each new component fuzzed.

For example, the [meta bug for Domino](https://bugzilla.mozilla.org/show_bug.cgi?id=domino) lists all the issues (over 1100!) identified by this tool. Using this Bugzilla data, we are able to show the impact over the years of our various fuzzers.

\[caption id="attachment\_47027" align="aligncenter" width="590"\]![Bar graph showing number of bugs reported by Domino over time](/images/Fuzzing-by-tools-Screenshot-221.svg) Number of bugs reported by Domino over time\[/caption\]

These dashboards help evaluate the return on investment of a fuzzer.

## Conclusion

There are many components in the fuzzing pipeline. These components are constantly evolving to keep up with changes in debugging tools, execution environments, and browser internals. Developers are always adding, removing, and updating browser features. Bugs are being detected, triaged, and logged. Keeping everything running continuously and targeting as much code as possible requires constant and ongoing efforts.

If you work on Firefox, you can help by keeping us informed of new features and initiatives that may affect or require fuzzing, by prioritizing [fuzz-blockers](https://firefox-source-docs.mozilla.org/tools/fuzzing/index.html#fuzz-blockers), and by curating [fuzzing preferences](https://github.com/MozillaSecurity/prefpicker/blob/master/prefpicker/templates/browser-fuzzing.yml) in PrefPicker. If fuzzing interests you, please take part in the [bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/). Our [tools are available](https://github.com/MozillaSecurity/) publicly, and we encourage bug hunting.
