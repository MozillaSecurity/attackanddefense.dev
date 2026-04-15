---
layout: post
title:  "Firefox Security & Privacy Newsletter 2026 Q1"
date:   2026-04-15 00:00:00 +0100
author: Frederik Braun, Christoph Kerschbaumer
---

**Welcome to the Q1 2026 edition of the Firefox Security & Privacy Newsletter.**

Security and privacy are foundational to [Mozilla’s manifesto](https://www.mozilla.org/en-US/about/manifesto/) and central to how we build Firefox. In this edition, we highlight key security and privacy work from **Q1 2026**, organized into the following areas:

* **Firefox Product Security & Privacy** — new security and privacy features and integrations in Firefox  
* **Community Engagement** — updates from our security research and bug bounty community  
* **Web Security & Standards** — advancements that help websites better protect their users from online threats

## Preface

Note: Some of the bugs linked below might not be accessible to the general public and restricted to specific work groups. [We de-restrict fixed security bugs after a grace-period](https://firefox-source-docs.mozilla.org/bug-mgmt/processes/fixing-security-bugs.html#keeping-private-information-private), until the majority of our user population have received Firefox updates. If a link does not work for you, please accept this as a precaution for the safety of all Firefox users.

## Firefox Product Security & Privacy

**Collaboration with Anthropic:** A few weeks ago, Anthropic’s Frontier Red Team shared the results of a new AI-assisted vulnerability detection approach. Using this method, we have identified more than a dozen confirmed security issues, each supported by reproducible test cases. Learn more in our blog: [Hardening Firefox with Anthropic’s Red Team](https://blog.mozilla.org/en/firefox/hardening-firefox-anthropic-red-team/). Leveraging our Firefox Security expertise, we ended up finding dozens of additional vulnerabilities that were fixed in the following Firefox updates.

**YouTube coverage of Firefox at pwn2own 2025:** To demonstrate Firefox’s focus on user security and Mozilla’s commitment to openness, we invited LiveOverflow to follow us during the prestigious hacking competition pwn2own last year. LiveOverflow’s four-party documentary provides behind-the-scenes coverage of our quick response to fixing two Firefox 0-day security bugs. The videos go from [preparation (part 1\)](https://www.youtube.com/watch?v=YQEq5s4SRxY), to [exploit analysis (part 2\)](https://www.youtube.com/watch?v=uXW_1hepfT4) and [disclosure (part 3\)](https://www.youtube.com/watch?v=NT1VCmJF3mU), all the way to the [rapid release of a Firefox update (part 4\)](https://www.youtube.com/watch?v=x4CUAuwoZVk) for the 2-day event coverage.

**Trustworthy JavaScript for the Open Web**: Alongside partners from Meta, Proton AG, Cloudflare, and the Freedom of the Press Foundation, we presented our plans to [improve the trustworthiness of JavaScript on the Web](https://www.youtube.com/watch?v=tCLGt0L174c) at [Real World Crypto](https://rwc.iacr.org/2026/acceptedtalks.php).

**SafeBrowsing:** Firefox 147 shipped with SafeBrowsing v5 support, allowing to protect users against malicious URLs. And starting with v149, Firefox blocks and revokes websites permissions for sites on the SafeBrowsing lists ([Bug 1986300](https://bugzilla.mozilla.org/show_bug.cgi?id=1986300)), leveling-up the built-in protection from online threats.

**Stronger XSS Protection through the Sanitizer API:** Starting with v148, Firefox was the first browser to add support for the [Sanitizer API](https://developer.mozilla.org/en-US/docs/Web/API/HTML_Sanitizer_API), helping prevent XSS attacks on the web. Learn more in our blog post, [Goodbye innerHTML, Hello setHTML: Stronger XSS Protection in Firefox 148](https://hacks.mozilla.org/2026/02/goodbye-innerhtml-hello-sethtml-stronger-xss-protection-in-firefox-148/), or tune in to the [ShopTalk Show podcast](https://shoptalkshow.com/704/), where Freddy Braun discusses the details of the Sanitizer API.

**2048-bit Minimum for RSA Certificates:** Firefox now [enforces a minimum 2048-bit RSA key size](https://bugzilla.mozilla.org/show_bug.cgi?id=1137484) for certificates issued by Mozilla’s built-in root CAs. As publicly trusted CAs already meet this requirement, no significant impact to the broader web is expected.

## Community Engagement

**Bug Bounty Program Updates:** As the threat landscape evolves, addressing the increasing volume of AI-assisted security bug reports, we’re evolving our security program alongside it. With continued advances in browser security architecture, our bug bounty program is refining its incentives to prioritize the highest-impact research and the most critical classes of vulnerabilities while focusing on novelty. Learn more in our blogpost: [Bug Bounty Program Updates 2026](https://attackanddefense.dev/2026/03/13/bug-bounty-program-updates-2026.html). We have also just updated our [Bug Bounty hall of fame](https://www.mozilla.org/en-US/security/bug-bounty/hall-of-fame/#year-2026), to list all people who helped us find and fix security vulnerabilities in Q1 of 2026\.

## Web Security & Standards

**Storage-Access Headers**: Firefox 147 is shipping an extension of the Storage Access API to improve both web compatibility and parity with Chrome. These [Storage Access headers](https://developer.mozilla.org/en-US/docs/Web/API/Storage_Access_API#storage_access_headers) allow web pages to opt out of storage isolation upfront and without the need to first load a document.

## Going Forward

As a Firefox user, you automatically benefit from the security and privacy improvements described above through Firefox’s regular automatic updates. If you’re not using Firefox yet, you can download it to enjoy a fast, secure browsing experience—while supporting Mozilla’s mission of a healthy, safe, and accessible web for everyone.

We’d like to thank everyone who helps make Firefox and the open web more secure and privacy-respecting.

See you next time with the **Q2 2026 report**.

— *The Firefox Security and Privacy Teams*

