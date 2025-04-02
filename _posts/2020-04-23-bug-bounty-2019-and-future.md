---
layout: post
title: "Firefox’s Bug Bounty in 2019 and into the Future"
author: Tom Ritter
date: 2020-04-23
categories: 
  - "bug-bounty"
---

Welcome to Mozilla’s new Attack & Defense blog. We’re going to use this blog as a vehicle for tailored content specifically for engineers, security researchers, and Firefox bug bounty participants. We’ll be highlighting improvements to the bug bounty program, as well as posting guides on how to test different parts of Firefox. Please follow us via [RSS](https://blog.mozilla.org/attack-and-defense/comments/feed/) or on [twitter](https://twitter.com/attackndefense), where we’ll be boosting tweets relevant to browser security: exploitation walkthroughs, conference talks, and the like.

Firefox has one of the oldest security bug bounties on the internet, dating back to 2004.  From 2017-2019, we paid out $965,750 to researchers across 348 bugs, making the average payout $2,775 - but as you can see in the graph below, our most common payout was actually $4,000!

[![](/images/Frequency-of-Bounty-Payment-Amounts-2017-2019.png)](http://blog.mozilla.org/attack-and-defense/files/2020/03/Frequency-of-Bounty-Payment-Amounts-2017-2019.png)

After [adding a new static analysis bounty](https://blog.mozilla.org/attack-and-defense/2019/11/14/adding-codeql-and-clang-to-our-bug-bounty-program/) late last year, we're excited to further expand our bounty program in the coming year, as well as provide an on-ramp for more participants. We're updating our bug bounty policy and payouts to make it more appealing to researchers and reflect the more hardened security stance we adopted after moving to a multi-process, sandboxed architecture. Additionally, we'll be publishing more posts about how to get started testing Firefox - which is something we began by [talking about the HTML Sanitization we rely on to prevent UXSS](https://blog.mozilla.org/security/2019/12/02/help-test-firefoxs-built-in-html-sanitizer-to-protect-against-uxss-bugs/) . By following the instructions there you can immediately start trying to bypass our sanitizer using your existing Firefox installation in less than a minute.

Firstly, we're amending our current policy to be more friendly and allowing duplicate submissions. Presently, we have a 'first reporter wins' policy, which can be very frustrating if you are fuzzing our Nightly builds (which we encourage you to do!) and you find and report a bug mere hours after another reporter. From now on, we will split the bounty between all duplicates submitted within 72 hours of the first report; with prorated amounts for higher quality reports. We hope this will encourage more people to fuzz our [Nightly ASAN](https://firefox-source-docs.mozilla.org/tools/sanitizer/asan.html) builds.

Besides rewarding duplicate submissions, we're clarifying our payout criteria and raising the payouts for higher impact bugs. Now, sandbox escapes and related bugs will be eligible for a baseline $8,000, with a high quality report up to $10,000.  Additionally, proxy bypass bugs are eligible for a baseline of $3,000, with a high quality report up to $5,000. And most submissions are above baseline: previously the typical range for high impact bugs was $3,000 - $5,000 but as you can see in the graphic above, the most common payout for those bugs was actually $4,000!  You can find full details of the criteria and amounts on our [Client Bug Bounty](https://www.mozilla.org/en-US/security/client-bug-bounty/) page.

Again, we want to emphasize - if it wasn't already - that a bounty amount is not determined based on your _initial_ submission, but rather on the outcome of the discussion with developers. So improving test cases post-submission, figuring out if an engineer's speculation is founded or not, or other assistance that helps resolve the issue _will_ increase your bounty payout.

And if you're inclined, you are able to donate your bounty to charity - if you wish to do so and have payment info on file, please indicate this in new bugs you file and we will contact you.

We're excited to launch this new blog that is targeted directly at security researchers who are interested in new developments in Mozilla's Bug Bounty, and guides, tips, and tricks for finding bugs in Firefox.  If you're interested in more general Mozilla Security news, such as audit results, updates to the CA policies, improvements and movements in Web Security Standards, you'll be interested in following the Mozilla Security blog too.
