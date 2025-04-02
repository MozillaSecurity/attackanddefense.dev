---
layout: post
title: "Remote Code Execution in Firefox beyond memory corruptions"
date: 2019-09-29
categories: 
  - "hack-and-tell"
---

_This is the blog post version of my presentation form OWASP Global AppSec in Amsterdam 2019. It was presented in the [AllStars Track](https://ams.globalappsec.org/program/allstars)._

### Abstract:

> _Browsers are complicated enough to have attack surface beyond memory safety issues. This talk will look into injection flaws in the user interface of Mozilla Firefox, which is implemented in JS, HTML, and an XML-dialect called XUL. With an Cross-Site Scripting (XSS) in the user interface attackers can execute arbitrary code in the context of the main browser application process. This allows for cross-platform exploits of high reliability. The talk discusses past vulnerabilities and will also suggest mitigations that benefit Single Page Applications and other platforms that may suffer from DOM-based XSS, like Electron._

## Prologue

(This is the part, where we reduce the lighting and shine a flashlight into my face)

_Listen, well, young folks. Old people, browser hackers or Mozilla fanboys, might use this as an opportunity to lean back and stroke their mighty neckbeard, as they have heard all of this before_

It was the year 1997, and people thought XML was a great idea. In fact, it was so much better than its warty and unparseable predecessor HTML. While XHTML was the clear winner and successor for great web applications, it was _obvious_ that XML would make a great user interface markup language to create a powerful cross-platform toolkit dialect. This folly marks the hour of birth for XUL. XUL was created as the _XML User Interface Language_ at Netscape (the company that created the origins of the Mozilla source code. Long story. The younger folks might want to read upon Wikipedia or watch the amazing Movie "Code Rush", which is available on archive.org). Jokingly, XUL was also a reference to the classic 1984 movie Ghostbusters, in which an evil deity called Zuul (with a Z) possesses innocent people.

Time went by and XUL did not take off as a widely-recognized standard for cross-platform user interfaces. Firefox has almost moved from XUL and re-implemented many parts in HTML. Aptly named after an evil spirit, we will see that XUL still haunts us today.

## Mapping the attack surface

Let's look into Firefox, to find some remnants of XUL, by visiting some internal pages. Let's take a look at some Firefox internal pages. By opening `about:preferences` in a new tab (I won't be able to link to it for various good reasons). Now either look at the source code using the Developer Tools (right-click "Inspect Element") or [view the source code of Firefox Nightly using the source code search at searchfox.org](https://searchfox.org/mozilla-central/source/browser/components/preferences/in-content/preferences.xul).

We can also open the developer console and poke around with the obscure objects and functions that are available for JavaScript in privileged pages. As a proof-of-concept, we may `alert(Components.stack)`, which gives us a stringified JavaScript call stack - notably this is a JavaScript object that is left undefined for normal web content.

Inspecting the source code we also already see some markup that screams both _XML_ as well as _XML-dialect_. While still in our information gathering phase, we will not go too deep, but make note of two observations: - XUL is not HTML. To get a better understanding of elements like `<command>`, `<colorpicker>` or `<toolbar>`, we will be able to look at the [XUL Reference on MDN](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XUL/XUL_Reference) - XUL is scriptable! A `<script>` tag exists and it may contain JavaScript.

There are also some newer pages like `about:crashes`, which holds previously submitted (or unsubmitted) crash reports. Whether those internal pages are written in (X)HTML or XUL. Most of the interacive parts are written in JavaScript. I suppose most of you will by now understand that we are looking for Cross-Site Scripting (XSS) vulnerabilities in the browser interface. What's notable here, is that this bypasses the sandbox.

As an aside the page behind `about:cache` is actually implemented using C++ that emits HTML-ish markup.

## Let's start with search and grep

Being equipped with the right kind of knowledge and the crave for a critical Firefox bug under my name, I started using our code search more smartly. Behold:

> Search: `.innerHTML =`
> 
> Number of results: 1000 (maximum is 1000)

Hm. Excluding test files.

> Search: `innerHTML =`
> 
> Number of results: 414

That's still a lot. And that's not even all kinds of XSS sinks. I would also look for `outerHTML`, `insertAdjacentHTML` and friends.

> Search (long and hairy regular expression that tries to find more than innerHTML)
> 
> Number of results: 997

That's bad. Let's try to be smarter!

## JavaScript Parsing - Abstract Syntax Trees. ESLint to the rescue!

I've actually dabbled in this space for a long while before. This would be another talk, but a less interesting one. So I'll skip ahead and tell you that I wrote an eslint plugin, that will analyze JavaScript files to look for the following:

1. Checking the right-hand side in assignments (`+`, `+=`) where the left part ends with either `innerHTML` or `outerHTML`.
2. Checking the first argument in calls to `document.write()`, `document.writeln()`, `eval` and the second argument for `insertAdjacentHTML`.

For both, we'll check whether they contain a variable. String literals or empty strings are ignored. The plug-in is available at as [`eslint-plugin-no-unsanitized`](https://github.com/mozilla/eslint-plugin-no-unsanitized) and allows for configuration to detect and ignore built-in escape and sanitize functions. If you're worried about DOM XSS, I recommend you check it out.

## Discovered Vulnerabilities

Using this nice extension to scan all of Firefox yields us a handy amount of 32 matches. We create a spreadsheet and audit all of them by hand. Following around long calling chains, with unclear input values and patterns that either escape HTML close to the final `innerHTML`, upon creation or stuff that's extracted from databses (like the browsing history), which does its escaping upon insertion.

* * *

Many nights later

* * *

### A first bug appears

Heureka! This sounds interesting:

```
  let html = `
    <div style="flex: 1;
                display: flex;
                padding: ${IMAGE_PADDING}px;
                align-items: center;
                justify-content: center;
                min-height: 1px;">
      <img class="${imageClass}"
           src="${imageUrl}"/>       <----- boing
    </div>`;
  // …
  div.innerHTML = html;

```

When hovering over an markup that points to an image in the web developer tools, they will helpfully create a tooltip that preloads and shows the image for the web developer to enjoy. Unfortunately, that URL is not escaped.

### Writing the exploit

After spending a few sleepless nights on this, I didn't get anything beyond a XML-conformant proof of concept of <button>i</button>. At some point I filed the bug as `sec-moderate`, i.e., this is almost bad, but likely needs another bug to be actually terrible. [I wrote](https://bugzilla.mozilla.org/show_bug.cgi?id=1372112#c22):

> I poked a bit again and I did not get further than <button>i</button> for various reasons … **In summary: I'd be amazed to see if someone else gets any farther.**

A few nights later, I actually came up with an exploit that breaks the existing syntax while staying XML conformant. We visit an evil web page that looks like this:

```
<img src='data:bb"/><button><img src="x" onerror="alert(Components.stack)" /></button><img src="x'>

```

The image URL that is used in the vulnerable code spans all the way from `data:` to the closing single quote at the end. Our injection alerts `Components.stack`, which indicates that we have left the realms of mortal humans.

This is [Bug 1372112 (CVE-2017-7795)](https://bugzilla.mozilla.org/show_bug.cgi?id=1372112). Further hikes through our spreadsheets of eslint violations lead to [Bug 1371586 (CVE-2017-7798)](https://bugzilla.mozilla.org/show_bug.cgi?id=1371586). Both were fixed in Firefox 56, which was released in the fall of 2017.

We find and fix some minor self-xss bugs (e.g., creating a custom preference in `about:config` with the name of `<button>hi</button>` lead to XUL injections. All of them are fixed and we're fearful that mistakes will be made again.

Critical bugs are a great way to impact coding style discussions and it is decided that the linter might as well be included in all of our linters. `innerHTML` and related badness is forbidden and we rub our hands in glee. Unfortunately, it turned out that lots of legacy code will not be rewritten and security engineers do not want to deal with the affairs of front end engineers (joke's on me in the end though, I promise). So, we allow some well-audited and finely escaped functions with a granular and exception, that gives us a confident feeling of absolute security (it's a trap!)

```
// eslint-disable-next-line no-unsanitized/property

```

## A Dark Shadow

I feel like I have eradicated the bug class from the entirety of our codebase. We may now look for more complicated bugs and our days get more exciting.

Of course, I wander through the office bragging with my cleverness, warning young folks from the danger of XSS and proudly wearing my security t-shirts. There's lots of colorful war stories to be told and even more free snacks or fizzy drinks to be consumed.

Meanwhile: My great colleagues that contribute and actually develope useful stuff. On top of their good work, some of them even mentor aspiring students and enthusiastic open source fans. Having listened to my stories of secure and well-audited code that should eventually be replaced, they make an effort to get someone remove all of the danger, so we get to live in an exception-less world that truly disallows all without these pesky `eslint-disable-next-line` comments.

Naturally, code is being moved around, refactored and improved by lots of other people in the organization.

So, while I'm sitting there, enjoying my internet fame (just browsing memes, really), people show up at my desk asking me for a quick look at something suspicious:

```
// eslint-disable-next-line no-unsanitized/property
doc.getElementById("addon-webext-perm-header").innerHTML = strings.header;

// data coming *mostly* from localization-templates
    let strings = {
      header: gNavigatorBundle.getFormattedString("webextPerms.header", [data.name]),
      text: gNavigatorBundle.getFormattedString("lwthemeInstallRequest.message2",
                                                [uri.host]),
// ..
// but of course all goes through _sanitizeTheme(aData, aBaseURI, aLocal)
// (which does not actually sanitize HTML)

```

I feel massively stupid and re-create my spreadsheet. Setting eslint to ignore the `disable-next-line` stuff locally allows me to start all over. We build an easy exploit that pops calc. How funny! We also notice that a few more bugs like that have crept in, since the "safe" call sites were whitelisted. Yikes.

Having learned about XML namespaces, a simpler example payload (without the injection trigger) would like look this:

```
<html:img onerror='Components.utils.import("resource://gre/modules/Subprocess.jsm");Subprocess.call({ command: "/usr/bin/gnome-open" });' src='x'/>,

```

This is [Bug 1432778](https://bugzilla.mozilla.org/show_bug.cgi?id=1432778).

## Hope on the horizon

A good patch is made and circulated with a carefully selected group of senior engineers. We have various people working on the code and are concerned about this being noticed by bad actors. With the help of the aforementioned group, we convince engineering leadership that this warrants an unscheduled release of Firefox. We start a simplified briefing for Release Management and QA.

People point out that updates always take a while to apply to all of our release base and shipping a new version with a single commit that replaces `.innerHTML` with `.textContent` seems a bit careless. Anyone with a less-than sign on their keyboard could write a "1-day exploit" that would affect lots of users.

What can we do? We agree that DOM XSS deserves a heavier hammer and change our implementation for HTML parsing (which is being used in `innerHTML`, `outerHTML`, `insertAdjacentHTML`, etc.). Normally, this function parses the DOM tree and inserts where assigned. But now, for _privileged JavaScript_, we parse the DOM tree and omit all kinds of badness before insertion. Luckily, we have something like that in our source tree. In fact, I have [tested it in 2013](https://frederik-braun.com/secreview-750436.html). We also use this to strip HTML email from `<script>` and its friends in Thunderbird, so it's even battle-tested. On top, we do some additional manual testing and identify some problems around leaving form elements in, which warrants follow-up patches in the future.

A nice benefit is that a commit which changes how DOM parsing works, doesn't allow reverse engineering our vulnerability from the patch. Neat.

In the next cycles, we've been able to make it stricter and removed more badness (e.g., form elements). This was [Bug 1432966: Sanitize HTML fragments created for chrome-privileged documents (CVE-2018-5124)](https://bugzilla.mozilla.org/show_bug.cgi?id=1432966)

## Closing credits and Acknowledgements

Exploitation and Remediation were achieved with the support of various people. Thanks to, security folks, Firefox engineers, release engineers, QA testers. Especially to Johnathan Kingston (co-maintainer of the eslint plugin), Johann Hofman, who found the bad 0day in 2018 and helped testing, shaping of and arguing for an unscheduled release version of Firefox.

No real geckos were harmed in the making of this blog post.
