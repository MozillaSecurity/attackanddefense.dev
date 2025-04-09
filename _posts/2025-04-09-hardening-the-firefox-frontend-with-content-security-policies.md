---
layout: post
title:  "Hardening the Firefox Frontend with Content Security Policies"
date:   2025-04-09 10:00:00 +0100
author: Tom Schuster, Frederik Braun, Christoph Kerschbaumer
---

Most of the Firefox User Interface (UI), including the address bar and the tab strip, are implemented using standard web technologies like HTML, CSS and JavaScript plus some additional custom components like [XUL](https://firefox-source-docs.mozilla.org/browser/components/storybook/docs/README.xul-and-html.stories.html). One of the advantages of using web technologies for the front end is that it allows rendering the frontend using the browser engine on all desktop operating systems. However, just like many web applications are susceptible to some form of injection attack ([OWASP Top Ten](https://owasp.org/www-project-top-ten/)), Firefox’s use of web technologies for the frontend makes it no exception and hence it is vulnerable to injection attacks as well.

The most well known type of injection attack are Cross-Site Scripting (XSS) attacks. Like the name suggests, these attacks violate the boundary between different sites and circumvent first line security defenses like the [same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy). As with the existence of boundaries between different sites, the Firefox UI, which runs in the parent process, is separated from web content running in [lower-privileged content processes](https://hacks.mozilla.org/2021/05/introducing-firefox-new-site-isolation-security-architecture/). The sandboxed web content should not be able to inject content, especially code, into the parent process. Of course there needs to be some way for both parent and child processes to communicate with each other \- otherwise it would, for example, be impossible to update a tab’s title in the UI after a web page sets its own document title. The way Firefox accomplishes this interaction is through a system called IPC \- [inter-process communication](https://attackanddefense.dev/bug-bounty/firefox-internals/hack-and-tell/2021/04/27/examining-javascript-inter-process-communication-in-firefox.html).

In Pwn2Own (a computer hacking contest) 2022 a participant managed to find a chain of exploits that allowed them to escape the web content sandbox (cf. [write-up by a ZDI employee](https://www.zerodayinitiative.com/blog/2022/8/23/but-you-told-me-you-were-safe-attacking-the-mozilla-firefox-renderer-part-2), a [generalized introduction](https://attackanddefense.dev/bug-bounty/firefox-internals/hack-and-tell/2021/04/27/examining-javascript-inter-process-communication-in-firefox.html) or [video by LifeOverflow](https://www.youtube.com/watch?v=StQ_6juJlZY&t=0s)). A part of this exploit chain involved creating a JavaScript inline event handler (using setAttribute) in the Firefox UI, and then triggering the execution of that event handler. For websites, the well known approach of mitigating XSS attacks using inline event handlers usually involves using a [Content-Security-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP) (CSP). CSP allows for restricting what scripts are allowed to execute on a given page. Typically by only allowing scripts from specific URLs, or with specific hashes. In particular, unless `‘unsafe-inline’` is specified, all attempts to define inline event handlers are blocked by the browser. And because the Firefox UI uses HTML with some special sauce, we can actually also use CSPs in the same way to harden the Firefox frontend code.

## Progress

The main Firefox UI, which contains the tabs, address bar, menu bar etc. is actually just one big XHTML document called [browser.xhtml](https://searchfox.org/mozilla-central/source/browser/base/content/browser.xhtml). Recently we have removed in total over 600 inline event handlers across [50 bugs](https://bugzilla.mozilla.org/show_bug.cgi?id=1890547) from this XHTML document. At this point we want to acknowledge the support of the Firefox frontend team, without their co-operation we wouldn’t have been able to land so many changes in such a short amount of time.

![Graph showing the number of inline event handlers in brower.xhtml](/images/2025/inline-event-handlers-in-browser-xhtml-progress.png)

Figure 1: Graph showing the number of inline event handlers in browser.xhtml over time in Firefox Nightly. The numbers were estimated using grep.

### Interlude: How to replace inline event handlers

In case you are a Firefox developer, the maintainer of a Firefox fork like the Tor Browser, or just a web developer interested in securing your own website, the following might be relevant to you. The process of removing inline event handlers usually involves finding all the places that define an inline event handler like `<button onclick="buttonClicked()">` and then replacing this with a call to `addEventListener` from a new JS file. Roughly like this:

```js
let button = document.querySelector("button");
button.addEventListener("click", buttonClicked);
```

However there are some important differences to keep in mind between JS code running as inline event handlers and normal event handlers. Firstly, it’s possible to `return false;` from the inline event handler, which is equivalent to calling `event.preventDefault()`. Also note that `this`, which is `event.currentTarget` for inline event handlers, would change if you replaced it with an [arrow function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions) as the event listener.

## Outlook

While browser.xhtml contains most parts of the main Firefox UI, some windows and sidebars are based on their own (X)HTML files. Due to the fact that browser.xhtml provides the largest attack vector of our frontend code we focused our initial efforts on securing and hardening browser.xhtml which already drastically improves the status quo to prevent inline script execution. For other windows, such as the “About Firefox” dialog, we go even further by adding CSPs that are much more restrictive and go beyond just blocking scripts. If you are familiar with CSP, by default we set a baseline CSP like `default-src chrome: resource:;`, which basically means we only allow loading resources from files that are shipped with Firefox.
Historically we have already added CSPs to `about:` pages such as `about:preferences` that are displayed like normal websites (cf. [Hardening Firefox against Injection Attacks](https://blog.mozilla.org/attack-and-defense/2020/07/07/hardening-firefox-against-injection-attacks-the-technical-details/)). Our end goal is to block all dynamic code execution in Firefox (like `eval`) entirely, to provide the best and most secure Firefox version possible that is resilient to any kind of XSS attacks.

## Summary

We have rewritten over 600 JavaScript event handlers to mitigate XSS and other injection attacks in the main Firefox user interface. This mitigation will ship in Firefox 138\. However, blocking the execution of scripts in the parent process is not the end \- we will expand this technique to other contexts in the near future. There is still more work to do as the UI requires JavaScript APIs with a high level of privileges. **However: We still eliminated a whole class of attacks**, significantly raising the bar for attackers to exploit Firefox. In fact, we hopefully just broke someone’s exploit chain.
