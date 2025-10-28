---
layout: post
title:  "Firefox Security & Privacy Newsletter 2025 Q3"
date:   2025-10-28 14:06:37 +0100
author: Frederik Braun, Christoph Kerschbaumer
---

Welcome to the Q3 2025 edition of the Firefox Security and Privacy newsletter\!

Security and Privacy on the web are the cornerstones of [Mozilla’s manifesto](https://www.mozilla.org/en-US/about/manifesto/), and they influence how we operate and build our products. Following are the highlights of our work from Q3 2025, grouped into the following categories:

* **Firefox Product Security & Privacy**, showcasing new Security & Privacy Features and Integrations in Firefox.  
* **Firefox for Enterprise**, highlighting security & privacy updates for administrative features, like Enterprise policies.  
* **Core Security**, outlining Security and Hardening efforts within the Firefox Platform.  
* **Web Security and Standards**, allowing websites to better protect themselves against online threats.


## Preface

Note: Some of the bugs linked below might not be accessible to the general public and restricted to specific work groups. [We de-restrict fixed security bugs after a grace-period](https://firefox-source-docs.mozilla.org/bug-mgmt/processes/fixing-security-bugs.html#keeping-private-information-private), until the majority of our user population have received Firefox updates. If a link does not work for you, please accept this as a precaution for the safety of all Firefox users.

## Firefox Product Security & Privacy

* As a follow-up to our [last newsletter](https://attackanddefense.dev/2025/07/17/firefox-security-privacy-newsletter-2025-q2.html), **Firefox has won a “Speedrunner” Award** by the TrendMicro Zero Day Initiative for being consistently fast to patch security vulnerabilities. This is the second consecutive year, in which Firefox is recognized for the speedy delivery of security updates.  
* **Protecting against Fingerprinting-based tracking:** With Firefox 143, we’ve introduced new defenses against online fingerprinting. Our analysis of the most frequently exploited user data shows that it’s possible to significantly lower the success rate of fingerprinting attacks, without compromising a user’s browsing experience. Specifically, Firefox now standardizes how it reports device attributes such as CPU core count, screen size, and touch input capabilities. By unifying these values across our entire user base, we cut the share of Firefox users who appear unique to fingerprinting scripts from roughly 35% to just 20%.  
* **Strict Tracking Protection with web compatibility in mind:** When users set Firefox’s tracking protection to strict, we already warn them that stricter blocking may result in missing content or broken websites. As of Firefox 142, we are providing a list of exceptions that may [help unbreak popular websites](https://support.mozilla.org/en-US/kb/manage-enhanced-tracking-protection-exceptions) without compromising the protection. The list of exceptions is transparently shared on [https://etp-exceptions.mozilla.org/](https://etp-exceptions.mozilla.org/).  
* **DoH on Android**: We have landed opt-in support for DoH Android in Firefox 143\. Opt-in available in Firefox preferences UI, Firefox Android users can enable DoH with [Increased or Max Protection settings](https://support.mozilla.org/en-US/kb/configure-dns-over-https-protection-levels-firefox-android#w_protection-levels-explained) to prevent network observers from tracking their browsing behaviour.  
* **Improved TLS Error Pages:** We improved non-overridable TLS error pages to provide more context for end users. Starting in Fx140, Firefox contains more information on why a connection was blocked, highlighting that Firefox is not causing the problem but rather that the website has a security problem and Firefox is actually keeping the user safe.  
* **SafeBrowsing v5**: Firefox Nightly now supports the [SafeBrowsing v5 protocol](https://developers.google.com/safe-browsing/reference), which protects against threats like phishing or malware sites, in preparation for the upcoming decommissioning of SafeBrowsing v4 server.  
* **Private Downloads in Private Browsing:** When downloading a file in Private Browsing mode, Firefox 143 now [asks](https://bugzilla.mozilla.org/show_bug.cgi?id=1981504) whether to keep or delete the files after that session ends. You can adjust this behavior in Settings, if desired.  
* **Improved Video sharing:** As of Firefox 143, the browser permission dialog will now show a preview of the selected Video camera, making it much easier to see and decide what is being shared before providing camera permissions to a page.


## Firefox for Enterprise

* **Updated Enterprise Policy for Tracking Protection:** The [EnableTrackingProtection](https://mozilla.github.io/policy-templates/#enabletrackingprotection) policy has been updated to allow you to set the category to either `strict` or `standard`. When the category is set using this policy, the user cannot change it. The [EnableTrackingProtection](https://mozilla.github.io/policy-templates/#enabletrackingprotection) policy has also been updated to allow you to set control Suspected fingerprinters. For more information, see [this SUMO page](https://support.mozilla.org/kb/firefox-protection-against-fingerprinting#w_suspected-fingerprinters).  
* **Improved Control over SVG, MathML, WebGL, CSP reporting and Fingerprinting Protection:** The [Preferences](https://mozilla.github.io/policy-templates/#preferences) policy has been updated to allow setting the preferences `mathml.disabled`, `svg.context-properties.content.enabled`, `svg.disabled`, `webgl.disabled`, `webgl.force-enabled`, `xpinstall.enabled`, and `security.csp.reporting.enabled` as well as prefs beginning with `privacy.baselineFingerprintingProtection` or `privacy.fingerprintingProtection.`

## Core Security

* **CRLite on Desktop and Mobile**: CRLite is a faster, more reliable and privacy-protecting certificate revocation check mechanism, as compared to the traditional OCSP (Online Certificate Status Protocol). CRLite is available in Desktop versions since Firefox 142 and on Firefox for Android in Firefox 145\. Read details on CRLite in the blogpost: [CRLite: Fast, private, and comprehensive certificate revocation checking in Firefox](https://hacks.mozilla.org/2025/08/crlite-fast-private-and-comprehensive-certificate-revocation-checking-in-firefox/).  
* **Supporting Certificate Compression in QUIC**: Certificate compression reduces the size of certificate chains during a Transport Layer Security (TLS) handshake, which improves performance by lowering latency and bandwidth consumption. The three compression algorithms zlib, brotli, and zstd are available in QUIC starting with Firefox 143\.

## Web Security & Standards

* **Improved Cache removal:** When a website uses the `"cache"` directive of the [`Clear-Site-Data`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Clear-Site-Data) response header, Firefox 141 now also clears the backwards-forwards cache ([bfcache](https://developer.mozilla.org/en-US/docs/Glossary/bfcache)). This allows a site to ensure that private session details can be removed, even if a user uses the browser back button. ([bug 1930501](https://bugzil.la/1930501)).  
* **Easy URL Pattern Matching**: The [URL Pattern API](https://developer.mozilla.org/en-US/docs/Web/API/URL_Pattern_API) is fully supported as of Firefox 142, enabling you to match and parse URLs using a standardized pattern syntax. ([bug 1731418](https://bugzil.la/1731418)).

## Going Forward

As a Firefox user, you will automatically benefit from all the mentioned security and privacy benefits with the enabled auto-updates in Firefox. If you aren’t a Firefox user yet, you can [download Firefox](https://www.mozilla.org/firefox/new/?_gl=1*3c2zyd*_ga*MTkzMzM4MjE2NC4xNjc0NzM5NDMy*_ga_X4N05QV93S*MTc0NTg0NzU4Ny4xODIuMS4xNzQ1ODQ3NjM5LjAuMC4w) to experience a fast and safe browsing experience while supporting Mozilla’s mission of a healthy, safe and accessible web for everyone.

Thanks to everyone who helps make Firefox and the open web more secure and privacy-respecting.

See you next time with the Q4 2025 Report\!  
\- Firefox Security and Privacy Teams.

