---
title: "Guest Blog Post: Firefox for Android LAN-Based Intent Triggering"
date: 2020-11-10
categories: 
  - "bug-bounty"
  - "guest-post"
  - "hack-and-tell"
---

 

_This blog post is one of several guest blog posts, where we invite participants of our [bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/) to write about bugs they've reported to us._

## Background

In its inception, Firefox for Desktop has been using one single process for all browsing tabs. Since Firefox Quantum, released in 2017, Firefox has been able to spin off multiple “content processes”. This revised architecture allowed for advances in security and performance due to sandboxing and parallelism. Unfortunately, Firefox for Android (code-named “Fennec”) could not instantly be built upon this new technology. Legacy architecture with a completely different user interface required Fennec to remain single-process. But in the fall of 2020, a [Fenix](https://github.com/mozilla-mobile/fenix) rose: The latest version of [Firefox for Android](https://www.mozilla.org/en-US/firefox/mobile/) is supported by [Android Components](https://github.com/mozilla-mobile/android-components), a collection of independent libraries to make browsers and browser-like apps that bring latest rendering technologies to the Android ecosystem.

For this rewrite to succeed, most work on legacy Android product (“Fennec”) was paused and put into maintenance mode except for high-severity security vulnerabilities.

It was during this time, coinciding with the general availability of the new Firefox for Android, that Chris Moberly from GitLab’s Security Red Team reported the following security  bug in the almost legacy browser.

## Overview

Back in August, I found a nifty little bug in the [Simple Service Discovery Protocol (SSDP)](https://en.wikipedia.org/wiki/Simple_Service_Discovery_Protocol) engine of Firefox for Android 68.11.0 and below. Basically, a malicious SSDP server could send Android Intent URIs to anyone on a local wireless network, forcing their Firefox app to launch activities without any user interaction. It was sort of like a magical ability to click links on other peoples' phones without them knowing.

I disclosed this directly to Mozilla as part of their [bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/) - you can read the original submission [here](https://bugzilla.mozilla.org/show_bug.cgi?id=1659381).

I discovered the bug around the same time Mozilla was already switching over to a completely rebuilt version of Firefox that did not contain the vulnerability. The vulnerable version remained in the Google Play store only for a couple weeks after my initial private disclosure. As you can read in the Bugzilla issue linked above, the team at Mozilla did a thorough review to confirm that the vulnerability was not hiding somewhere in the new version and took steps to ensure that it would never be re-introduced.

Disclosing bugs can be a tricky process with some companies, but the Mozilla folks were an absolute pleasure to work with. I highly recommend checking out their bug bounty program and seeing what you can find!

I did a short technical write-up that is available in the [exploit repository](https://gitlab.com/gitlab-com/gl-security/security-operations/gl-redteam/red-team-tech-notes/-/tree/master/firefox-android-2020). I'll include those technical details here too, but I'll also take this opportunity to talk a bit more candidly about how I came across this particular bug and what I did when I found it.

 

\[video width="1920" height="1080" mp4="http://blog.mozilla.org/attack-and-defense/files/2020/11/firefox-android-2020\_poc.mp4"\]\[/video\]

 

## Bug Hunting

I'm lucky enough to be paid to hack things at GitLab, but after a long hard day of hacking for work I like to kick back and unwind by... hacking other things! I'd recently come to the devastating conclusion that a bug I had been chasing for the past two years either didn't exist or was simply beyond my reach, at least for now. I'd learned a lot along the way, but it was time to focus on something new.

A friend of mine had been doing some cool things with Android, and it sparked my interest. I ordered a paperback copy of "[The Android Hacker's Handbook](https://www.wiley.com/en-us/Android+Hacker%27s+Handbook-p-9781118608647)" and started reading. When learning new things, I often order the big fat physical book and read it in chunks away from my computer screen. Everyone learns differently of course, but there is something about this method that helps break the routine for me.

Anyway, I'm reading through this book and taking notes on areas that I might want to explore. I spend a lot of time hacking in Linux, and Android is basically Linux with an abstraction layer. I'm hoping I can find something familiar and then tweak some of my old techniques to work in the mobile world.

I get to Chapter 2, where it introduces the concept of Android "Intents" and describes them as follows:

> "These are message objects that contain information about an operation to be performed... Nearly all common actions - such as tapping a link in a mail message to launch the browser, notifying the messaging app that an SMS has arrived, and installing and removing applications - involve Intents being passed around the system."

This really piqued my interest, as I've spent quite a lot of time looking into how applications pass messages to each other under Linux using things like Unix domain sockets and D-Bus. This sounded similar. I put the book down for a bit and did some more research online.

The Android developer documentation provides a great [overview on Intents](https://developer.android.com/guide/components/intents-filters). Basically, developers can expose application functionality via an Intent. Then they create something called an "Intent filter" which is the application's way of telling Android what types of message objects it is willing to receive.

For example, an application could offer to send an SMS to one of your contacts. Instead of including all of the code required actually to send an SMS, it could [craft an Intent](https://developer.android.com/guide/components/intents-common#Messaging) that included things like the recipient phone number and a message body and then send it off to the default messaging application to handle the tricky bits.

Something I found very interesting is that while Intents are often built in code via a complex function, they can also be expressed as URIs, or links. In the case of our text message example above, the Intent URI to send an SMS could look something like this:

[sms://8675309?body="hey%20there!”](sms://8675309?body="hey%20there!”)

In fact, if you click on that link while reading this on an Android phone, it should pop open your default SMS application with the phone number and message filled in. You just triggered an Intent.

![Preview of draft SMS](images/intent-preview.png)

It was at this point that I had my "_Hmmm, I wonder what would happen if I..._" moment. I don't know how it is for everyone else, but that phrase pretty much sums up hacking for me.

You see, I've had a bit of success in forcing desktop applications to visit URIs and have used this in the past to find exploitable bugs in targets like [Plex](https://seclists.org/fulldisclosure/2018/Aug/1), [Vuze](https://seclists.org/fulldisclosure/2018/Aug/2), and even [Microsoft Windows](https://initblog.com/2019/switcheroo/).

These previous bugs abused functionality in something called Simple Service Discovery Protocol (SSDP), which is used to advertise and discover services on a local network. In SSDP, an initiator will send a broadcast out over the local network asking for details on any available services, like a second-screen device or a networked speaker. Any system can respond back with details on the location of an XML file that describes their particular service. Then, the initiator will automatically go to that location to parse the XML to better understand the service offering.

And guess what that XML file location looks like...

... That's right, a URI!

In my previous research on SSDP vulnerabilities, I had spent a lot of time running Wireshark on my home network. One of the things I had noticed was that my Android phone would send out SSDP discovery messages when using the Firefox mobile application. I tried attacking it with all of the techniques I could think of at that time, but I never observed any odd behavior.

But that was before I knew about Intents!

So, what would happen if we created a malicious SSDP server that advertised a service's location as an Android Intent URI? Let's find out!

## What happened...

First, let's take a look at what I observed over the network prior to putting this all together. While running Wireshark on my laptop, I noticed repeated UDP packets originating from my Android phone when running Firefox. They were being sent to port 1900 of the UDP multicast address 239.255.255.250. This meant that any device on the local network could see these packets and respond if they chose to.

The packets looked like this:

```
M-SEARCH * HTTP/1.1
```

This is Firefox asking if there are any Roku devices on the network. It would actually ask for other device types as well, but the device type itself isn't really relevant for this bug.

It looks like an HTTP request, right? It is... sort of. SSDP uses two forms of HTTP over UDP:

- **HTTPMU**: HTTP over multi-cast UDP. Meaning the request is broadcast to the entire network over UDP. This is used for clients discovering devices and servers announcing devices.
- **HTTPU**: HTTP over uni-cast UDP. Meaning the request is sent from one host directly to another over UDP. This is used for responding to search requests.

If we had a Roku on the network, it would immediately respond with a UDP uni-cast packet that looks something like this:

```
HTTP/1.1 200 OK
CACHE-CONTROL: max-age=1800
DATE: Tue, 16 Oct 2018 20:17:12 GMT
EXT:
LOCATION: http://192.168.1.100/device-desc.xml
OPT: "http://schemas.upnp.org/upnp/1/0/"; ns=01
01-NLS: uuid:7f7cc7e1-b631-86f0-ebb2-3f4504b58f5c
SERVER: UPnP/1.0
ST: roku:ecp
USN: uuid:7f7cc7e1-b631-86f0-ebb2-3f4504b58f5c::roku:ecp
BOOTID.UPNP.ORG: 0
CONFIGID.UPNP.ORG: 1
```

Then, Firefox would query that URI in the `LOCATION` header (http://192.168.1.100/device-desc.xml), expecting to find an XML document that told it all about the Roku and how to interact with it.

This is where the experimenting started. What if, instead of http://192.168.1.100/device-desc.xml, I responded back with an Android Intent URI?

I grabbed my old SSDP research tool [evil-ssdp](https://github.com/initstring/evil-ssdp), made some quick modifications, and replied with a packet like this:

```
HTTP/1.1 200 OK
CACHE-CONTROL: max-age=1800
DATE: Tue, 16 Oct 2018 20:17:12 GMT
EXT:
LOCATION: tel://666
OPT: "http://schemas.upnp.org/upnp/1/0/"; ns=01
01-NLS: uuid:7f7cc7e1-b631-86f0-ebb2-3f4504b58f5c
SERVER: UPnP/1.0
ST: upnp:rootdevice
USN: uuid:7f7cc7e1-b631-86f0-ebb2-3f4504b58f5c::roku:ecp
BOOTID.UPNP.ORG: 0
CONFIGID.UPNP.ORG: 1
```

Notice in the `LOCATION` header above we are now using the URI `tel://666`, which uses the URI scheme for Android's default [Phone Intent](https://developer.android.com/guide/components/intents-common#Phone\)).

Guess what happened? The dialer on the Android device magically popped open, all on its own, with absolutely no interaction on the mobile device itself. It worked!

 

![SSDP Flow](images/ssdp-flow.png)

 

## PoC

I was pretty excited at this point, but I needed to do some sanity checks and make sure that this could be reproduced reliably.

I toyed around with a couple different Android devices I had, including the official emulator. The behavior was consistent. Any device on the network running Firefox would repeatedly trigger whatever Intent I sent via the `LOCATION` header as long as the PoC script was running.

This was great, but for a good PoC we need to show some real impact. Popping the dialer may look like a big scary Remote Code Execution bug, but in this case it wasn't. I needed to do better.

The first step was to write an exploit that was specifically designed for this bug and that provided the flexibility to dynamically provide an Intent at runtime. You can take a look at what I ended up with in this [Python script](https://gitlab.com/gitlab-com/gl-security/security-operations/gl-redteam/red-team-tech-notes/-/blob/master/firefox-android-2020/ffssdp.py).

The TL;DR of the exploit code is that it will listen on the network for SSDP "discovery" messages and respond back to them pretending to be exactly the type of device they are looking for. Instead of providing a link to an XML in the `LOCATION` header, I would pass in a user supplied argument, which can be an Android Intent. There's some specifics here that matter and have to be formatted exactly right in order for the initiator to trust them. Luckily, I'd spent some quality time reading the [IETF specs](https://tools.ietf.org/html/draft-cai-ssdp-v1-03) to figure that out in the past.

The next step was to find an Intent that would be meaningful in a demonstration of web browser exploitation. Targeting the browser itself seemed like a likely candidate. However, the [developer docs](https://developer.android.com/guide/components/intents-common#Browser) were not very helpful here - the URI scheme to open a web browser is defined as `http` or `https`. However, the native SSDP code in Firefox mobile would handle those URLs on its own and not pass them off to the Intent system.

A bit of searching around and I learned that there was another way to build an Intent URI while specifying the actual app you would like to send it to. Here's an example of an Intent URI to specifically launch Firefox and browse to example.com:

```
intent://example.com/#Intent;scheme=http;package=org.mozilla.firefox;end
```

If you'd like to try out the exploit yourself, you can grab an older version of Firefox for Android [here](https://archive.mozilla.org/pub/mobile/releases/68.11.0/).

To start, simply join an Android device running a vulnerable version of Firefox to a wireless network and make sure Firefox is running.

Then, on a device that is joined to the same wireless network, run the following command:

```
# Replace "wlan0" with the wireless device on your attacking machine.
python3 ./ffssdp.py wlan0 -t "intent://example.com/#Intent;scheme=http;package=org.mozilla.firefox;end"
```

All Android devices on the network running Firefox will automatically browse to example.com.

If you'd like to see an example of using the exploit to launch another application, you can try something like the following, which will open the mail client with a pre-filled message:

```
# Replace "wlan0" with the wireless device on your attacking machine.
python3 ./ffssdp.py wlps0 -t "mailto:itpeeps@work.com?subject=I've%20been%20hacked&body=OH%20NOES!!!"
```

These of course are harmless examples to demonstrate the exploit. A bad actor could send a victim to a malicious website, or directly to the location of a malicious browser extension and repeatedly prompt the user to install it.

It is possible that a higher impact exploit could be developed by looking deeper into the Intents offered inside Firefox or to target known-vulnerable Intents in other applications.

When disclosing vulnerabilities, I often feel a sense of urgency once I've found something new. I want to get it reported as quickly as possible, but I also try to make sure I've built a PoC that is meaningful enough to be taken seriously. I'm sure I don't always get this balance right, but in this case I felt like I had enough to provide a solid report to Mozilla.

## What came next

I worked out a public disclosure date with Mozilla, waited until that day, and then uploaded the exploit code with a short technical write-up and posted a link on Twitter.

To be perfectly honest, I thought the story would end there. I assumed I would get a few thumbs-up emojis from friends and it would then quickly fade away. Or worse, someone would call me out as a fraud and point out that I was completely wrong about the entire thing.

But, guess what? It caught on a bit. Apparently people are really into web browser bugs AND mobile bugs. I think probably it resonates as it is so close to home, affecting that little brick that we all carry in our pockets and share every intimate detail of our life with.

I have a Google "search alert" for my name, and it started pinging me a couple times a day with mentions in online tech articles all around the world, and I even heard from some old friends who came across my name in their news feeds. Pretty cool!

While I'll be the first to admit that this isn't some super-sophisticated, earth-shattering discovery, it did garner me a small bit of positive attention and in my line of work that's pretty handy. I think this is one of the major benefits of disclosing bugs in open-source software to organizations that care - the experience is transparent and can be shared with the world.

 

\[video width="1920" height="1080" mp4="http://blog.mozilla.org/attack-and-defense/files/2020/11/firefox-android-2020\_poc2.mp4"\]\[/video\]

_Another POC video created by [@LukasStefanko](https://twitter.com/LukasStefanko)_

 

## Lessons learned

This was the first significant discovery I've made both in a mobile application and in a web browser, and I certainly wouldn't consider myself an expert in either. But here I am, writing a guest blog with Mozilla. Who'd have thought? I guess the lesson here is to try new things. Try old things in new ways. Try normal things in weird things. Basically, just try things.

Hacking, for me at least, is more of an exploration than a well-planned journey. I often chase after random things that look interesting, trying to understand exactly what is happening and why it is happening that way. Sometimes it pays off right away, but more often than not it's an iterative process that provides benefits at unexpected times. This bug was a great example, where my past experience with SSDP came back to pay dividends with my new research into Android.

Also, read books! And while the book I mention above just happens to be about hacking, I honestly find just as much (maybe more) inspiration consuming information about how to design and build systems. Things are much easier to take apart if you understand how they are put together.

Finally, don't avoid looking for bugs in major applications just because you may not be the globally renowned expert in that particular area. If I did that, I would never find any at all. The bugs are there, you just have to have curiosity and persistence.

If you read this far, thanks! I realize I went off on some personal tangents, but hopefully the experience will resonate with other hackers on their own explorations.

With that, back to you Freddy.

## Conclusion

As one of the oldest [Bug Bounty Programs](https://www.mozilla.org/en-US/security/bug-bounty) out there, we really enjoy and encourage direct participation on Bugzilla. The direct communication with our developers and security engineers help you gain insight into browser development and allows us to gain new perspectives.

Furthermore, we will continue to have guest blog posts for interesting bugs here. If you want to learn more about our security internals, follow us on Twitter [@attackndefense](https://twitter.com/attackndefense) or subscribe to our [RSS feed](https://blog.mozilla.org/attack-and-defense/feed/).

Finally, thanks to Chris for reporting this interesting bug!
