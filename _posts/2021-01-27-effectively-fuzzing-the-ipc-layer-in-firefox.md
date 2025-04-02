---
title: "Effectively Fuzzing the IPC Layer in Firefox"
date: 2021-01-27
categories: 
  - "firefox-internals"
  - "security-internals"
tags: 
  - "fuzzing"
---

 

The **Inter-Process Communication (IPC)** **Layer** within Firefox provides a cornerstone in Firefox’ multi-process Security Architecture. Thus, eliminating security vulnerabilities within the IPC Layer remains critical. Within this blogpost we survey and describe the different communication methods Firefox uses to perform inter-process communication which hopefully provide logical entry points to effectively fuzz the IPC Layer in Firefox.

 

### Background on the Multi Process Security Architecture of Firefox

When starting the Firefox web browser it internally spawns one privileged process (also known as the parent process) which then launches and coordinates activities of multiple content processes. This multi process architecture allows Firefox to separate more complicated or less trustworthy code into processes, most of which have reduced access to operating system resources or user files. (Entering about:processes into the address bar shows detailed information about all of the running processes). As a consequence, less privileged code will need to ask more privileged code to perform operations which it itself cannot. That request for delegation of operations or in general any communication between content and parent process happens through the IPC Layer.

 

### IPC - Inter Process Communication 

