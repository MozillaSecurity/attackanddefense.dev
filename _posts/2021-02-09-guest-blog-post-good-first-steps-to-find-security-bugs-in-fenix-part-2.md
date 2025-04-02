---
layout: post
title: "Guest Blog Post: Good First Steps to Find Security Bugs in Fenix (Part 2)"
date: 2021-02-09
categories: 
  - "bug-bounty"
  - "guest-post"
  - "hack-and-tell"
---

 

_This blog post is one of several guest blog posts, where we invite participants of our bug bounty program to write about bugs they’ve reported to us._

Continuing with [Part 1](https://blog.mozilla.org/attack-and-defense/2020/12/08/good-first-steps-in-fenix-part-1/), this article introduces some practices for finding security bugs in Fenix.

Fenix's architecture is unique. Many of the browser features are not implemented in Fenix itself - they come from independent and reusable libraries such as GeckoView and Mozilla Android Components (known as [Mozac](https://mozac.org/)). Fenix as a browser application combines these libraries as building parts for the internals, and the [fenix project](https://github.com/mozilla-mobile/fenix) itself is primarily a User Interface. Mozac is noteworthy because it connects web contents rendered in GeckoView into the native Android world.

There are common pitfalls that lead to security bugs in the connection between web content and native apps. In this post, we’ll take a look at one of the pitfalls: private browsing mode bypasses. While looking for this class of bug, I discovered three separate but similar issues (Bugs [1657251](https://bugzilla.mozilla.org/show_bug.cgi?id=1657251), [1658231](https://bugzilla.mozilla.org/show_bug.cgi?id=1658231), and [1663261](https://bugzilla.mozilla.org/show_bug.cgi?id=1663261).)

### Pitfalls in Private Browsing Mode

Take a look at the following two lines of HTML.

`<img src="test1.png"> <link rel="icon" type="image/png" href="test2.png">`

Although these two HTML tags look similar in that they both fetch and render PNG images from the server, their internal processing is very different. In the former `<img>` tag, GeckoView fetches the image from the server and renders it in an HTML document, whereas in the latter `<link rel="icon">` tag identifies a favicon for the page, and code in Mozac fetches the image and renders it as a part of a view of Android. When I discovered these vulnerabilities in the fall of 2020, a HTTP request sent from `<img>` tag showed the string "Firefox" in the User-Agent header, whereas the request from `<link rel="icon">` showed the string "MozacFetch".

Like other browsers, GeckoView has a separated context for normal mode and private browsing mode. So the cookies and local storage areas in private browsing mode are completely separated from the normal mode, and these values are not shared. On the other hand, the URL fetch class that Mozac has - written in Kotlin - has only a single cookie store. If a favicon request responded with a Set-Cookie header; it would be stored in that cookie store and a later fetch of the favicon in private browsing mode would respond with the same cookie and vice versa. ([Bug 1657251](https://bugzilla.mozilla.org/show_bug.cgi?id=1657251)).

![](/images/pasted-image-0.png)

This same type of bug appears not only in Favicon, but also in other features that have a similar mechanism. One example is the Web Notification API. Web Notifications is a feature that shows an OS-level notification through JavaScript. Similar to favicons, an icon image can appear in the notification dialog - and it had a bug that shared private browsing mode cookies with the normal mode in the exact same way ([Bug 1658231](https://bugzilla.mozilla.org/show_bug.cgi?id=1658231)).

![](/images/pasted-image-0-1.png)

These bugs do not only occur when loading icon images. [Bug 1663261](https://bugzilla.mozilla.org/show_bug.cgi?id=1663261) points out that a similar bypass occurs when downloading linked files via _<a download>_. File downloads are also handled by Mozac's Downloads feature, which satisfies the same conditions to cause a similar flaw.

As you can see, Mozac's URL fetch is one of the places that creates inconsistencies with web content. Other than private browsing mode, there are various other security protection mechanisms in the web world, such as port blocks, HSTS, CSP, Mixed-Content Block, etc. These protections are sometimes overlooked when issuing HTTP requests from another component. By focusing on these common pitfalls, you'll likely be able to find new security bugs continuously into the future.

Using the difference in User-Agent to distinguish the initiator of the request was a useful technique for finding these kinds of bugs, but it’s no longer available in today's Fenix. If you can build Fenix yourself, you can still use this technique by setting a custom request header indicating the request from Mozac like below.

[GeckoViewFetchClient.kt](https://github.com/mozilla-mobile/android-components/blob/2d1842df117dc7e42543207bb62ed872eab5cd86/components/browser/engine-gecko-nightly/src/main/java/mozilla/components/browser/engine/gecko/fetch/GeckoViewFetchClient.kt#L89)

  ` ```   private fun WebRequest.Builder.addHeadersFrom(request: Request): WebRequest.Builder {   request.headers?.forEach { header ->     addHeader(header.name, header.value)   }  + addHeader("X-REQUESTED-BY", "MozacFetch") // add     return this      } ``` `

For monitoring HTTP requests, the [Remote Debugging](https://developer.mozilla.org/en-US/docs/Tools/Remote_Debugging) is useful. Requests sent from MozacFetch will be output to the Network tab of the Multiprocess Toolbox process in Remote Debug window.You can find requests from Mozac by filtering by the string “MozacFetch”.

![](/images/pasted-image-0-2.png)

Have a good bug hunt!
