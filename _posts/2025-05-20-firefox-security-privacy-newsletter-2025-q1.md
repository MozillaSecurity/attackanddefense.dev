# Firefox Security and Privacy Newsletter 2025 Q1

Security and privacy on the web are cornerstones of [Mozilla’s manifesto](https://www.mozilla.org/en-US/about/manifesto/), and they influence how we operate and build our products. Following are the highlights of our work from Q1 2025, grouped into the following categories:

* **Firefox Product Security & Privacy**, showcasing new Security & Privacy Features and Integrations in Firefox.  
* **Core Security**, outlining Security and Hardening efforts within the Firefox Platform.  
* **Web Security**, allowing websites to better protect themselves against online threats.

## Preface

Note: Some of the bugs linked below might not be accessible to the general public and are still restricted to specific work groups. [We de-restrict fixed security bugs after a grace-period](https://firefox-source-docs.mozilla.org/bug-mgmt/processes/fixing-security-bugs.html#keeping-private-information-private), until the majority of our user population have received their updates. If a link does not work for you, please accept this as a precaution for the safety of all of our users.

## Firefox Product Security & Privacy

**HTTPS by Default**: After shipping mixed content upgrades last year ([Bug 1779757](https://bugzilla.mozilla.org/show_bug.cgi?id=1779757)), starting with Firefox  136, we now upgrade all navigations to HTTPS ([Bug 1921221](https://bugzilla.mozilla.org/show_bug.cgi?id=1921221)) and provides a fallback to HTTP if a secure and encrypted connection cannot be established. Our telemetry suggests that **HTTPS requests for Firefox users have increased by at least 1.5%**. See [The Evolution of HTTPS Adoption in Firefox](https://attackanddefense.dev/2025/03/31/https-first-in-firefox-136.html) for more details. In the same vein, our research paper [The State of Adoption on the Web](https://research.mozilla.org/files/2025/03/the_state_of_https_adoption_on_the_web.pdf) (presented at [MADWeb 2025](https://madweb.work/)) captures HTTPS adoption around the world as well as the effectiveness of the different upgrading mechanisms.

**Introducing CRLite**: CRLite, a faster, more reliable and privacy-preserving certificate revocation check mechanism as compared to the traditional OCSP (Online Certificate Status Protocol), has been rolled out to release ([Bug 1429800](https://bugzilla.mozilla.org/show_bug.cgi?id=1429800)) in Firefox 135\. A [corresponding talk on CRLite](https://www.youtube.com/watch?v=gnB76DQI1GE&t=19517s) was presented at [Real World Crypto 2025](https://rwc.iacr.org/2025/). Also, [a research paper about CRLite](https://research.mozilla.org/files/2025/04/clubcards_for_the_webpki.pdf) was accepted at [IEEE Symposium on Security and Privacy](https://sp2025.ieee-security.org/).

**Certificate Transparency:** Firefox is shipping Certificate Transparency (CT) in Firefox 136 ([Bug 1927085](https://bugzilla.mozilla.org/show_bug.cgi?id=1927085)). CT enforces that Firefox will only accept certificates that have been publicly logged, and can be checked for malice or mistake. This change brings Firefox into alignment with other browsers and we are now listed under “User agents using CT” on [https://certificate.transparency.dev/useragents/](https://certificate.transparency.dev/useragents/). CT and CRLite have propelled Firefox to the forefront of TLS security amongst browsers.

**TLS Client Auth**: Firefox on Android now supports TLS Client Auth ([Bug 1813930](https://bugzilla.mozilla.org/show_bug.cgi?id=1813930)) starting with Firefox 138\. TLS Client Certificates are typically used in enterprise and governmental contexts to authenticate users and offer many desirable security and phishing-resistant properties.

**Root Store Policy updates:** Mozilla maintains a policy for inclusions and considerations to the root store as it applies to Certificate authorities operations and the certificates they issue. The new and updated version of the [Mozilla Root Store Policy (MRSP) v3.0](https://blog.mozilla.org/security/2025/03/12/enhancing-ca-practices-key-updates-in-mozilla-root-store-policy-v3-0/) went into effect on March 15, 2025\.

**Safebrowsing updates**: Starting with Firefox 138, Safebrowsing now classifies only top-level loads ([Bug 1953088](https://bugzilla.mozilla.org/show_bug.cgi?id=1953088)) and no longer blocks subresource loads, like external JS scripts, or also images, which reduces the risk of web-compat issues caused by unsupported subresource blocking.

**Unbreaking websites from Anti-Tracking mechanisms**: We have worked with [disconnect.me](http://disconnect.me) to unblock certain consent management providers (CMP) in private browsing mode and ETP-Strict (see non-exhaustive [bug list](https://bugzilla.mozilla.org/buglist.cgi?bug_id=1909809%2C1906427%2C1909418%2C1942290%2C1951065%2C1924998%2C1936252%2C1934494&list_id=17532862)). CMPs \- often seen as “Cookie Banners” \- are responsible for almost half of all site breakage when they are blocked by our anti tracking protection. This work allowed many websites to work as expected again in private browsing mode.

**Smartblock Embeds gives users control over blocked content**: Starting with Firefox 136, SmartBlock Embeds support empowers ETP-Strict and PBM users to consciously opt into highly visible content, and prevents the perception that the page is ‘broken’ \- even though our protections are working as intended. Smartblock works for embedded content from Instagram ([Bug 1892173](https://bugzilla.mozilla.org/show_bug.cgi?id=1892173)), TikTok ([Bug 1892172](https://bugzilla.mozilla.org/show_bug.cgi?id=1892172)) and Twitter/X ([Bug 1901602](https://bugzilla.mozilla.org/show_bug.cgi?id=1901602)), which are blocked by default in Private windows or ETP-Strict mode. Embedded blocked content from these domains will show a 1-button click option for users to un-block all content from that domain on that specific website.

**Away from DNT and towards GPC**: In Q1, Firefox became the first major browser to remove the “Do Not Track” checkbox from the privacy preferences/settings, recognising that Global Privacy Control (GPC) is a better option for 2025\. See [SUMO article](https://support.mozilla.org/en-US/kb/how-do-i-turn-do-not-track-feature) for details.

## Core Security

**Hardening Firefox:** Starting in Firefox 135, Firefox UI uses a strict Content-Security-Policy (CSP) in [browser.xhtml](https://searchfox.org/mozilla-central/source/browser/base/content/browser.xhtml) to prevent XSS-like sandbox escapes ([Bug 1935985](https://bugzilla.mozilla.org/show_bug.cgi?id=1935985)). This involved modifying over 600 inline event handlers and a careful rollout with notice to people who run modified Firefox configurations. A detailed description of these security hardening efforts are available in our blogpost: [Hardening the Firefox Frontend with Content Security Policies](https://attackanddefense.dev/2025/04/09/hardening-the-firefox-frontend-with-content-security-policies.html).

**Sandboxing:** The Windows sandbox for Content Processes has been [moved to USER\_RESTRICTED](https://bugzilla.mozilla.org/show_bug.cgi?id=1403931) and rolled out in Firefox 134 and is enabled by default in Firefox 135\. This change removes read access from most of the filesystem and registry for the content process. Before, read access was generally allowed outside of the User’s home directory, which has been blocked since Firefox 56\. In Firefox 138, by [further differentiating the Gecko Media Plugin process sandbox policy](https://bugzilla.mozilla.org/show_bug.cgi?id=1952926), we were also able to enable win32k lockdown and Arbitrary Code Guard (blocking dynamic code) for the openh264 GMP process.

**CSS Fuzzing:** We are also attributing a major piece of our platform goals to [Interop 2025](https://wpt.fyi/interop-2025): Enhancements in CSS syntax generation have shown immediate results and will continue to support the work throughout the year.

**XSLT Fuzzing:** Early in Q1, we have incorporated an experimental XSLT fuzzer by Ivan Fratric of Google Project Zero into our infrastructure, where it is now running continuously. Thank you, Ivan\!

**NSS Fuzzing:** Our fuzzing focus on NSS (Network Security Services) has also shown a significant uptick in NSS fuzzing coverage across both oss-fuzz as well as our internal tools. We also added a TSan target for NSS, ensuring stricter thread-safety going forward.

**Site-scout**: Site-scout is a tool that visits web pages with instrumented builds (like Address Sanitizer). We are continuing to enhance site-scout, such that it interacts with websites and ideally teases out more bugs.

## Community Engagement

**Bug Bounty:** The [Firefox client bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/) rewards contributors for finding security bugs in Firefox. Our award categories ranges from $500 to $20,000 depending on bug severity and report quality. With 31 awarded bounties this quarter, it is one of our main sources of security bug reports. The majority of the reported bugs are *low* or *moderate* severity UI spoofs. We are changing the reward structure for these kinds of bugs going forward \- please find details on our [Firefox bug bounty pages](https://www.mozilla.org/en-US/security/client-bug-bounty/)**.** We are also opening the opportunity for **guest blog posts** on attackanddefense.dev. [Please reach out if you want to write about a previous finding of yours\!](https://attackanddefense.dev/about/#guest-blog-posts)

**Mozilla Research Summit:** The [Mozilla Research Summit](https://surf.mozilla.org/events/2025/sandiego/) took place on March 1, 2025 in San Diego California. We had 56 participants from 3 continents and 25+ different research institutions. The day included some high-caliber talks from Mozillians as well as security and privacy academics and sparked new research collaborations between Mozilla and the academic community.

**Co-organized Open-Source Crypto Summit**: We co-organized the [Open Source Cryptography Workshop (OSCW)](https://opensourcecryptowork.shop/) which brings together practitioners interested in all sorts of open source crypto projects ranging from production and maintenance challenges to best practices for designing clean-sheet crypto systems with modern primitives.

## Web Security & Standards

**Fingerprinting:** We have taken on co-authorship of the [W3C’s guidance on Fingerprinting](https://w3c.github.io/fingerprinting-guidance/), to ensure that it matches our understanding of the evolving threats and the meaningful defenses we envision.  We have merged some changes already with more under discussion.

**Messaging Layer Security**: Firefox is now the first browser to have a prototype implementation of MLS (Messaging Layer Security) ([Bug 1876002](https://bugzilla.mozilla.org/show_bug.cgi?id=1876002)). MLS enables end-to-end security for a wide range of use cases such as secure group messaging. MLS is available in Firefox 136 as an [origin trial](https://wiki.mozilla.org/Origin_Trials) (set the preference dom.origin-trials.mls.state to 1) which exposes an experimental Web API that will be used by our partners.

**Sanitizer API:** We have [updated our implementation of the Sanitizer API](https://bugzilla.mozilla.org/show_bug.cgi?id=1956310), which allows mitigating the risk of DOM-based cross-site scripting (XSS) attacks. This update aligns our implementation of the sanitizer API with recent developments and prepares it for a pre-release.

**Integrity:** We presented our plans for wider Web App Integrity at the W3C WebAppSec working group ([explainer](https://github.com/beurdouche/explainers/blob/main/waict-explainer.md)) and started to work on [a first milestone towards script-integrity enforcements](https://github.com/w3c/webappsec-subresource-integrity/pull/133) as part of the Subresource Integrity specification with Yoav Weis from Shopify. We also [implemented integrity for import maps.](https://bugzilla.mozilla.org/show_bug.cgi?id=1945540)

## Going Forward

As a Firefox user, you will automatically benefit from all the mentioned security and privacy benefits with the enabled auto-updates in Firefox. If you are not a Firefox user yet, you can simply [download Firefox](https://www.mozilla.org/firefox/new/?_gl=1*3c2zyd*_ga*MTkzMzM4MjE2NC4xNjc0NzM5NDMy*_ga_X4N05QV93S*MTc0NTg0NzU4Ny4xODIuMS4xNzQ1ODQ3NjM5LjAuMC4w) and start benefiting from all the ways that Firefox works to protect you when browsing the internet.

Thanks to everyone who helps make Firefox and the open web more secure and privacy-respecting.

See you next time with the Q2 2025 Report\!  
\- Firefox Security and Privacy Teams
