---
title: "Sponsoring an RTSP Server Fuzzer"
date: 2020-06-15
---

The Secure Open Source track of the Mozilla Open Source Support (MOSS) Program primarily funds audits of open source projects. For example, we funded the [iTerm2 Audit](https://blog.mozilla.org/security/2019/10/09/iterm2-critical-issue-moss-audit/) which turned out to find a critical remote code execution flaw. But audits are not all we do for open source software security - besides sponsoring the remediation of issues identified during audits, we’ve been experimenting with funding other types of projects as well.

While evaluating open protocols and open source software for programs and ecosystems that could benefit from an investment, we considered the Real Time Streaming Protocol (RTSP) protocol which is used in browsers, media players, media editors, and other related software systems. Across the Internet, companies including Mozilla have been investing in automation to support software security, from static analysis to dynamic fuzzing, and at MOSS, we felt we could make an investment to that end as well.

Therefore, we sponsored some work by IncludeSec to write a protocol layer fuzzer for RTSP that was more comprehensive than what's currently out there in the Free/Open Source Software world. We asked them to focus on the server-side parts of RTSP because let's face it - server-side exploits are just more fun. Today they published that [fuzzer](https://github.com/IncludeSecurity/RTSPhuzz) and detailed some of the key points of consideration and usage [on their blog](https://blog.includesecurity.com/2020/06/RTSPhuzzer-Mozilla-Sponsored-Fuzzer-Release.html).

We hope the community uses and extends the fuzzer to improve the security of RTSP server implementations Internet-wide.
