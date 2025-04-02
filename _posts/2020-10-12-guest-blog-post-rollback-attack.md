---
layout: post
title: "Guest Blog Post: Rollback Attack"
author: Xiaoyin Liu
date: 2020-10-12
categories: 
  - "bug-bounty"
  - "guest-post"
  - "hack-and-tell"
---

 

_This blog post is the first of several guest blog posts we'll be publishing, where we invite participants of our [bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/) to write about bugs they've reported to us._

This blog post is about a vulnerability I found in the Mozilla Maintenance Service on Windows that allows an attacker to elevate privileges from a standard user account to SYSTEM. While the specific vulnerability only works on Windows, this is not really because of any Windows-specific issue but rather about how Mozilla validated trust in files it operated on with privileged components. This vulnerability is assigned [CVE-2020-15663](https://www.mozilla.org/en-US/security/advisories/mfsa2020-36/#CVE-2020-15663). It was reported on Mozilla Bugzilla [Bug 1643199](https://bugzilla.mozilla.org/show_bug.cgi?id=1643199).

One day I read the “Mozilla Foundation Security Advisory 2019-25,” and one bug caught my attention: “[CVE-2019-11753](https://www.mozilla.org/en-US/security/advisories/mfsa2019-25/#CVE-2019-11753): Privilege escalation with Mozilla Maintenance Service in custom Firefox installation location.” The description mentioned that a privilege escalation was caused “due to a lack of integrity checks.” My past experience taught me that maybe the fix was to check digital signatures only. If that’s the case, then a version rollback attack may be used to bypass the fix. So, I decided to check that, and it worked.

### **The Bug: A Classic Rollback Attack**

Firefox’s Windows installer allows users to customize the Firefox installation directory. If Firefox is installed to a non-standard location, and not the typical C:\\Program Files\\Mozilla Firefox\\, the Firefox installation path may be user-writable. If that is the case, then a local attacker with a standard account can replace any files in the installation path. If the attacker has permission to do this, they can already execute arbitrary code as the user - but they are interested in elevating privileges to SYSTEM.

The Mozilla Maintenance Service is a Windows service, installed by default. This service runs with SYSTEM privilege. One of its tasks is to launch the Firefox updater with SYSTEM privilege, so that it can update write-protected files under C:\\Program Files\\Mozilla Firefox without showing UAC prompts. The Mozilla Maintenance Service can be started by standard users, but its files are in C:\\Program Files (x86)\\Mozilla Maintenance Service\\ and _not_ writable by standard users.

However, the Maintenance Service copies files from the Firefox installation and then runs them with SYSTEM privileges: those files are updater.exe and updater.ini. Although the Maintenance Service checks if updater.exe contains an identity string and is signed by Mozilla, it doesn't check the file version. Thus, an attacker can replace the currently installed updater.exe with an old and vulnerable version of updater.exe. Then the Maintenance Service copies the old updater.exe and updater.ini to C:\\Program Files (x86)\\Mozilla Maintenance Service\\update\\, and runs the updater.exe in this update directory with SYSTEM privilege.

This type of bug is a classic rollback attack (sometimes called a replay attack). Most search results for this attack are focused on network protocols (especially TLS), but it’s also applicable to software updates. It was described for Linux package managers at least [ten years ago](https://www2.cs.arizona.edu/stork/packagemanagersecurity/attacks-on-package-managers.html). The general fix for this type of bug is to include in the signed metadata for a file when it should no longer be trusted. Debian uses a Valid-Until date in the metadata for the entire package repository; and [The Update Framework](https://theupdateframework.io/) is a specification for doing this in a more generic way.

Another example of a software rollback attack is a [Steam Service privilege escalation vulnerability](https://xiaoyinl.github.io/steam_EoP_bypass.html), CVE-2019-15315, which I discovered last year. The Steam Service vulnerability was also caused by only verifying digital signatures but not the file version.

### **Finding a Candidate Bug for Exploitation**

I looked for old bugs that may allow me to exploit this issue by [searching “Mozilla updater privilege escalation”](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=Mozilla+updater+privilege+escalation) on MITRE. Luckily, there were two matches: CVE-2017-7766 and CVE-2017-7760. The description of [CVE-2017-7766](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-7766) is “an attack using manipulation of ‘updater.ini’ contents, used by the Mozilla Windows Updater, and privilege escalation through the Mozilla Maintenance Service to allow for arbitrary file execution and deletion by the Maintenance Service, which has privileged access.” This looked very promising to me, because it only required manipulation of updater.ini contents, which standard users can fully control in our bug, so I decided to try this one.

### **Exploitation**

CVE-2017-7766 corresponds to [Bug 1342742](https://bugzilla.mozilla.org/show_bug.cgi?id=1342742) on Mozilla Bugzilla. Bug 1342742 is a combination of two bugs: the first is an arbitrary file execution by manipulating “updater.ini”; the second is arbitrary file overwriting with partially attacker controlled data.

The idea of CVE-2017-7766 is that they use the second bug to overwrite “updater.ini” in a way that can trigger the first bug to achieve arbitrary file execution. In our exploitation, we don’t need the arbitrary file overwriting bug, because our “updater.ini” is copied from the Firefox installation directory, which a standard user can already modify.

To exploit the first bug, we first need to change the value of “ExeRelPath” in updater.ini. The “ExeRelPath” is the path of an executable that will be launched after updater.exe successfully applies an update. According to the comments, “ExeRelPath” must be “in the same directory or a sub-directory of the directory of the application executable that initiated the software update.” However, the author of Bug 1342742 discovered that updater.exe accepts any file path, as long as we specify its absolute path. This executable file still needs be signed by Mozilla though, but we can overcome this restriction by using the DLL hijacking.

First, we create a new directory in the root of C:, e.g. C:\\poc. (Standard users can create new directories under C:\\, but not new files.) Then copy a Mozilla-signed exe file, such as crashreporter.exe, and the DLL we wish to inject to C:\\poc. It can be any other Mozilla-signed executable, as long as its Import Directory contains some library that is not a known DLL. Known DLLs are listed in Registry key HKLM\\SYSTEM\\CurrentControlSet\\Control\\Session Manager\\KnownDLLs. As far as I know, only installers of Mozilla products guarantee that they only load known DLLs at executable’s loading time, to prevent DLL hijacking in the Downloads folder.

To recap, the whole exploitation process is: a standard user starts the Mozilla Maintenance Service -> the Mozilla Maintenance Service runs old updater.exe -> the old updater.exe executes the file specified in ExeRelPath as SYSTEM e.g. crashreporter.exe -> crashreporter.exe loads the hijacked DLL located in the same directory. Thus, the local attacker can successfully escalate the privilege from standard user to SYSTEM.

### **Conclusion**

The Mozilla team [fixed this vulnerability](https://hg.mozilla.org/mozilla-central/rev/6a541f963216b68b88f16164bf11f93fdc7c9bd7) by only allowing the Mozilla Maintenance Service to update files under C:\\Program Files and C:\\Program Files (x86). In the [ExecuteServiceCommand](https://searchfox.org/mozilla-central/rev/89d33e1c3b0a57a9377b4815c2f4b58d933b7c32/toolkit/components/maintenanceservice/workmonitor.cpp#577) function, the function [now tests](https://searchfox.org/mozilla-central/rev/89d33e1c3b0a57a9377b4815c2f4b58d933b7c32/toolkit/components/maintenanceservice/workmonitor.cpp#640) if the install directory path, passed to the Maintenance Service as argv\[5\], is a Program Files path. Since files under Program Files are protected from standard users, they can no longer downgrade the updater.exe.

I want to thank the Mozilla security team for investigating and fixing this vulnerability and for the bug bounty. A special thank goes to Tom Ritter from Mozilla for his help with the bug bounty process, and for his help with this article. I also thank Holger Fuhrmannek, the reporter of CVE-2019-11753, and Seb Patane, the reporter of CVE-2017-7766, where I borrowed the idea for the exploitation.
