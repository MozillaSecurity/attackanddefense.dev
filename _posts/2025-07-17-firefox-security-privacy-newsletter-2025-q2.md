---
layout: post
title:  "Firefox Security & Privacy newsletter 2025 Q2"
date:   2025-07-17 10:00:00 +0100
author: Frederik Braun, Christoph Kerschbaumer
---

Welcome to the Q2 2025 edition of the Firefox Security and Privacy newsletter\!

Security and Privacy on the web are the cornerstones of [Mozilla’s manifesto](https://www.mozilla.org/en-US/about/manifesto/), and they influence how we operate and build our products. Following are the highlights of our work from Q2 2025, grouped into the following categories:

* **Firefox Product Security & Privacy**, showcasing new Security & Privacy Features and Integrations in Firefox.  
* **Core Security**, outlining Security and Hardening efforts within the Firefox Platform.  
* **Community Engagement,** news from our security and bug bounty community.  
* **Web Security and Standards**, allowing websites to better protect themselves against online threats.


## Preface

Note: Some of the bugs linked below might not be accessible to the general public and restricted to specific work groups. [We de-restrict fixed security bugs after a grace-period](https://firefox-source-docs.mozilla.org/bug-mgmt/processes/fixing-security-bugs.html#keeping-private-information-private), until the majority of our user population have received Firefox updates. If a link does not work for you, please accept this as a precaution for the safety of all Firefox users.

## Firefox Product Security & Privacy

* **CHIPS support:** Starting with Firefox 141, Firefox now supports [Cookies Having Independent Partitioned State (CHIPS)](https://developer.mozilla.org/en-US/docs/Web/Privacy/Guides/Privacy_sandbox/Partitioned_cookies), which introduces a new cookie attribute that allows website developers to access partitioned cookies. CHIPS provides the same privacy guarantees as Firefox’s built-in [Total Cookie Protection (TCP)](https://blog.mozilla.org/security/2021/02/23/total-cookie-protection/) which we have been shipping by default since 2022\.  
* **Expanding Total Cookie Protection for Known Trackers:** Beginning with Firefox 141, Firefox has switched to Total Cookie Protection for known trackers. This update replaces the previous method of blocking third-party tracking cookies, offering improved web compatibility while maintaining the same level of cookie privacy.  
* **Improving Web compatibility in Firefox:** We have improved the private browsing experience in Firefox based on a recent increase in user reports for [broken websites](https://support.mozilla.org/en-US/kb/report-breakage-due-blocking?redirectslug=report-breakage-due-blocking-redirect-1&redirectlocale=en-US#w_how-can-i-access-the-report-broken-site-tool) on [webcompat.com](http://webcompat.com). This improvement alone allowed our users to browse privately in more than 450 additional cases since the beginning of the year, representing a major share of all broken websites reports.  
* **Improving non-overrideable Error Pages**: When establishing an HTTPS connection, a browser can encounter a variety of different TLS errors. Firefox used to display different certificate error pages based on the error type (e.g., permanent and temporary). We unified these error pages \- starting with Firefox 140 we are now providing additional information, especially when an error is permanent and can not be skipped.  
* **Firefox Relay Integration:** Firefox Relay⁩ allows to create new temporary email address that forwards all emails to your true inbox. This allows you to protect your actual identity from hackers and spammers. Firefox Relay is available for all versions after Firefox 135, and we rolled it out to all desktop users on June 16\.

## Core Security

* **Firefox Response to Pwn2Own**: The [Pwn2Own](https://www.zerodayinitiative.com/blog/2025/5/14/pwn2own-berlin-the-full-schedule) hacking competition targets popular software organized by the TrendMicro Zero Day Initiative (ZDI). This year, Firefox was targeted twice but no team managed to break out of the Firefox sandbox. We immediately responded to the two contained exploits with a new Firefox release that included both security fixes in just 11 hours since the second exploit demonstration. This beats Firefox’s previous record of [21 hours last year](https://blog.mozilla.org/security/2024/04/04/rapidly-leveling-up-firefox-security/). Read more details in this blogpost: [Firefox Security Response to pwn2own 2025](https://blog.mozilla.org/security/2025/05/17/firefox-security-response-to-pwn2own-2025/). You can also read the [technical analysis](https://www.zerodayinitiative.com/blog/2025/7/14/cve-2025-4919-corruption-via-math-space-in-mozilla-firefox) of the bug and the exploit from the ZDI.  
* **Securing the open source ecosystem**: Firefox developers found a bug upstream in libvpx, a video codec library, which we identified and resolved. As a widely shared library, this fix protects media playback in Firefox as well as many other browsers, and rich media applications ([Bug 1962421](https://bugzilla.mozilla.org/show_bug.cgi?id=1962421), [Chrome Bug 419467315](https://crbug.com/419467315)).

## Community Engagement

* **Firefox Bug Bounty Hall of Fame**: We just updated the [Hall of Fame](https://www.mozilla.org/en-US/security/bug-bounty/hall-of-fame/), which credits all of the skillful security researchers that strive to keep Firefox secure. If you also want to contribute to Firefox security, please look at our [Bug Bounty pages](https://www.mozilla.org/en-US/security/bug-bounty/).  
* **Call for Guest Blog posts:** We are opening the opportunity for guest blog posts on [attackanddefense.dev](https://attackanddefense.dev/about/#guest-blog-posts) for all bug bounty participants. If you are reading this and want to write a blog post about your findings, [let us know](https://attackanddefense.dev/about/#guest-blog-posts)\!

## Web Security & Standards

* **Abridged Certificates in Firefox Nightly**: [Abridged Certs is an IETF draft](https://datatracker.ietf.org/doc/draft-ietf-tls-cert-abridge/) for a new compression scheme which enables a successful transition to Certificates using Post-Quantum Encryption Algorithms by enabling smaller and more performant Certificate Chains. The code is available since Firefox Nightly 141 and we are planning to do partner experiments in the upcoming months.   
* **Integrity Policy:** Recent work on the [Subresource Integrity (SRI)](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity) standard introduced a new web security policy, called [Integrity-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Integrity-Policy). This will allow websites to ensure that all of their scripts are protected with `integrity` data. Future work will expand this security mechanism to provide full web application integrity.  
* **HTTP [`Origin-Agent-Cluster`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Origin-Agent-Cluster)** **header**: Starting with Firefox 138, this new response header can be used by a site to hint that all documents from this origin are using the same operating system process. Besides these security isolation benefits, supporting this header makes it less likely that a resource-intensive document can degrade the performance on other origins ([Bug 1665474](https://bugzil.la/1665474)).  
* **HTTP [Clear-Site-Data: cache](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Clear-Site-Data#cache) header:** Starting with Firefox 141, websites can use the cache directive header to clear caches for the requested origin. This header allows websites to properly clear local data when users sign out of a website. ([Bug 1838506](https://bugzilla.mozilla.org/show_bug.cgi?id=1838506))  
* **WebRTC Security:** The [`getFingerprints()`](https://developer.mozilla.org/en-US/docs/Web/API/RTCCertificate/getFingerprints) method of the [`RTCCertificate`](https://developer.mozilla.org/en-US/docs/Web/API/RTCCertificate) interface is now available in Firefox 138\. An application can use this API to get fingerprints for a certificate, which might be shared out-of-band in order to identify a particular user or browser across WebRTC sessions ([Bug 1525241](https://bugzil.la/1525241)).  
* **Cookie Store API:** The [Cookie Store API](https://developer.mozilla.org/en-US/docs/Web/API/Cookie_Store_API) is now supported in Firefox 140 ([Bug 1958875](https://bugzil.la/1958875)). This API provides a modern, [asynchronous](https://developer.mozilla.org/en-US/docs/Glossary/Asynchronous) [`Promise`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)\-based method of managing cookies, which can be used in both the main thread as well as in [service workers](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API). Using the CookieStore API is less error-prone than relying on the old `document.cookie` property.

## Going Forward

As a Firefox user, you will automatically benefit from all the mentioned security and privacy benefits with the enabled auto-updates in Firefox. If you aren’t a Firefox user yet, you can [download Firefox](https://www.mozilla.org/firefox/new/?_gl=1*3c2zyd*_ga*MTkzMzM4MjE2NC4xNjc0NzM5NDMy*_ga_X4N05QV93S*MTc0NTg0NzU4Ny4xODIuMS4xNzQ1ODQ3NjM5LjAuMC4w) to experience a fast and safe browsing experience while supporting Mozilla’s mission of a healthy, safe and accessible web for everyone.

Thanks to everyone who helps make Firefox and the open web more secure and privacy-respecting.

See you next time with the Q3 2025 Report\!  
\- Firefox Security and Privacy Teams.