From a security perspective the Inter-Process Communication (IPC) is of particular interest because it spans several security boundaries in Firefox. The most obvious one is the PARENT <-> CONTENT process boundary. The content (or child process), which hosts one or more tabs containing web content, is unprivileged and sandboxed and in threat modeling scenarios often considered to be compromised and running arbitrary attacker code. The parent process on the other hand has full access to the host machine. While this parent-child relationship is not the only security boundary (see e.g. [this documentation on process privileges](https://wiki.mozilla.org/index.php?title=Security/Sandbox)), it is the most critical one from a security perspective because any violation will result in a sandbox escape.

 

![](images/ipc_fuzzing-600x299.jpeg)

 

Firefox internally uses three main communication methods (plus an obsolete one) through which content processes can communicate with the main process. In more detail, inter process communication within processes in Firefox happen through either: (1) IPDL protocols, (2) Shared Memory, (3) JS Actors and sometimes through (4) the obsolete and outdated process communication mechanism of Message Manager. Please note that (3) and (4) internally are built on top of (1) and hence are IPDL aware.

1. **IPDL** The [**I**nter-process-communication **P**rotocol **D**efinition **L**anguage (IPDL)](https://wiki.mozilla.org/IPDL) is a Mozilla-specific language used to define the mechanism to pass messages between processes in C++ code.The primary protocol that governs the communication between the parent and child process in Firefox is the _Content_ protocol. As you can see in its source named [PContent.ipdl](https://searchfox.org/mozilla-central/source/dom/ipc/PContent.ipdl), the definition is fairly large and includes a lot of other protocols - because there are a number of sub-protocols in Content the whole protocol definition is hierarchic.Inside each protocol, there are methods for starting each sub-protocol and providing access to the methods inside the newly-started sub-protocol. For automated testing, this structure also means that there is not one flat attack surface containing all interesting methods. Instead, it might take a number of message round trips to reach a certain sub-protocol which then might surface a vulnerability.In addition to the PContent.ipdl Firefox also uses [PBackground.ipdl](https://searchfox.org/mozilla-central/source/ipc/glue/PBackground.ipdl) which is not included in Content but allows for background communication and hence provides another mechanism to pass messages between processes in C++.
2. **Shared Memory** [Shmems](https://wiki.mozilla.org/IPDL/Shmem) are the main objects to share memory across processes. They are IPDL-aware and are created and managed by IPDL actors. The easiest way to find them in code is to search for uses of ‘AllocShmem/AllocUnsafeShmem’, or to search all files ending in ‘.ipdl’. As an example, [PContent::InvokeDragSession](https://searchfox.org/mozilla-central/rev/96e2c6e14998f38e419850d55d8a3d32a3fc244a/dom/ipc/PContent.ipdl#701) uses PARENT <-> CONTENT Shmems to transfer drag/drop images, files, etc.An easier example, though not always involving the parent process, can be found in [PCompositorBridge::EndRecordingToMemory](https://searchfox.org/mozilla-central/rev/96e2c6e14998f38e419850d55d8a3d32a3fc244a/gfx/layers/ipc/PCompositorBridge.ipdl#288), which is used to record the browser display for things like collecting visual performance metrics. It returns a [CollectedFramesParams](https://searchfox.org/mozilla-central/rev/96e2c6e14998f38e419850d55d8a3d32a3fc244a/gfx/layers/ipc/PCompositorBridgeTypes.ipdlh#13) struct which contains a Shmem. First, the message is [sent by a PCompositorBridgeChild actor](https://searchfox.org/mozilla-central/rev/96e2c6e14998f38e419850d55d8a3d32a3fc244a/dom/base/nsDOMWindowUtils.cpp#4331) to a GPU process’ PCompositorBridgeParent, where the Shmem is [allocated, filled as a char\* and returned](https://searchfox.org/mozilla-central/rev/96e2c6e14998f38e419850d55d8a3d32a3fc244a/gfx/layers/ipc/CompositorBridgeParent.cpp#3021). It is then interpreted by the child actor as a buffer of raw bytes, using a Span object.Please note that the above example does not necessarily involve the parent process because it illustrates two of the more subtly confusing aspects of the process design. First, PCompositorBridge actors exist as both GPU <-> PARENT actors for browser UI rendering, called "chrome" in some parts of documentation, and GPU <-> CONTENT actors for page rendering. ([The comment for PCompositorBridge](https://searchfox.org/mozilla-central/rev/96e2c6e14998f38e419850d55d8a3d32a3fc244a/gfx/layers/ipc/PCompositorBridge.ipdl#71) explains this relationship in more detail.) Second, the GPU process does not always exist. For example, there is no GPU process on Mac based systems. This does not mean that these actors are not used -- it just means that when there is no GPU process, the parent process serves in its place.Finally, SharedMemoryBasic is a lower-level shared memory object that does not use IPDL. Its use is not well standardized but it is heavily used in normal operations, for example by SourceSurfaceSharedData to share texture data between processes.
3. **JSActors** The preferred way to perform IPC in JavaScript is based on [JS Actors](https://firefox-source-docs.mozilla.org/dom/ipc/jsactors.html) where a _JSProcessActor_ provides a communication channel between a child process and its parent and a _JSWindowActor_ provides a communication channel between a frame and its parent. After registering a new JS Actor it provides two methods for sending messages: (a) sendAsyncMessage() and (b) sendQuery(), and one for receiving messages: receiveMessage().
4. **Message Manager** In addition to JS Actors there is yet another method to perform IPC in JavaScript, namely the Message Manager. Even though we are replacing IPC communication based on Message Manager with the more robust JS Actors, there are still instances in the codebase which rely on the Message Manager. Its general structure is very loose and there is no registry of methods and also no specification of types. MessageManager messages are sent using the following four JavaScript methods: (a) sendSyncMessage(), (b) sendAsyncMessage(), (c) sendRpcMessage(), and (d) broadcastAsyncMessage().Enabling MOZ\_LOG=”MessageManager:5” will [output all of the messages](https://searchfox.org/mozilla-central/source/dom/ipc/MMPrinter.cpp) passed back and forth the IPC Layer. MessageManager is built atop IPDL using [PContent::SyncMessage](https://searchfox.org/mozilla-central/rev/16d30bafd4e5276d6d3c632fb52a6c71e739cc44/dom/ipc/PContent.ipdl#1037-1038) and [PContent::AsyncMessage](https://searchfox.org/mozilla-central/rev/16d30bafd4e5276d6d3c632fb52a6c71e739cc44/dom/ipc/PContent.ipdl#1713).

 

### Fuzz Testing In-Isolation vs. System-Testing

For automated testing, in particular fuzzing, isolating the components to be tested has proven to be effective many times. For IPC, unfortunately this approach has not been successful because the interesting classes, such as the _ContentParent_ IPC endpoint, have very complex runtime contracts with their surrounding environment. Using them in isolation results in a lot of false positive crashes. As an example, compare [the libFuzzer target for ContentParent](https://searchfox.org/mozilla-central/source/dom/ipc/fuzztest/content_parent_ipc_libfuzz.cpp), which has found some security bugs, but also an even larger number of (mostly nullptr) crashes that are related to missing initialization steps. Reporting false positives not only lowers the confidence in the tool’s results and requires additional maintenance but also indicates some missing parts of the attack surface because of an improper set up. Hence, we believe that a system testing approach is the only viable solution for comprehensive IPC testing.

One potential approach to fuzz such scenarios effectively could be to start Firefox with a new tab, navigate to a dummy page, then perform a snapshot of the parent (process- or VM-based snapshot fuzzing) and then replace the regular child messages with coverage-guided fuzzing. The snapshot approach would further allow to reset the parent to a defined state from time to time without suffering the major performance bottleneck of restarting the process. As described above in the IPDL section, it is crucial to have multiple messages going back and forth to ensure that we can reach deep into the protocol tree. And finally, the reproducibility of the crashes is crucial, since bugs without reliable steps to reproduce usually receive a lot less traction. Put differently, vulnerabilities with reliable steps to reproduce can be isolated, addressed and fixed a lot faster.

For VM-based snapshot fuzzing, we are aware of certain requirements that need to be fulfilled for successful Fuzzing. In particular:

- **_Callback when the process is ready for fuzzing_** In order to know when to start intercepting and fuzzing communication, some kind of callback must be made at the right point when things are “ready”. As an example, at the point when a new tab becomes ready for browsing, or more precisely has loaded its URI, that might be a good entrypoint for PARENT <-> CONTENT child fuzzing. One way to intercept the message exactly at this point in time might be to check for a STATE\_STOP message with the expected URI that you are loading, e.g. in [OnStateChange()](https://searchfox.org/mozilla-central/rev/16d30bafd4e5276d6d3c632fb52a6c71e739cc44/dom/ipc/BrowserParent.cpp#2614).

- **Callback for when communication sockets are created** On Linux, Firefox uses UNIX Sockets for IPC communication. If you want to intercept the creation of sockets in the parent, the [ChannelImpl::CreatePipe](https://searchfox.org/mozilla-central/rev/16d30bafd4e5276d6d3c632fb52a6c71e739cc44/ipc/chromium/src/chrome/common/ipc_channel_posix.cc#204) method is what you should be looking for.

 

### Examples of Previous Bugs

We have found (security) bugs through various means, including static analysis, manual audits, and libFuzzer targets on isolated parts (which has the problems described above). Looking through the reports of those bugs might additionally provide some useful information:

- [Heap-use-after-free in PNecko](https://bugzilla.mozilla.org/show_bug.cgi?id=1544526)
- [Various issues in PHal](https://bugzilla.mozilla.org/buglist.cgi?quicksearch=1469914%2C1469309%2C1465898)
- [Arbitrary file listing in PContent](https://bugzilla.mozilla.org/show_bug.cgi?id=1459206)
- [Out-of-bounds read in PContent through arbitrary icon size](https://bugzilla.mozilla.org/show_bug.cgi?id=1456975)
- [Heap-use-after-free related to PPrinting](https://bugzilla.mozilla.org/show_bug.cgi?id=1451376)
- [Out-of-bounds read in URL Classifier through PContent](https://bugzilla.mozilla.org/show_bug.cgi?id=1392739)

 

### Further Fuzzing related resources for Firefox

The following resources are not specifically for IPC fuzzing but might provide additional background information and are widely used at Mozilla for fuzzing Firefox in various ways:

- [prefpicker](https://github.com/MozillaSecurity/prefpicker) - Compiles prefs.js files used for fuzzing Firefox
- [fuzzfetch](https://github.com/MozillaSecurity/fuzzfetch/) - A tool to download all sorts of Firefox or JS shell builds from our automation
- [ffpuppet](https://github.com/MozillaSecurity/ffpuppet/) - Makes it easier to automate browser profile setup, startup etc.
- [Fuzzing Interface](https://firefox-source-docs.mozilla.org/tools/fuzzing/fuzzing_interface.html) - Shows how libFuzzer targets, instrumentation, etc. work in our codebase
- [Sanitizers](https://firefox-source-docs.mozilla.org/tools/sanitizer/index.html) - How to build with various sanitizers, known issues, workarounds, etc.

 

### Going Forward

Providing architectural insights into the security design of a system is crucial for truly working in the open and ultimately allows contributors, hackers, and bug bounty hunters to verify and  challenge our design decisions. We would like to point out that bugs in the IPC Layer are eligible for a [bug bounty](https://www.mozilla.org/en-US/security/client-bug-bounty/) -- you can report potential vulnerabilities simply by filing a [bug on Bugzilla](https://bugzilla.mozilla.org/form.client.bounty). Thank you!
