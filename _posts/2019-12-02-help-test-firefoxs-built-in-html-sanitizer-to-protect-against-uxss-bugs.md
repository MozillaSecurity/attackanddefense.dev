---
title: "Help Test Firefox’s built-in HTML Sanitizer to protect against UXSS bugs"
date: 2019-12-02
categories: 
  - "bug-bounty"
  - "hardening"
---

_This post first appeared on the [Mozilla Security Blog](https://blog.mozilla.org/security/2019/12/02/help-test-firefoxs-built-in-html-sanitizer-to-protect-against-uxss-bugs/)_

I recently gave a talk at OWASP Global AppSec in Amsterdam and summarized the presentation in a blog post about [_how to achieve "critical"-rated code execution vulnerabilities in Firefox with user-interface XSS_](https://frederik-braun.com/firefox-ui-xss-leading-to-rce.html). The end of that blog posts encourages the reader to participate the bug bounty program, but did not come with proper instructions. This blog post will describe the mitigations Firefox has in place to protect against XSS bugs and how to test them.

Our about: pages are privileged pages that control the browser (e.g., `about:preferences`, which contains Firefox settings). A successful XSS exploit has to bypass [_the Content Security Policy (CSP), which we have recently added_](https://blog.mozilla.org/security/2019/10/14/hardening-firefox-against-injection-attacks/) but also our built-in XSS sanitizer to gain arbitrary code execution. A bypass of the sanitizer without a CSP bypass is in itself a severe-enough security bug and warrants a bounty, subject to the discretion of the Bounty Committee. See the [_bounty pag_](https://www.mozilla.org/en-US/security/client-bug-bounty/)[_es_](https://www.mozilla.org/en-US/security/client-bug-bounty/) for more information, including how to submit findings.

## How the Sanitizer works

The Sanitizer runs in the so-called [_"fragment parsing"_](https://w3c.github.io/DOM-Parsing/#dfn-fragment-parsing-algorithm) step of innerHTML. In more detail, whenever someone uses innerHTML (or similar functionality that parses a string from JavaScript into HTML) the browser builds a DOM tree data structure. Before the newly parsed structure is appended to the existing DOM element our sanitizer intervenes. This step ensures that our sanitizer can not mismatch the result the actual parser would have created - because it is indeed the actual parser.

The line of code that triggers the sanitizer is in [_nsContentUtils::ParseFragmentHTML_](https://searchfox.org/mozilla-central/rev/171109434c6f2fe086af3b2322839b346a112a99/dom/base/nsContentUtils.cpp#4683) and _nsContentUtils::ParseFragmentXML_. This aforementioned link points to a specific source code revision, to make hotlinking easier. Please click the file name at the top of the page to get to the newest revision of the source code.

The sanitizer is implemented as an allow-list of elements, attributes and attribute values in [_nsTreeSanitizer.cpp_](https://searchfox.org/mozilla-central/source/dom/base/nsTreeSanitizer.cpp). Please consult the allow-list before testing.

**Finding a Sanitizer bypass is a hunt for** [_**Mutated XSS (mXSS)**_](https://en.wikipedia.org/wiki/Cross-site_scripting#Mutated_XSS_\(mXSS\)) **bugs in Firefox** – Unless you find an element in our allow-list that has recently become capable of running script.

## How and where to test

A browser is a complicated application which consists of millions of lines of code. If you want to find new security issues, you should test the [_latest development version_](https://nightly.mozilla.org/). We often times rewrite lots of code that isn't related to the issue you are testing but might still have a side-effect. To make sure your bug is actually going to affect end users, test [_Firefox Nightly_](https://nightly.mozilla.org/). Otherwise, the issues you find in Beta or Release might have already been fixed in Nightly.

## Sanitizer runs in all privileged pages

Some of Firefox’s internal pages have more privileges than regular web pages. For example about:config allows the user to modify advanced browser settings and hence relies on those expanded privileges.

Just open a new tab and navigate to about:config. Because it has access to privileged APIs it can not use innerHTML (and related functionality like outerHTML and so on) without going through the sanitizer.

### Using Developer Tools to emulate a vulnerability

From `about:config`, open The developer tools console (Go to _Tools_ in the menu bar. Select _Web Developers_, then _Web Console_ (Ctrl+Shift+k)).

To emulate an XSS vulnerability, type this into the console:

```
document.body.innerHTML = '<img src=x onerror=alert(1)>'
```

Observe how Firefox sanitizes the HTML markup by looking at the error in the console:

```
“Removed unsafe attribute. Element: img. Attribute: onerror.”
```

You may now go and try other variants of XSS against this sanitizer. Again, try finding an mXSS bug or by identifying an allowed combination of element and attribute which execute script.

### Finding an actual XSS vulnerability

Right, so for now we have _emulated_ the Cross-Site Scripting (XSS) vulnerability by typing in innerHTML ourselves in the Web Console. That's pretty much cheating. But as I said above: _What we want to find are sanitizer bypasses. This is a call to test our mitigations._

But if you still want to find real XSS bugs in Firefox, I recommend you run some sort of smart static analysis on the Firefox JavaScript code. And by smart, I probably do not mean [_eslint-plugin-no-unsanitized_](https://github.com/mozilla/eslint-plugin-no-unsanitized).

## Summary

This blog post described the mitigations Firefox has in place to protect against XSS bugs. These bugs can lead to remote code execution outside of the sandbox. We encourage the wider community to double check our work and look for omissions. This should be particularly interesting for people with a web security background, who want to learn more about browser security. Finding severe security bugs is very rewarding and we’re looking forward to getting some feedback. If you find something, please consult the [_Bug Bounty pages_](https://www.mozilla.org/en-US/security/bug-bounty/) on how to report it.
