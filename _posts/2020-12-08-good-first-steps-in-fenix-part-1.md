---
layout: post
title: "Guest Blog Post: Good First Steps to Find Security Bugs in Fenix (Part 1)"
date: 2020-12-08
categories: 
  - "bug-bounty"
  - "guest-post"
  - "hack-and-tell"
---

_This blog post is one of several guest blog posts, where we invite participants of our [bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/) to write about bugs they've reported to us._

Fenix is a newly designed Firefox for Android that officially launched in August 2020. In Fenix, many components required to run as an Android app have been rebuilt from scratch, and various new features are being implemented as well. While they are re-implementing features, security bugs fixed in the past may be introduced again. If you care about the open web and you want to participate in the [Client Bug Bounty Program of Mozilla](https://www.mozilla.org/en-US/security/client-bug-bounty/), Fenix is a good target to start with.

Let's take a look at two bugs I found in the _firefox:_ scheme that is supported by Fenix.

### Bugs Came Again with Deep Links

Fenix provides an interesting custom scheme URL _firefox://open?url=_ that can open any specified URL in a new tab. On Android, a deep link is a link that takes you directly to a specific part of an app; and the _firefox:open_ deep link is not intended to be called from web content, but its access was not restricted.

Web Content should not be able to link directly to a _file://_ URL (although a user can type or copy/paste such a link into the address bar.) While Firefox on the Desktop has long-implemented this fix, Fenix did not - I submitted [Bug 1656747](http://bugzilla.mozilla.org/show_bug.cgi?id=1656747) that exploited this behavior and navigated to a local file from web content with the following hyperlink:

`<a href="firefox://open?url=file:///sdcard/Download"> Go </a>`

But actually, the same bug affected the older Firefox for Android (unofficially referred to as Fennec) and was filed three years ago [Bug 1380950](https://bugzilla.mozilla.org/show_bug.cgi?id=1380950).

Likewise, security researcher Jun Kokatsu reported [Bug 1447853](https://bugzilla.mozilla.org/show_bug.cgi?id=1447853), which was an <iframe> sandbox bypass in Firefox for iOS. He also abused the same type of deep link URL for bypassing the popup block brought by <iframe> sandbox.

`<iframe src="data:text/html,<a href=firefox://open-url?url=https://example.com> Go </a>" sandbox></iframe>`

I found this attack scenario in [a test file of Firefox for iOS](https://github.com/mozilla-mobile/firefox-ios/blob/76faae8c6c94e22c8fbc0de72e2d06f615738e2c/UITests/firefoxScheme.html) and I re-tested it in Fenix. I submitted [Bug 1656746](https://bugzilla.mozilla.org/show_bug.cgi?id=1656746) which is the same issue as what he found.

### Conclusion

As you can see, retesting past attack scenarios can be a good starting point. We can find past vulnerabilities from the [Mozilla Foundation Security Advisories](https://www.mozilla.org/en-US/security/advisories/). By examining histories accumulated over a decade, we can see what are considered security bugs and how they were resolved. These resources will be useful for retesting past bugs as well as finding attack vectors for newly introduced features.

Have a good bug hunt!
