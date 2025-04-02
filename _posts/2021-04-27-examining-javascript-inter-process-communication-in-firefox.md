---
layout: post
title: "Examining JavaScript Inter-Process Communication in Firefox"
date: 2021-04-27
categories: 
  - "bug-bounty"
  - "firefox-internals"
  - "hack-and-tell"
---

 

Firefox uses Inter-Process Communication (IPC) to implement privilege separation, which makes it an important cornerstone in our security architecture. A previous blog post focused on [fuzzing the C++ side of IPC](https://blog.mozilla.org/attack-and-defense/2021/01/27/effectively-fuzzing-the-ipc-layer-in-firefox/). This blog post will look at IPC in JavaScript, which is used in various parts of the user interface. First, we will briefly revisit the multi-process architecture and upcoming changes for [Project Fission](https://wiki.mozilla.org/Project_Fission), Firefox’ implementation for Site Isolation. We will then move on to examine two different JavaScript patterns for IPC and explain how to invoke them. Using Firefox’s Developer Tools (DevTools), we will be able to debug the browser itself.

Once equipped with this knowledge, we will revisit a sandbox escape bug that was used in a [0day attack against Coinbase in 2019](https://blog.coinbase.com/responding-to-firefox-0-days-in-the-wild-d9c85a57f15b) and reported as [CVE-2019-11708](https://www.mozilla.org/en-US/security/advisories/mfsa2019-19/#CVE-2019-11708). This 0day-bug has found extensive coverage in [blog posts](https://blog.exodusintel.com/2020/11/10/firefox-vulnerability-research-part-2/) and [publicly available exploits](https://github.com/0vercl0k/CVE-2019-11708). We believe the bug provides a great case study and the underlying techniques will help identify similar issues. Eventually, by finding more sandbox escapes you can help secure hundreds of millions of Firefox users as part of the [Firefox Bug Bounty Program](https://www.mozilla.org/en-US/security/client-bug-bounty/).

## **Multi-Process Architecture Now and Then**

As of April 2021, Firefox uses one privileged process to launch other process types and coordinate activities. These types are web content processes, semi-privileged web content processes (for special websites like accounts.firefox.com or addons.mozilla.org) and four kinds of utility processes for web extensions, GPU operations, networking or media decoding. Here, we will focus on the communication between the main process (also called "parent") and a multitude of web processes (or "content" processes).

Firefox is shifting towards a new security architecture to achieve Site Isolation, which moves from a “process per tab” to a “process per [site](https://html.spec.whatwg.org/multipage/origin.html#sites)” architecture.

\[caption id="attachment\_228" align="aligncenter" width="1123"\]![Left: Firefox using roughly a process per tab - Right: Fission-enabled Firefox, which uses a process per site (i.e., a seperate one for each banner ad and social button).](/images/new-fission.png) Left: Current Firefox generally grouping a tab in its own process. Right: Fission-enabled Firefox, separating each site in its own process\[/caption\]

The parent process acts as a broker and trusted user interface host. Some features, like our settings page at about:preferences are essentially web pages (using HTML and JavaScript) that are hosted in the parent process. Additionally, various control features like modal dialogs, form auto-fill or native user interface pieces (e.g., the `<select>` element) are also implemented in the parent process. This level of privilege separation also requires receiving messages from content processes.

Let's look at JSActors and MessageManager, the two most common patterns for using inter-process communication (IPC) from JavaScript:

### **JSActors**

Using a [JSActor](https://firefox-source-docs.mozilla.org/dom/ipc/jsactors.html) is the preferred method for JS code to communicate between processes. JSActors always come in pairs - with one implementation living in the child process and the counterpart in the parent. There is a separate parent instance for every pair in order to closely and consistently associate a message with either a specific content window (JSWindowActors), or child process (JSProcessActors).

Since all JSActors are lazy-loaded we suggest to exercise the implemented functionality at least once, to ensure they are all present and allow for a smooth test and debug experience.

\[caption id="attachment\_229" align="aligncenter" width="555"\]![Inter-Process Communication building on top of JSActors and implemented as FooParent and FooChild](/images/jsactors.png) Inter-Process Communication building on top of JSActors and implemented as FooParent and FooChild\[/caption\]

The example diagram above shows a pair of JSActors called _FooParent_ and _FooChild_. Messages sent by invoking _FooChild_ will only be received by a _FooParent_. The child instance can send a _one-off_ message with `sendAsyncMesage("someMessage", value)`. If it needs a response (wrapped in a `Promise`), it can send a query with `sendQuery("someMessage", value)`.

The parent instance must implement a `receiveMessage(msg)` function to handle all incoming messages. Note that the messages are namespace-tied between a specific actor, so a _FooChild_ could send a message called _Bar:DoThing_ but will never be able to reach a _BarParent_. Here is some example code ([permalink, revision from March 25th](https://searchfox.org/mozilla-central/diff/3075dbd453f011aaf378bcac6b2700dccfcf814c/browser/actors/PromptParent.jsm)) which illustrates how a message is handled in the parent process.

\[caption id="attachment\_221" align="aligncenter" width="506"\]![Code sample for a receiveMessage function in a JSActor](/images/promptparent-excerpt.png) Code sample for a `receiveMessage` function in a JSActor\[/caption\]

As illustrated, the _PromptParent_ has a `receiveMessage` handler (line 127) and is passing the message data to additional functions that will decide where and how to open a prompt from the parent process. Message handlers like this and its callees are a source of untrusted data flowing into the parent process and provide logical entry points for in-depth audits

### **Message Managers**

Prior to the architecture change in Project Fission, most parent-child IPC occurred through the MessageManagers system. There were multiple message managers, including the per-process message manager and the content frame message manager, which was loaded per-tab.

Under this system, JS in both processes would register message listeners using the `addMessageListener` methods and would send messages with `sendAsyncMessage`, that have a name and the actual content. To help track messages throughout the code-base their names are usually prefixed with the components they are used in (e.g., `SessionStore:restoreHistoryComplete`).

Unlike JSActors, Message Managers need verbose initialization with addMessageListener and are not tied together. This means that messages are available for all classes that listen on the same message name and can be spread out through the code base.

\[caption id="attachment\_230" align="aligncenter" width="555"\]![Inter-Process Communication using MessageManager](/images/image6.png) Inter-Process Communication using MessageManager\[/caption\]

As of late April 2021, our AddonsManager - the code that handles the installation of WebExtensions into Firefox - is using MessageManager APIs:

\[caption id="attachment\_222" align="aligncenter" width="511"\]![Code sample for a receiveMessage function using the MessageManger API](/images/addonsmanager-receivemessage-code-sample.png) Code sample for a `receiveMessage` function using the MessageManger API\[/caption\]

The code ([permalink to exact revision](https://searchfox.org/mozilla-central/rev/6309f663e7396e957138704f7ae7254c92f52f43/toolkit/mozapps/extensions/addonManager.js#216)) for setting a MessageManager looks very similar to the setup of a JSActor with the difference that messaging can be used synchronously, as indicated by the [sendSyncMessage call in the child process](https://searchfox.org/mozilla-central/rev/6309f663e7396e957138704f7ae7254c92f52f43/toolkit/mozapps/extensions/amInstallTrigger.jsm#64). Except for the lack of lazy-loading, you can assume the same security considerations: Just like with JSActors above, the `receiveMessage` function is where the untrusted information flows from the child into the parent process and should therefore be the focus of additional scrutiny.

Finally, if you want to inspect `MessageManager` traffic live, you can use our [logging framework](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Gecko_Logging) and run Firefox with the environment variable `MOZ_LOG` set to `MessageManager:5`. This will log the received messages for all processes to the shell and give you a better understanding of what’s being sent and when.

## **Inspecting, Debugging, and Simulating JavaScript IPC**

Naturally, source auditing a `receiveMessage` handler is best paired with testing. So let's discuss how we invoke these functions in the child process and attach a JavaScript debugger to the parent process. This allows us to simulate a scenario where we have already full control over the child process. For this, we recommend you download and test against [Firefox Nightly](https://nightly.mozilla.org/) to ensure you're testing the latest code - it will also give you the benefit of being in sync with codesearch for the latest revisions at [https://searchfox.org](https://searchfox.org). For best experience, we recommend you [download Firefox Nightly](https://nightly.mozilla.org/) right now and follow this part of the blog post step by step.

**DevTools Setup - Parent Process**

First, set up your Firefox Nightly to enable browser debugging. Note that the instructions for how to enable browser debugging can change over time, so it's best you cross-check with the [instructions for Debugging the browser on MDN](https://developer.mozilla.org/en-US/docs/Tools#debugging_the_browser).

Open the Developer Tools, click the "**···**" button in the top-right and find the settings. Within _Advanced settings_ in the bottom-right, check the following:

- _Enable browser chrome and add-on debugging toolboxes_
- _Enable remote debugging_

\[caption id="attachment\_231" align="aligncenter" width="1662"\]![Enabling Browser debugging in Firefox Developer Tools](/images/image4.png) Enabling Browser debugging in Firefox Developer Tools\[/caption\]

Restart Firefox Nightly and open the Browser debugger (Tools -> Browser Tools -> Browser Toolbox). This will open a new window that looks very similar to the common DevTools.

This is your debugger for the parent process (i.e., Browser Toolbox = Parent Toolbox).

The frame selector button, which is left of the three balls "**···**" will allow you to select between windows. Select browser.xhtml, which is the main browser window. Switching to the Debug pane will let you search files and find the Parent actor you want to debug, as long as they have been already loaded. To ensure the _PromptParent_ actor has been properly initialized, open a new tab on e.g. [https://example.com](https://example.com) and make it call `alert(1)` from the normal DevTools console.

\[caption id="attachment\_223" align="aligncenter" width="1847"\]![Hitting a breakpoint in Firefox’s parent process using Firefox Developer Tools (left)](/images/alert-as-triggered-from-website-in-parent-devtools.png) Hitting a breakpoint in Firefox’s parent process using Firefox Developer Tools (left)\[/caption\]

You should now be able to find PromptParent.jsm (Ctrl+P) and set a debugger breakpoint for all future invocations (see screenshot above). This will allow you to inspect and copy the typical arguments passed to the Prompt JSActor in the parent.

Note: Once you hit a breakpoint, you can enter code into the Developer Console which is then executed within the currently intercepted function.

**DevTools Setup - Child Process**

Now that we know how to inspect and obtain the parameters which the parent process is expecting for `Prompt:Open`, let's try and trigger it from a debugged child process: Ensure you are on a typical web page, like [https://example.com](https://example.com), so you get the right kind of content child process. Then, through the Tools menu, find the "Browser Content Toolbox". Content here refers to the child process (Content Toolbox = Child Toolbox).

Since every content process might have many windows of the same site associated with it, we need to find the current window. This snippet assumes it is the first tab and gets the Prompt actor for that tab:

`actor = tabs[0].content.windowGlobalChild.getActor("Prompt");`

Now that we have the actor, we can use the data gathered in the parent process and send the very same data. Or maybe, a variation thereof:

`actor.sendQuery("Prompt:Open", {promptType: "alert", title: "👻", modalType: 1, promptPrincipal: null, inPermutUnload: false, _remoteID: "id-lol"});`

\[caption id="attachment\_225" align="aligncenter" width="1848"\]![Invoking JavaScript IPC from Firefox Developer Tools (bottom right) and observing the effects (top right)](/images/Bildschirmfoto-vom-2021-03-26-15-45-18.png) Invoking JavaScript IPC from Firefox Developer Tools (bottom right) and observing the effects (top right)\[/caption\]

In this case, we got away with not sending a reasonable value for `promptPrincipal` at all. This is certainly not going to be true for all message handlers. For the sake of this blog post, we can just assume that a _Principal_ is the implementation of an [Origin](https://html.spec.whatwg.org/multipage/origin.html#concept-origin) (and for background reading, we recommend an explanation of the _Principal_ Objects in our two-series blog post "Understanding Web Security Checks in Firefox": See [part 1](https://blog.mozilla.org/attack-and-defense/2020/06/10/understanding-web-security-checks-in-firefox-part-1/) and [part 2](https://blog.mozilla.org/attack-and-defense/2020/08/05/understanding-web-security-checks-in-firefox-part-2/)).

In case you wonder why the content process is allowed to send a potentially arbitrary _Principal_ (e.g., the origin): This is currently a known limitation and will be fixed while we are en route to full site-isolation ([bug 1505832](https://bugzilla.mozilla.org/show_bug.cgi?id=1505832)).

If you want to try to send another, faked origin - maybe from a different website or maybe the most privileged _Principal_ - the one that is bypassing all security checks, the _SystemPrincipal_, you can use these snippets to replace the _promptPrincipal_ in the IPC message:

```
const {Services} = ChromeUtils.import("resource://gre/modules/Services.jsm");
otherPrincipal = Services.scriptSecurityManager.createContentPrincipalFromOrigin("https://evil.test");
systemPrincipal = Services.scriptSecurityManager.getSystemPrincipal();
```

Note that validating the association between process and site is already [enforced in debug builds](https://searchfox.org/mozilla-central/rev/6309f663e7396e957138704f7ae7254c92f52f43/dom/ipc/ContentParent.cpp#1348). If you compiled your own Firefox, this will cause the content process to crash.

## **Revisiting Previous Security Issues**

Now that we have the setup in place we can revisit the security vulnerability mentioned above: [CVE-2019-11708](https://www.mozilla.org/en-US/security/advisories/mfsa2019-19/#CVE-2019-11708).

The issue in itself was a typical logic bug: Instead of switching which prompt to open in the parent process, the vulnerable version of this code accepted the URL to an internal prompt page, implemented as an XHTML page. But by invoking this message, the attacker could cause the parent process to open any web-hosted page instead. This allowed them to re-open their content process exploit again in the parent process and escalate to a full compromise.

Let's take a look at  the diff for the security fix to see how we replaced the vulnerable logic and handled the prompt type switching in the parent process ([permalink to source](https://searchfox.org/mozilla-central/diff/3075dbd453f011aaf378bcac6b2700dccfcf814c/browser/actors/PromptParent.jsm#141)).

\[caption id="attachment\_226" align="aligncenter" width="629"\]![Handling of untrusted message.data before and after fixing CVE-2019-11708.](/images/diff-prompt-coinbase-0day.png) Handling of untrusted `message.data` before and after fixing CVE-2019-11708.\[/caption\]

You will notice that line 140+ used to accept and use a parameter named `uri`. This was fixed in a multitude of patches. In addition to only allowing certain dialogs to open in the parent process we also generally [disallow opening web-URLs in the parent process](https://blog.mozilla.org/attack-and-defense/2020/07/07/hardening-firefox-against-injection-attacks-the-technical-details/).

If you want to try this yourself, download a version of Firefox before 67.0.4 and try sending a `Prompt:Open` message with an arbitrary URL.

## **Next Steps**

In this blog post, we have given an introduction to Firefox IPC using JavaScript and how to debug the child and the parent process using the Content Toolbox and the Browser Toolbox, respectively. Using this setup, you are now able to simulate a fully compromised child process, audit the message passing in source code and analyze the runtime behavior across multiple processes.

If you are already experienced with Fuzzing and want to analyze how high-level concepts from JavaScript get serialized and deserialized to pass the process boundary, please check our previous blog post on [Fuzzing the IPC layer of Firefox](https://blog.mozilla.org/attack-and-defense/2021/01/27/effectively-fuzzing-the-ipc-layer-in-firefox/).

If you are interested in testing and analyzing the source code at scale, you might also want to look into the [CodeQL databases](https://blog.mozilla.org/attack-and-defense/2020/05/25/firefox-codeql-databases-available-for-download/) that we publish for all Firefox releases.

If you want to know more about how our developers port legacy MessageManager interfaces to JSActors, you can take another look at our [JSActors documentation](https://firefox-source-docs.mozilla.org/dom/ipc/jsactors.html) and at how Mike Conley ported the popup blocker in his [Joy of Coding live stream Episode 204.](https://www.youtube.com/watch?v=mtIt6ir9GHU)

Finally, we at Mozilla are really interested in the bugs you might find with these techniques - bugs like confused-deputy attacks, where the parent process can be tricked into using its privileges in a way the content process should not be able to (e.g. reading/writing arbitrary files on the filesystem) or UXSS-type attacks, as well as [bypasses of exploit mitigations](https://www.mozilla.org/en-US/security/client-bug-bounty/#exploit-mitigation-bounty). Note that as of April 2021, we are not enforcing full site-isolation. Bugs that allow one to impersonate another site will _not_ yet be eligible for a bounty. Submit your findings through our [bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/#claiming-a-bounty) and follow us at the [@attackndefense](https://twitter.com/attackndefense) Twitter account for more updates.
