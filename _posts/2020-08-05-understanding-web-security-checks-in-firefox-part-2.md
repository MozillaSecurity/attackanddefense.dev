---
layout: post
title: "Understanding Web Security Checks in Firefox (Part 2)"
author: Frederik Braun, Christoph Kerschbaumer
date: 2020-08-05
categories: 
  - "security-internals"
---

 

This is the second and final part of a blog post series that explains how Firefox implements Web Security fundamentals, like the [Same-Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) and [Content-Security-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP). While the [first post](https://blog.mozilla.org/attack-and-defense/2020/06/10/understanding-web-security-checks-in-firefox-part-1/) explained Firefox security terminology and theoretical foundations, this second post covers how to log internal security information to the console in a human readable format. Ultimately, we hope to inspire new security research in the area of web security checks and to empower participants in our bug bounty program to do better, deeper work.

Generally, we encourage everyone to do their security testing in [Firefox Nightly](https://www.mozilla.org/en-US/firefox/channel/desktop/#nightly). That being said, the logging mechanisms described in this post, work in all versions of Firefox – from [self-build](https://firefox-source-docs.mozilla.org/setup/index.html), to versions of [Nightly](https://www.mozilla.org/en-US/firefox/channel/desktop/#nightly), [Beta](https://www.mozilla.org/en-US/firefox/channel/desktop/), [Developer Edition](https://www.mozilla.org/en-US/firefox/developer/), [Release](https://www.mozilla.org/en-US/firefox/new/) and [ESR](https://www.mozilla.org/en-US/firefox/enterprise/) you may have installed locally already.

## Logging Security Internals to the console

All Firefox versions ship with a [logging framework](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Gecko_Logging) that allows one to inspect [Gecko's](https://developer.mozilla.org/en-US/docs/Mozilla/Gecko) (Firefox’ rendering engine) internals. In general, logging Firefox internals is as easy as defining an environment variable called `MOZ_LOG`. When testing on mobile, the document [Configuring GeckoView](https://firefox-source-docs.mozilla.org/mobile/android/geckoview/consumer/automation.html) describes and guides you through defining environment variables in GeckoView’s runtime environment. For printing web security checks we simply have to enable the logger named `["CSMLog"](https://searchfox.org/mozilla-central/source/dom/security/nsContentSecurityManager.cpp#49)`. You can also set `[MOZ_LOG_FILE](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Gecko_Logging#Enabling_Logging)` if you want to write into a log file.

Defining `MOZ_LOG="CSMLog:5"` as an environment variable will print all security checks to the console, where the digit 5 indicates the highest log-level ([Verbose](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Gecko_Logging)) which naturally prints a lot of debugging information but in turn allows debugging program flow.

## Understanding Security Load Information

Here is a log example when defining `CSMLog:5` and opening [https://example.com](http://example.com) in a new tab. For the sake of simplicity we removed line prefixes like `[Parent 361301: Main Thread]` which provide additional information around process type (parent/child), process id (pid) and the thread in which the logging occurred. These [prefixes can also be omitted](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Gecko_Logging#Enabling_Logging) by appending `",raw"` to the `MOZ_LOG` value.

|   ``` V/CSMLog doContentSecurityCheck: V/CSMLog   - channelURI: https://example.com/ V/CSMLog   - httpMethod: GET D/CSMLog   - loadingPrincipal: nullptr D/CSMLog   - triggeringPrincipal: SystemPrincipal D/CSMLog   - principalToInherit: NullPrincipal V/CSMLog   - redirectChain: V/CSMLog   - internalContentPolicyType: TYPE_DOCUMENT V/CSMLog   - externalContentPolicyType: TYPE_DOCUMENT V/CSMLog   - upgradeInsecureRequests: false V/CSMLog   - initialSecurityChecksDone: false V/CSMLog   - allowDeprecatedSystemRequests: false D/CSMLog   - CSP: V/CSMLog   - securityFlags: V/CSMLog     - SEC_ALLOW_CROSS_ORIGIN_SEC_CONTEXT_IS_NULL ```   |
| --- |

Listing 1: Firefox Security Log (`CSMLog:5`) when opening [https://example.com](http://example.com) in a new tab.

As of Firefox 80, every logged Web Security Check appears in the format from Listing 1. At the beginning of each line you see `CSMLog` which is the name of our logger and D/V are short for Debug/Verbose, which indicate the log level (4 is Debug, 5 is Verbose). The output format after this prefix is valid YAML and should be easy to parse automatically. These blocks of attributes are surrounded by **#DebugDoContentSecurityCheck Begin** and **End** for each and every outgoing request, which corresponds to the function name [doContentSecurityCheck](https://searchfox.org/mozilla-central/rev/82c04b9cad5b98bdf682bd477f2b1e3071b004ad/dom/security/nsContentSecurityManager.cpp#1017) in the SecurityManager. This function contains all available information of the current security context, before a request is sent over the wire. Naturally, this does not include the security checks that can only happen after the response is received (e.g., [X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options), [X-Content-Type-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options), etc.).

Most of the information below is readily available through the [nsILoadinfo](https://searchfox.org/mozilla-central/source/netwerk/base/nsILoadInfo.idl) object that corresponds to the request:

`**channelURI**` The URI about to be loaded and evaluated within this security check.

`**httpMethod**` (optional) Indicates the [HTTP request methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) like e.g., GET, HEAD or POST and hence is only available for HTTP requests.

`**loadingPrincipal**` In general, the [loadingPrincipa](https://searchfox.org/mozilla-central/rev/82c04b9cad5b98bdf682bd477f2b1e3071b004ad/netwerk/base/nsILoadInfo.idl#254)l reflects the security context ([the Principal](https://blog.mozilla.org/attack-and-defense/2020/06/10/understanding-web-security-checks-in-firefox-part-1/)) where the result of this resource load will be used. In case of an image load for example, this would be the security context (ContentPrincipal) of the loading document. Since this is a top-level load, the loadingPrincipal is null because the result of the load is a new document.

`**triggeringPrincipal**` The security context ([reflected through a Principal](https://blog.mozilla.org/attack-and-defense/2020/06/10/understanding-web-security-checks-in-firefox-part-1/)) which triggered this load. For almost all subresource loads, the [triggeringPrincipal](https://searchfox.org/mozilla-central/rev/82c04b9cad5b98bdf682bd477f2b1e3071b004ad/netwerk/base/nsILoadInfo.idl#304) and the [loadingPrincipal](https://searchfox.org/mozilla-central/rev/82c04b9cad5b98bdf682bd477f2b1e3071b004ad/netwerk/base/nsILoadInfo.idl#254) are identical. Since the user entered [https://example.com](http://example.com) in the URL-Bar, it is a user-triggered action which in Firefox is equivalent to a load triggered by the System and hence is using a SystemPrincipal as the triggeringPrincipal. For the sake of completeness, imagine a cross-origin CSS file which loads a background image. Then the triggeringPrincipal for that image load would be the security context (ContentPrincipal) of the CSS, but the loadingPrincipal would be the security context (ContentPrincipal) of the document where the image load will be used.

`**principalToInherit**` The [principalToInherit](https://searchfox.org/mozilla-central/rev/82c04b9cad5b98bdf682bd477f2b1e3071b004ad/netwerk/base/nsILoadInfo.idl#320) is only relevant to loads of type document (top-level loads) and type subdocument (e.g., iframe loads) and indicates the principal that will be inherited, e.g., when loading a data URI.

`**redirectChain**` In case a load encounters a redirect (e.g. a [302 server side redirect](https://developer.mozilla.org/en-US/docs/Web/HTTP/Redirections)) then the RedirectChain contains all Principals (serialized into origin URI strings) of all the redirects this load went through.

`**internalContentPolicyType/externalContentPolicyType**` Indicates the (internal and external) content policy type of the load. In that particular case, the load was of [TYPE\_DOCUMENT](https://searchfox.org/mozilla-central/rev/82c04b9cad5b98bdf682bd477f2b1e3071b004ad/dom/base/nsIContentPolicy.idl#90). The reason there is an internal and an external type is because Firefox needs to enforce different security mappings. For example, the separation of different script types allows precise mapping to e.g. CSP directives.

`**upgradeInsecureRequests**` Indicates whether the request needs to be upgraded from HTTP to HTTPS before it hits the network due to the CSP directive [upgrade-insecure-requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Upgrade-Insecure-Requests).

`**initialSecurityChecksDone**` Whenever the ContentSecurityManager performs security checks the first time for a channel, then this bit gets flipped indicating that initial security checks on that channel have been completed. In turn, this flag causes certain security checks, like e.g. [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS), to be bypassed after a redirect.

`**allowDeprecatedSystemRequests**` In one of our [hardening efforts](https://blog.mozilla.org/attack-and-defense/2020/07/07/hardening-firefox-against-injection-attacks-the-technical-details/), we started to disallow web requests in system privileged contexts, when the triggeringPrincipal is of type SystemPrincipal (see `[CheckAllowLoadInSystemPrivilegedContext()](https://searchfox.org/mozilla-central/rev/1b95a0179507a4dc7d4b0c94c2df420dc1a72885/dom/security/nsContentSecurityManager.cpp#821)`). However, there are certain system requests where we allow that temporarily, e.g., for [OCSP requests](https://searchfox.org/mozilla-central/rev/9b282b34b5aa0f836beb735656c55efb2cc4c617/security/manager/ssl/nsNSSCallbacks.cpp#275).

`**CSP**` The [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) that needs to be enforced for this request. If there is no CSP to enforce, then the value for this field remains empty.

`**securityFlags**` The [securityFlags](https://searchfox.org/mozilla-central/rev/1b95a0179507a4dc7d4b0c94c2df420dc1a72885/netwerk/base/nsILoadInfo.idl#71) indicate what kind of security checks need to be performed for that channel. For example, whether to enforce the same-origin-policy or whether this load requires [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS).

## Analyzing and Finding Security Bugs with Logging

Now that we know all the entries in a web security log, let’s dive a little deeper and investigate a historical bug and how it could have been found using the discussed logging capabilities. We will be looking at [CVE-2019-17000](https://www.mozilla.org/en-US/security/advisories/mfsa2019-34/#CVE-2019-17000), a limited CSP bypass from October 2019 that we fixed in Firefox 70. A Proof of Concept could look a bit like this:

|   ``` <meta http-equiv="Content-Security-Policy"       content="default-src 'self'; object-src data:; img-src 'none';"/> <body>   The following object element is allowed.   <object data='data:text/html,<p>Object element has an image:<br>   <img src="https://freddyb.neocities.org/danger.jpg"        alt="This image should not load"/><br>    This text is underneath the image.'>   </object> </body> ```   |
| --- |

Listing 2: Document defining a CSP blocking all images and loading an <object> tag with a data: URI.

The document in Listing 2 comes with a Content-Security-Policy that allows loading object elements which point to a data URL (object-src data:), but denies all image loads (img-src 'none'). A browser needs to perform various security checks to completely load this document: First, we’re loading the top level document in a new tab, which is allowed. Then, we adjust the document’s CSP given the `<meta>` element and load the `<object>` element according to the CSP (allowed in object-src). Given it’s data URL, the `<object`\> element is cross-origin (i.e., loaded using a [NullPrincipal](https://blog.mozilla.org/attack-and-defense/2020/06/10/understanding-web-security-checks-in-firefox-part-1/)). Therefore, the inner `<img>` element needs to be loaded using a different, unique origin. However, the image load should still be forbidden based on the img-src 'none' directive of the _parent_ document!

Let’s take a closer look at just the image request when logging in an unaffected version of Firefox:

|   ``` doContentSecurityCheck:  - channelURI: https://freddyb.neocities.org/danger.jpg  - httpMethod: GET  - loadingPrincipal: NullPrincipal  - triggeringPrincipal: NullPrincipal  - principalToInherit: nullptr  - redirectChain:  - internalContentPolicyType: TYPE_INTERNAL_IMAGE  - externalContentPolicyType: TYPE_IMAGE  - upgradeInsecureRequests: true  - initialSecurityChecksDone: false  - allowDeprecatedSystemRequests: false  - CSP:     - default-src 'self'; object-src data:; img-src 'none'  - securityFlags:    - SEC_ALLOW_CROSS_ORIGIN_INHERITS_SEC_CONTEXT    - SEC_ALLOW_CHROME ```   |
| --- |

Listing 3: The request is loaded into a new Security Context (using a NullPrincipal), but the CSP is inherited.

But here is what it looked like in the vulnerable Firefox version 69:

| ![Screenshot of the page opened in a vulnerable version of Firefox 69.](/images/csp-bypass-example.png) |
| --- |

Listing 4: Screenshot of the page opened in a vulnerable version of Firefox 69. Photo by [Mikael Seegen](https://unsplash.com/@mikael_seegen).

So, what happened here - why did the image load?

We can investigate the erroneous behavior using our logging. Here’s a listing for the very same image request from Firefox 69:

|   ``` doContentSecurityCheck:  - channelURI: https://freddyb.neocities.org/danger.jpg  - httpMethod: GET  - loadingPrincipal: NullPrincipal  - triggeringPrincipal: NullPrincipal  - principalToInherit: nullptr  - redirectChain:  - internalContentPolicyType: TYPE_INTERNAL_IMAGE  - externalContentPolicyType: TYPE_IMAGE  - upgradeInsecureRequests: true  - initialSecurityChecksDone: false  - CSP:  - securityFlags:    - SEC_ALLOW_CROSS_ORIGIN_INHERITS_SEC_CONTEXT    - SEC_ALLOW_CHROME ```   |
| --- |

Listing 5: The very same request as in Listing 3, but logged using Firefox 69.

Can you spot the difference?

In the vulnerable case (Listing 5), the security check was not aware of an effective CSP. Even though the `<object>`’s content does not inherit the origin of the page (it’s loaded with a data: URL after all) - it should inherit the CSP of the document.

An attacker could use a CSP bypass like this and target users on web pages that are susceptible to XSS or content injections. However, this bug was identified in a previous version of Firefox and has been fixed for all of our users since.

To summarize, using the provided logging mechanism allows us to effectively detect security problems by visual inspection. One could take it even further and generate graph structures for nested page loads. Using these graphs to observe where the security context (e.g., the CSP) changes can be a very powerful tool for runtime security analysis.

## Going Forward

We have explained how to enable logging mechanisms within Firefox which allows for visual inspection of every web security check performed. We would like to point out that finding security flaws might be eligible for a [bug bounty](https://www.mozilla.org/en-US/security/client-bug-bounty/). Finally, we hope the provided instructions foster security research and in turn allow researchers, bug bounty hunters and generally everyone interested in web security to contribute to Mozilla and the Security of the Open Web.
