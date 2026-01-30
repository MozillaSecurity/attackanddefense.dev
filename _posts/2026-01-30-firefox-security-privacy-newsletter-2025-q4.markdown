---
layout: post
title:  "Firefox Security & Privacy Newsletter 2025 Q4"
date:   2026-01-30 03:13:37 +0100
author: Frederik Braun, Christoph Kerschbaumer
---

Welcome to the Q4 2025 edition of the Firefox Security & Privacy Newsletter.

Security and privacy are foundational to [Mozilla’s manifesto](https://www.mozilla.org/en-US/about/manifesto/) and central to how we build Firefox. In this edition, we highlight key security and privacy work from **Q4 2025**, organized into the following areas:

* **Firefox Product Security & Privacy** — new security and privacy features and integrations in Firefox
* **Core Security** — platform-level security and hardening efforts
* **Community Engagement** — updates from our security research and bug bounty community
* **Web Security & Standards** — advancements that help websites better protect their users from online threats

## Preface

Note: Some of the bugs linked below might not be accessible to the general public and restricted to specific work groups. [We de-restrict fixed security bugs after a grace-period](https://firefox-source-docs.mozilla.org/bug-mgmt/processes/fixing-security-bugs.html#keeping-private-information-private), until the majority of our user population have received Firefox updates. If a link does not work for you, please accept this as a precaution for the safety of all Firefox users.

## Firefox Product Security & Privacy

**Functional Privacy.** Firefox empowers users with control and choice \- including the option for maximum privacy protections. Yet, our commitment lies in targeting online tracking by default in ways that ensures the web continues to function accurately and smoothly. With focus on this important balance, our protections have blocked more than 1 trillion tracking attempts, while reported site compatibility issues were driven down to an all time low: 500, as compared to 1,100 in Q1 of 2025\.

**Improved page redirect prevention:** Firefox now [blocks top-level redirects from iframes](https://support.mozilla.org/en-US/kb/pop-blocker-settings-exceptions-troubleshooting). This new prevention mechanism aligns Firefox behaviour with other browsers and protects users against so-called malvertising attacks.

**Improved protections against navigational cross-site tracking**: Navigational tracking is used to track users across different websites using browser navigations. [Bounce tracking](https://developer.mozilla.org/en-US/docs/Web/Privacy/Guides/Bounce_tracking_mitigations) is a type of navigational tracking that “bounces” user navigations through an intermediary tracking site. [Firefox’s Bounce Tracking Protection](https://developer.mozilla.org/en-US/docs/Web/Privacy/Guides/Bounce_tracking_mitigations) already protects against this tracking vector. And Firefox 145 uplevels this by also eliminating cache access for these intermediate redirect pages.

**Global Privacy Control (GPC):** Following Firefox's lead as the first major browser to do this, Thunderbird has also now replaced the legacy "Do Not Track" (DNT) in favor of [Global Privacy Control (GPC)](https://github.com/w3c/gpc/blob/main/explainer.md). This new control has the much needed legal footing to clearly communicate a user’s “do-not-sell-or-share preference” and [other browsers are expected to follow soon](https://groups.google.com/a/chromium.org/g/blink-dev/c/ztV1otnl_NQ/m/rvvLFkrdBQAJ).

**Warning prompts for digital identity requests**: When a webpage attempts to open a digital wallet app using custom URL schemes such as `openid4vp`, `mdoc`, `mdoc-openid4vp`, or `haip`, Firefox on Desktop and Android (Firefox 145 and newer)  now displays clear warning prompts that explain what’s happening and give users control.

## Core Security

**Certificate Transparency (CT) on Android:** Certificate Transparency enables rapid detection of unauthorized or fraudulent SSL/TLS certificates. CT has been available in Firefox Desktop since Firefox 136 and is now also available on Android starting with Firefox 145\.

**Post-Quantum Cryptography (ML-KEM):** ML-KEM is a next-generation public-key cryptosystem designed to resist attacks from large-scale quantum computers. Post-quantum (PQ) cryptography with ML-KEM support shipped in Firefox 132 for Desktop. Support is now also available on Android starting with Firefox 145 and in WebRTC starting with Firefox 146\.

## Community Engagement

**Mozilla and Firefox at the 39th [Chaos Communication Congress](https://events.ccc.de/tag/chaos-communication-congress/) (39C3)**: Teams from Firefox Security, Privacy, Networking, Thunderbird, and Public Policy collaborated to raise awareness of their work and gather direct community feedback. A clear highlight was the popularity of our swag, with our folks distributing 1,000 Fox Ears. The high level of engagement was further sustained by a dedicated community meetup and an impromptu AMA session, which drew attention from over 100 people.

**Firefox Bug Bounty Hall of Fame**: We just updated the [Hall of Fame](https://www.mozilla.org/en-US/security/bug-bounty/hall-of-fame/), which credits all of the skillful security researchers that strive to keep Firefox secure as of Q4 2025\. If you also want to contribute to Firefox security, please look at our [Bug Bounty pages](https://www.mozilla.org/en-US/security/bug-bounty/).

## Web Security & Standards

**Integrity-Policy:** Firefox 145 has added support for the [Integrity-Policy response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Integrity-Policy). The header allows websites to ensure that only scripts with an integrity attribute will load. Errors will be logged to the console, with support for the Reporting API coming in early 2026\.

**Compressed Elliptic Curve Points in WebCrypto:** Firefox 146 adds support for compressed elliptic curve points in WebCrypto. This reduces public key sizes by nearly half, saving bandwidth and storage, while still allowing the full point to be reconstructed mathematically. With this addition, Firefox now leads in [WebCrypto web platform test coverage.](https://wpt.fyi/results/WebCryptoAPI?label=experimental&label=master&aligned)

## Going Forward

As a Firefox user, you automatically benefit from the security and privacy improvements described above through Firefox’s regular automatic updates. If you’re not using Firefox yet, you can download it to enjoy a fast, secure browsing experience — while supporting Mozilla’s mission of a healthy, safe, and accessible web for everyone.

We’d like to thank everyone who helps make Firefox and the open web more secure and privacy-respecting.

See you next time with the **Q1 2026 report**.

— *The Firefox Security and Privacy Teams*
