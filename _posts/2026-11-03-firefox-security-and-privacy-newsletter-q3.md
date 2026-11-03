---
layout: post
title:  "Firefox Security & Privacy Newsletter 2026 Q3"
date:   2026-11-03 00:00:00 +0100
author: Frederik Braun, Christoph Kerschbaumer
---

Welcome to the Q3 2026 edition of the Firefox Security & Privacy Newsletter.

Security and privacy are core principles of [Mozilla’s Manifesto](https://www.mozilla.org/en-US/about/manifesto/) and remain at the heart of Firefox’s development. This edition highlights key initiatives from Q3 2026 across three areas:

* **Firefox Security:** New security features, platform improvements, hardening efforts, and security integrations in Firefox.  
* **Firefox Privacy:** New privacy features, enhanced protections, and privacy integrations in Firefox.  
* **Community Engagement:** Highlights from our security research community and bug bounty program.

## Preface

Note: Some of the bugs linked below might not be accessible to the general public and restricted to specific work groups. [We de-restrict fixed security bugs after a grace-period](https://firefox-source-docs.mozilla.org/bug-mgmt/processes/fixing-security-bugs.html#keeping-private-information-private), until the majority of our user population have received Firefox updates. If a link does not work for you, please accept this as a precaution for the safety of all Firefox users.

## Firefox Security

**Rapid Releases, Now Even Faster:** Since September 2026, Firefox has followed a faster release cycle, bringing improvements, including security fixes, to users more quickly. We also invite you to [see and shape what’s next for Firefox](https://www.firefox.com/en-US/whatsnext/), including upcoming security and privacy features.

Many recent releases have included more [security fixes](https://www.mozilla.org/security/announce). We continue to use frontier LLMs to identify and fix security bugs in Firefox to always keep you safe online. We have also changed how we publish advisories: instead of grouping internally identified memory safety vulnerabilities under a single CVE, we now publish a separate advisory for each bug, anticipating a [potential change to the CVE Numbering Rules](https://github.com/CVEProject/cve-documents/issues/47#issuecomment-5587294133). CVE counts alone are not a reliable measure of security, as software vendors are not required to assign CVEs to vulnerabilities they discover internally.

**Know when your TLS is broken:** Starting with Firefox 157, you will see [warnings in the site information panel](https://www.firefox.com/en-US/firefox/157.0/releasenotes/) (the shield icon in the address bar) and in Settings when your connection may not be completely private. For example, if an administrator configures your Firefox to record its encryption keys, you will be able to catch that. Recording these keys could allow another software to read all of your encrypted web traffic without your knowledge.

**Mozilla Root Store Policy v3.1**: We are strengthening transparency and assurance in the WebPKI with [Root Store Policy v3.1](https://blog.mozilla.org/security/2026/06/29/improving-transparency-and-assurance-in-the-web-pki-mozilla-root-store-policy-v3-1/), which introduces stricter transparency, documentation, and audit requirements for public certificate authorities.

**Email Client Spam Prevention:** Firefox now [treats mailto: links as popups](https://bugzilla.mozilla.org/show_bug.cgi?id=299116), requiring explicit user action before opening the email client. This prevents websites from repeatedly opening email composition windows through JavaScript and overwhelming the user.

**Restricting Local File Access for Extensions:**  Starting in Firefox 153, [extensions need your explicit permission to access local files](https://blog.mozilla.org/addons/2026/07/23/firefox-153-webextensions-api-updates/) through \`file://\` URLs. Access is off by default for all extensions, including those already installed, so you decide which extensions can access files on your device.

## Firefox Privacy

**Fingerprinting Protections:** Following a [report on Alibaba’s use of audio fingerprinting](https://blog.laserphile.com/2026/08/aliexpress-webpage-keeping-multipoint.html), [Tom Ritter published a data-backed analysis](https://ritter.vg/blog-webaudio_alibaba.html) explaining how Firefox’s protections reduce the accuracy of fingerprinting through the Web Audio API. The story received coverage from [Ars Technica](https://arstechnica.com/security/2026/08/aliexpress-caught-fingerprinting-visitors-after-sending-inaudible-sounds-to-browsers/), [The Register](https://www.theregister.com/security/2026/08/24/aliexpress-accused-of-fingerprinting-shoppers-with-silent-audio-trick-that-also-muted-a-devs-headphones/5291662), and others.

**Preventing Link Tracking:** Right-clicking a link and selecting [Copy Clean Link](https://support.mozilla.org/en-US/kb/refresh-firefox-reset-add-ons-and-settings) will now strip tracking parameters across arbitrary subdomains, along with 130 additional tracking parameters.

**Bringing Containers Directly into Firefox:** Previously available as extension, [Firefox 153 introduces built-in Containers](https://blog.mozilla.org/en/firefox/firefox-containers-preview/), helping you keep work, personal browsing, and other activities separate. Each container maintains its own cookies and login sessions, so you could use containers for multiple accounts on the same site, or to limit online tracking by isolating certain websites in their respective containers. Firefox 155 also adds the ability to reorder containers.

**Tracker Blocking Information on Android:** Starting in Firefox 153, the Privacy Dashboard on Android shows which types of trackers Firefox blocks, giving you a clearer view of how Firefox protects your privacy as you browse.

**Built-in Ad Blocking to iOS**: Introduced on September 1, [Firefox’s built-in Ad Blocker on iOS](https://blog.mozilla.org/en/firefox/ad-blocker-on-ios/) blocks many third-party ads and ad-related trackers, reducing clutter and distractions. It works alongside Enhanced Tracking Protection and can be enabled in the ‘Browsing’ settings in your Firefox on iOS.

## Community Engagement

**Advancing Web Security at Dagstuhl:** Firefox security experts joined researchers and other browser vendors at the [Dagstuhl Seminar on Web Application Security](https://www.dagstuhl.de/de/seminars/seminar-calendar/seminar-details/26291) in July, presenting on LLM-assisted vulnerability detection, XSS protections, and web-scale security measurements.

**Mozilla Security Research Summit:** Mozilla’s Security-Engineering University Relationship Framework (SURF) is back. On November 20, 2026, we will host another invitation-only summit at TU Delft in the Netherlands, bringing together Mozilla engineers and academic researchers to explore security and privacy challenges and spark new collaborations. The summit takes place immediately after the [ACM Conference on Computer and Communications Security (CCS)](https://www.sigsac.org/ccs/CCS2026/).

**TLS, QUIC, HTTP/3, and Happy Eyeballs:** Max Inden gave a guest lecture at TU Dresden on [modern networking protocols](https://mxinden-bot.github.io/slides/01-tu-dresden/), covering the technologies behind fast, secure web connections.

## Looking Ahead

*Firefox users receive the security and privacy improvements highlighted in this newsletter through regular updates. If you haven’t tried Firefox yet, download it to experience a fast, secure browser and support Mozilla’s mission to keep the web open, safe, healthy, and accessible to everyone.*

*Thank you to everyone who helps make Firefox and the web more secure and privacy-focused, by reporting bugs, conducting research, contributing code, or sharing feedback.*

*We look forward to sharing more updates in the Q4 2026 edition.*

*— The Firefox Security & Privacy Teams*

