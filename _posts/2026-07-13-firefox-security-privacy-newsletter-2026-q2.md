---
layout: post
title:  "Firefox Security & Privacy Newsletter 2026 Q2"
date:   2026-07-13 00:00:00 +0100
author: Frederik Braun, Christoph Kerschbaumer
---

Welcome to the Q2 2026 edition of the Firefox Security & Privacy Newsletter.

Security and privacy are core principles of [Mozilla’s Manifesto](https://www.mozilla.org/en-US/about/manifesto/) and remain at the heart of Firefox’s development. In this edition, we highlight some of the key security and privacy initiatives from Q2 2026, grouped into the following areas:

* **Firefox Product Security & Privacy**, new security and privacy features, protections, and integrations in Firefox  
* **Core Security**, platform security improvements, hardening efforts, and foundational enhancements  
* **Community Engagement**, highlights from our security research community and bug bounty program  
* **Web Security & Standards**, progress on web technologies and standards that help websites better protect users from online threats

## Preface

Note: Some of the bugs linked below might not be accessible to the general public and restricted to specific work groups. [We de-restrict fixed security bugs after a grace-period](https://firefox-source-docs.mozilla.org/bug-mgmt/processes/fixing-security-bugs.html#keeping-private-information-private), until the majority of our user population have received Firefox updates. If a link does not work for you, please accept this as a precaution for the safety of all Firefox users.

## Firefox Product Security & Privacy

**Private Access Control Tokens (PACT):** PACT is a cross-industry initiative designed to tackle one of the web's most urgent challenges: enabling websites to reliably distinguish legitimate users and authorized automated agents from abusive traffic without compromising user privacy. To introduce the initiative, we published a [technical deep dive on Mozilla Hacks](https://hacks.mozilla.org/2026/06/pact-anonymous-credentials-for-the-web/) alongside a [companion Mozilla blog post](https://blog.mozilla.org/en/privacy-security/keeping-the-web-open-and-private-in-the-bot-era/) that explains the vision, motivation, and privacy-preserving design behind PACT.

**Qualified Website Authentication Certificates (QWACs):** Firefox is prepared to meet upcoming eIDAS requirements under the [EU Digital Identity Framework.](https://eidas.ec.europa.eu/efda/home) [Qualified Website Authentication Certificates (QWACs), as required by the framework, are supported](https://eidas.ec.europa.eu/efda/discover/qwac) in Firefox 153 ([Bug 2043399](https://bugzilla.mozilla.org/show_bug.cgi?id=2043399)) onwards.

**Hardening Firefox with Claude Mythos:** In a [blogpost](https://hacks.mozilla.org/2026/05/behind-the-scenes-hardening-firefox/) we shared how our AI-assisted security testing pipeline, powered by Claude Mythos, uncovered and helped remediate hundreds of previously hidden vulnerabilities in Firefox, significantly strengthening the browser's security while demonstrating the transformative potential of AI to enhance defensive cybersecurity.

**Visual Indications for Geolocation Access:** In light of some web pages using geolocation for activities that are not related to their maps functionality, Firefox now displays [a real-time visual indicator](https://bugzilla.mozilla.org/show_bug.cgi?id=2038194) whenever a web page is accessing the user’s geolocation. Starting with Firefox 153, the address bar now provides a [real-time visual indicator](https://bug2038194.bmoattachments.org/attachment.cgi?id=9586032) the moment a website begins accessing a user's location, providing users with  immediate awareness and greater transparency into when and how their geolocation data is being used.

**Improving Website Compatibility in Private Browsing:** Starting with Firefox 152, Private Browsing Mode now offers users the option to [temporarily lower tracking protections](https://bugzilla.mozilla.org/show_bug.cgi?id=1994405) for the current tab when stricter tracker blocking could be causing a website to malfunction.  Previously, this may have resulted in users turning off privacy protections completely to continue using visited web page. With our new feature, users can quickly restore site functionality of the current tab, preserving users' overall privacy settings.

**Instant fresh start through new [Fire Button](https://support.mozilla.org/en-US/kb/private-browsing-use-firefox-without-history):** Firefox 151 introduced the new Fire Button for Private Browsing, giving users an instant fresh start with a single click. Instead of closing and reopening a Private Window, users can [immediately clear all browsing data and continue browsing in a clean session](https://bugzilla.mozilla.org/show_bug.cgi?id=1846495), making Private Browsing faster, more convenient, and just as private.

**Advanced Anti-Fingerprinting Protections:** Firefox 151 expands our default anti-fingerprinting defenses by ensuring the Available Screen Resolution, Touch Points, and Canvas APIs will provide uniform results for all of our users while also maintaining performance and compatibility. On macOS, for example, these enhancements are expected to reduce the share of users identified as unique by more than 20%, making it significantly harder for websites to uniquely identify and track users using obscure fingerprinting.

**Local Network Access Protections:** Firefox now requires user permission before websites can access apps and services on a user's local network or device, helping prevent unauthorized access and sneaky tracking attempts. The [LNA](https://support.mozilla.org/en-US/kb/control-personal-device-local-network-permissions-firefox) feature is rolling out gradually, starting with Firefox Desktop 151 through 153\. Android support will follow in upcoming releases. 

## Core Security

**Firefox CA Root Program:** We published [Root Store Policy v3.1](https://blog.mozilla.org/security/2026/06/29/improving-transparency-and-assurance-in-the-web-pki-mozilla-root-store-policy-v3-1/), introducing stricter transparency, documentation, and audit requirements for public CAs to strengthen trust in the Web PKI.

[**WebAuthn Related Origin Requests**](https://bugzilla.mozilla.org/show_bug.cgi?id=2010193)**:** This feature allows seamless passkey sign-ins across related domains e.g., the same provider using multiple top-level domains. In contrast to other browsers, Firefox UI provides transparency and choice so users are aware and can control when websites request for passkeys from other, related sites.

## Community Engagement

**Hosting Events:** We organized and hosted multiple [web tech meet-ups in the Mozilla Berlin office](https://www.meetup.com/de-DE/berlin-mozilla-meetup/), bringing together the developer community to explore the latest advances in web technology, privacy, and security. If you're in the area, we'd love to have you join us at a future event.

**Community Shares:**  Firefox tracking protection was presented at the [SnooSec conference held in the Reddit NYC office](https://www.reddit.com/r/SnooSec/comments/1te55fx/thanks_for_joining_us_at_snoosec_nyc/). We also had a presentation about existing and upcoming protections against web tracking at the [Chemnitz Linux Days](https://chemnitzer.linux-tage.de/2026/en) conference, and a talk about the latest browser-based XSS protections at [OWASP AppSec ‘26](https://owasp.glueup.com/event/owasp-global-appsec-eu-2026-vienna-austria-162243/) in Vienna.

## Web Security & Standards

**Web Application Integrity, Consistency and Transparency (WAICT):** We are working on WAICT, a new proposal to bring stronger integrity and transparency guarantees to web applications, helping make the web a more trustworthy platform for security-sensitive applications such as end-to-end encrypted messaging. We shared our technical vision in a [Mozilla Hacks blog post](https://hacks.mozilla.org/2026/05/trustworthy-javascript-for-the-open-web/), including a prototype implementation in Firefox Nightly that works with our [WAICT Demo](https://demo.waict.dev/) and a [draft specification](https://github.com/waict-wg).

**Sanitizer API:** We are advancing the Sanitizer API to make robust protection against cross-site scripting (XSS) vulnerabilities more accessible. By exploring an [implicit sanitizer policy](https://github.com/mozilla/explainers/blob/main/trusted-or-sanitized-html.md) that integrates with Trusted Types, we aim to prevent an entire class of XSS attacks with no application code changes, making secure-by-default web applications easier to build and deploy.

## Looking Ahead

Firefox users will receive these security and privacy improvements automatically. If you’re not already a user, [we recommend you give it a try](https://firefox.com/). Firefox helps you shape a more personal internet that puts you back in control \- all while supporting the non-profit Mozilla in its mission to keep the web open, safe, and accessible for everyone.

Thank you to everyone who contributes to making Firefox and the web more secure and privacy-focused. You can have an impact too, just by [reporting bugs](https://bugzilla.mozilla.org/enter_bug.cgi), conducting research, contributing code, or providing feedback.

We look forward to sharing more updates in the Q3 2026 edition.

*— The Firefox Security & Privacy Teams*
