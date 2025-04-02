---
layout: post
title: "Adding CodeQL and clang to our Bug Bounty Program"
author: Tom Ritter
date: 2019-11-14
categories: 
  - "bug-bounty"
---

_This post first appeared on the [Mozilla Security Blog](https://blog.mozilla.org/security/2019/11/14/adding-codeql-and-clang-to-our-bug-bounty-program/)._

At Github Universe, [Github announced the GitHub Security Lab](https://github.blog/2019-11-14-announcing-github-security-lab/), an initiative to help secure open source software alongside the community and an initial set of partners including Mozilla. As part of this announcement, Github is providing free access to CodeQL, a security research tool which makes it easier to identify flaws in open source software. Mozilla has used these tools privately for the past two years, and have been very impressed and hopeful about how these tools will improve software security. Mozilla recognizes the need to scale security to work automatically, and tighten the feedback loop in the development <-> security auditing/engineering process.

One of the ways we’re supporting this initiative at Mozilla is through renewed investment in automation and static analysis. We think the broader Mozilla community can participate, and we want to encourage it. Today, we’re announcing a new area of our [bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/) to encourage the community to use the CodeQL tools. We are exploring the use of CodeQL tools and will award a bounty - above and beyond our existing bounties - for static analysis work that identifies present or historical flaws in Firefox.

The highlights of the bounty are:

- We will accept static analysis queries written in CodeQL or as clang-based checkers (clang analyzer, clang plugin using the AST API or clang-tidy).
- Each previously unknown security vulnerability your query matches will be eligible for a bug bounty per the normal policy.
- The query itself will also be eligible for a bounty, the amount dependent upon the quality of the submission.
- Queries that match _historical_ issues but do not find new vulnerabilities **are eligible**. This means you can look through our [historical advisories](https://www.mozilla.org/en-US/security/advisories/) to find examples of issues you can write queries for.
- Mozilla and Github’s Bug Bounties are _compatible_ not _exclusive_ so if you meet the requirements of both, you are eligible to receive bounties from both. (More details below.)
- _The full details of this program are available at_ [_our bug bounty program's homepage_](https://www.mozilla.org/en-US/security/client-bug-bounty/)_._

When fixing any security bug, retrospective is an important part of the remediation process which should provide answers to the following questions: Was this the only instance of this issue? Is this flaw representative of a wider systemic weakness that needs to be addressed? And most importantly: can we prevent an issue like this from ever occurring again? Variant analysis, driven manually, is usually the way to answer the first two questions. And static analysis, integrated in the development process, is one of the best ways to answer the third.

Besides our existing clang analyzer checks, we’ve made use of CodeQL over the past two years to do variant analysis. This tool allows identifying bugs both in the context of targeted, zero-false-positive queries, and more expansive results where the manual analysis starts from a more complete and less noise-filled point than simple string matching. To see examples of where we’ve successfully used CodeQL, we have a [meta tracking bug](https://bugzilla.mozilla.org/show_bug.cgi?id=1458117) that illustrates the types of bugs we’ve identified.

We hope that security researchers will try out CodeQL too, and share both their findings and their experience with us. And of course regardless of how you find a vulnerability, you’re always welcome to submit bugs using the regular [bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/). So if you have custom static analysis tools, fuzzers, or just the mainstay of grep and coffee - you’re always invited.

### Getting Started with CodeQL

Github is publishing a guide covering how to use CodeQL at [https://securitylab.github.com/tools/codeql](https://securitylab.github.com/tools/codeql)

### Getting Started with Clang Analyzer

We currently have a number of custom-written checks [in our source tree](https://searchfox.org/mozilla-central/source/build/clang-plugin). So the easiest way to write and run your query is to [build Firefox](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Build_Instructions/Simple_Firefox_build), add ‘ac\_add\_options --enable-clang-plugin’ to your mozconfig, add your check, and then ‘./mach build’ again.

To learn how to add your check, you can review [this recent bug that added a couple of new checks](https://bugzilla.mozilla.org/show_bug.cgi?id=1569681) - it shows how to add a new plugin to Checks.inc, ChecksIncludes.inc, and additionally how to add tests. This particular plugin also adds a couple of attributes that can be used in the codebase, which your plugin may or may not need. Note that depending on how you view the diffs, it may appear that the author modified existing files, but actually they copied an existing file, then modified the copy.

### Future of CodeQL and clang within our Bug Bounty program

We retain the ability to be flexible. We’re planning to evaluate the effectiveness of the program when we reach $75,000 in rewards or after a year. After all, this is something new for us and for the bug bounty community. We—and Github—welcome your communication and feedback on the plan, especially candid feedback. If you’ve developed a query that you consider more valuable than what you think we’d reward - we would love to hear that. (If you’re keeping the query, hopefully you’re submitting the bugs to us so we can see that we are not meeting researcher expectations on reward.) And if you spent hours trying to write a query but couldn’t get over the learning curve - tell us and show us what problems you encountered!

We’re excited to see what the community can do with CodeQL and clang; and how we can work together to improve on our ability to deliver a browser that answers to no one but you.
