---
title: "Insights into HTTPS-Only Mode"
date: 2021-03-10
categories: 
  - "firefox-internals"
  - "security-internals"
---

 

In a recent academic publication titled [HTTPS-Only: Upgrading all connections to https in Web Browsers](http://blog.mozilla.org/attack-and-defense/files/2021/03/httpsonly_paper.pdf) (to appear at [MadWeb - Measurements, Attacks, and Defenses for the Web](https://madweb.work/program21/)) we present a new browser connection model which paves the way to an ‘https-by-default’ web. In this blogpost, we provide technical details about HTTPS-Only Mode’s upgrading mechanism and share data around the success rate of this feature. _(Note that links to source code are perma-linked to a recent revision as of this blog post. More recent changes may have changed the location of the code in question.)_

 

### Connection Model of HTTPS-Only

The fundamental security problem of the current browser practice of defaulting to use insecure http, instead of secure https, when initially connecting to a website, is that attackers can intercept the initial request to a website. Hijacking the initial request suffices for an attacker to perform a man-in-the-middle attack, which in turn allows the attacker to downgrade the connection, eavesdrop or modify data sent between client and server.

\[caption id="attachment\_207" align="aligncenter" width="600"\]![](images/principle_approach-600x281.jpg) **Left:** The current standard behavior of browsers defaulting to http with a server reachable over https; **Right:** HTTPS-Only behaviour defaulting to https with fallback to http when a server is not reachable over https.\[/caption\]

 

**_Industry-wide default Connection Model:_** Current best practice to counter the explained man-in-the-middle security risk primarily relies on [HTTP Strict-Transport-Security (HSTS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security). However, HSTS does not solve the problems associated with performing the initial request in plain http. As illustrated in the above Figure (left), the current browser default is to first connect to foo.com using http (see 1). If the server follows best practice and implements HSTS, then the server responds with a redirect to the secure version of the website (see 2). After the next GET request (see 3) the server adds the HSTS response header (see 4), signalling that the server prefers https connections and the browser should always perform https requests to foo.com (see 5).

**_HTTPS-Only Connection Model:_** In contrast and as illustrated in the above Figure (right), the presented HTTPS-Only approach first tries to connect to the web server using https (see 1). Given that most popular websites support https, our upgrading algorithm commonly establishes a secure connection and starts loading content. In a minority of cases, connecting to the server using https fails and the server reports an error (see 2). The proposed HTTPS-Only Mode then prompts the user, explaining the security risk, to either abandon the request or to connect using http (see 3).

 

### Implementation Details of HTTPS-Only

We designed HTTPS-Only Mode following the principle of _Secure by Default_ which means that by default, our approach will upgrade all outgoing connections from http to https. Following this principle allows us to provide a future-proof implementation where exceptions to the rule require explicit annotation by setting the flag [HTTPS\_ONLY\_EXEMPT](https://searchfox.org/mozilla-central/rev/26330a08b1f9d06938faa0aa5e0f8c7a58064aa2/netwerk/base/nsILoadInfo.idl#457).

Our proposed security-enhancing feature internally upgrades (a) top-level document loads as well as (b) all subresource loads (images, stylesheets, scripts) within a secure website by rewriting the scheme of a URL from http to https. Internally this upgrading algorithm is realized by consulting the function [nsHTTPSOnlyUtils::ShouldUpgradeRequest()](https://searchfox.org/mozilla-central/rev/26330a08b1f9d06938faa0aa5e0f8c7a58064aa2/dom/security/nsHTTPSOnlyUtils.cpp#111).

Upgrading a top-level (document) request with HTTPS-Only entails uncertainties about the response that the browser needs to handle. For example, a non-responding firewall or a misconfigured or outdated server that fails to send a response can result in long timeouts. To mitigate this degradation of a users browsing experience, HTTPS-Only first sends a top-level request for https, and after a three second delay, if no response is received, sends an additional http background request by calling the function [nsHTTPSOnlyUtils::PotentiallyFireHttpRequestToShortenTimout()](https://searchfox.org/mozilla-central/rev/44695ef057e422a8d6c6056972bdf32766c36187/dom/security/nsHTTPSOnlyUtils.cpp#45). If the background http connection is established prior to the https connection, then this signal is a strong indicator that the https request will result in a timeout. In this case, the https request is canceled and the user is shown the HTTPS-Only Mode exception page.

In addition to comprehensively enforcing https for sub-resources, HTTPS-Only also accounts for [WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket) by consulting the function [nsHTTPSOnlyUtils::ShouldUpgradeWebSocket()](https://searchfox.org/mozilla-central/rev/3f97afc8db535f9b0232222cb48cc4cbf8334c76/dom/security/nsHTTPSOnlyUtils.cpp#168).

Ultimately, HTTPS-Only also needs full integration with two critical browser security mechanisms related to subresource loads: (a) [Mixed Content Blocker](https://developer.mozilla.org/en-US/docs/Web/Security/Mixed_content), and (b) [Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS). We adapt both security mechanisms by consulting  [nsHTTPSOnlyUtils::IsSafeToAcceptCORSOrMixedContent()](https://searchfox.org/mozilla-central/rev/26330a08b1f9d06938faa0aa5e0f8c7a58064aa2/dom/security/nsHTTPSOnlyUtils.cpp#302).

 

### Success Rate of HTTPS-Only

The HTTPS-Only approach specifically aims to ensure connections use the secure https protocol, where browsers traditionally would connect using the http protocol. To target this data set, we record information when HTTPS-Only Mode is able to upgrade a connection by rewriting the scheme of a URL from http to https and a load succeeds.

\[caption id="attachment\_208" align="aligncenter" width="486"\]![](images/top-level-pie-chart-error-page-600x360.jpg) Attempts to upgrade top-level (document) legacy addresses from http to https via HTTPS-Only (data collected between Nov 17th and Dec. 17th 2020).\[/caption\]

 

As illustrated in the Figure above, the HTTPS-Only mechanism successfully upgrades top-level (document) loads from http to https for more than 73% of legacy addresses. These 73% of successful upgrades originate from the user clicking legacy http links, or entering http (or even scheme-less) URLs in the address-bar, where the target website, fortunately, supports https.

Our observation that HTTPS-Only can successfully upgrade seven out of ten of top-level loads from http to https reflects a general migration of websites supporting https. At the same time, this fraction of successful upgrades of legacy addresses also confirms that web pages still contain a multitude of http-based URLs where browsers would traditionally establish an insecure connection.

\[caption id="attachment\_209" align="aligncenter" width="600"\]![](images/top-level-overview-pie-600x316.jpg) Use of https for top-level (document) loads when HTTPS-Only is enabled (data collected between Nov 17th and Dec. 17th 2020).\[/caption\]

 

As illustrated in the Figure above, our collected information shows that in 92.8% of cases the top-level URL is already https without any need to upgrade. We further see that HTTPS-Only users experienced a successful upgrade from http to https in 3.5% of the loads, such that, overall, 96.3% of page loads are secure. The remaining 3.7% of page loads are insecure, on websites for which the user has explicitly opted to allow insecure connections. Note that the total number of insecure top-level loads observed depends on how many pages a user visited on the website or websites they had exempted from HTTPS-Only.

 

### Going Forward

Providing architectural insights into the security design of a system is crucial for truly working in the open. We hope that sharing technical details and the collected upgrading information allows contributors, hackers, and researchers to verify our claims or even build new research models on top of the provided insights.
