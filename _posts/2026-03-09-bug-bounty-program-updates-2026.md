---
layout: post
title:  "Bug Bounty Program Updates 2026"
date:   2026-03-13 03:13:37 +0100
author: Frederik Braun, Christoph Kerschbaumer
---

The Firefox bug bounty program is the longest-running security bug bounty program. Born out of [Netscape’s bug bounty program](https://en.wikipedia.org/wiki/Bug_bounty_program#History), we’ve been awarding ingenious security research for over two decades, helping keep our hundreds of millions of users safe.

With the threat landscape changing, we’re updating our security program along with it. As browser security architecture continues to improve, our bug bounty program is evolving to focus incentives on the highest-impact work and the most critical threats.

## Policy changes 

**We are keeping updates to the reporting process to a minimum so that experienced and successful researchers do not need to change how their security bugs are reported right now.**

**Raising the bar for report quality and novelty**

**Quality:** As part of our policy update, we are putting an **increasing emphasis on actionable reports**: Reducing the time spent on validating theoretical concerns and focusing on confirmed vulnerabilities is ensuring faster resolution, faster reward payments and a better Firefox.
Our existing policy already differentiates between high-quality reports and reports that only provide sufficient information to inform us of a new security vulnerability. Going forward, **we will now require security researchers to provide simple and reproducible test cases** for reports to be eligible for bounty rewards (e.g., demonstrable with an [Address Sanitizer build](https://firefox-source-docs.mozilla.org/tools/sanitizer/asan.html#downloading-artifact-builds)). Reports without any credible evidence of a security issue will be assigned a much lower priority.

**Novelty:** In light of a recent increase in duplicate security bug reports and timing issues where researchers were able to race some of our internal tools, **we are reserving an increased time window of seven days for our internal automated methods to find security issues**. While this might affect more bug reports than before, we also believe this encourages researchers to focus on novel research that genuinely benefits advancements in the security posture, rather than research that replicates existing open source tools or results already captured by internal processes. This puts original, high-quality research back in the center of our bug bounty program

**Aligning Security Architecture and Reward Structure**

**Security Architecture:** Firefox is already using multiple utility processes to reduce the level of privilege for security-critical code. We are therefore focusing our rewards for “Highest Impact” / “Sandbox Escape” solely on attacks that compromise the parent process. We are also going to release a separate, sandboxed GPU process on all Operating Systems in 2026, which means that **from now on, we will assess vulnerabilities in our graphics stack according to their sandboxed context, rather than treating them as full sandbox escapes.** These bugs will still be considered as “[High Impact](https://www.mozilla.org/en-US/security/client-bug-bounty/#:~:text=High%20Impact)” memory corruptions. We are also working on better definitions for compromising other processes and will update the bounty program again with more eligible process pairs in the future.

**Changes to our Reward Structure:** We limit our definition of sandbox escapes to true escapes that will allow the attacker to gain control over a higher privileged, unsandboxed process. This definition also excludes memory-reads as well as cross-process exploits targeting other sandboxed processes. As with above, we will continue to rate memory corruptions and attacks on other processes as “[High Impact](https://www.mozilla.org/en-US/security/client-bug-bounty/#:~:text=High%20Impact)“ findings. Consequently, the highest reward will be limited to breaches of the most hardened security boundaries.

# Join our bug bounty program

These updates are a natural step forward in maintaining a highly effective, quality-focused program that addresses the real and evolving security risks facing modern browsers.  
We encourage researchers to engage with these clarified guidelines and focus their efforts on high-impact vulnerabilities and continue working alongside us to strengthen Firefox by focusing on the vulnerabilities that matter most.

We will provide additional guides and tips for improved bug finding and reporting based on previous bug reports in upcoming blog posts. We are also reminding our readers and security community that [we welcome guest articles on our blog](https://attackanddefense.dev/about/) where successful researchers can share their discoveries.
