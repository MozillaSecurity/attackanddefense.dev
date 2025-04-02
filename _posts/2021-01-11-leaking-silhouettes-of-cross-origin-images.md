---
title: "Guest Blog Post: Leaking silhouettes of cross-origin images"
date: 2021-01-11
categories: 
  - "bug-bounty"
  - "guest-post"
  - "hack-and-tell"
---

_This blog post is one of several guest blog posts, where we invite participants of our [bug bounty program](https://www.mozilla.org/en-US/security/client-bug-bounty/) to write about bugs they've reported to us._

This is a writeup of a vulnerability I found in Chromium and Firefox that could allow a malicious page to read some parts of an image located on an origin it is not supposed to be able to access. Although technically interesting, it is quite limited in scope—I am not aware of any major websites it could’ve been used against. As of November 17th, 2020, the vulnerability has been fixed in the most recent versions of both browsers.

### tl;dr

The time that it takes `CanvasRenderingContext2D.drawImage` to draw a pixel depends on whether it is fully transparent, opaque, or semi-transparent. By timing a bunch of calls to `drawImage`, we can reliably infer the transparency of each pixel in a cross-origin image, which is enough to, for example, read text on a transparent background, like this:

\[video autoplay="autoplay" width="320" height="200" webm="http://blog.mozilla.org/attack-and-defense/files/2021/01/exploit.webm"\]\[/video\]

  

### Background: Same-Origin Policy and Canvas

JavaScript running on a website is not allowed to read the contents of documents located on other websites. For example, your bank might have a page `https://mybank.com/get_balance/` that the application at `https://mybank.com/accounts/` should be able to read, but this blog certainly shouldn’t. This is formalized by the [Same-Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy), which allows resources that are located on a different origin from the current page to be embedded in it, but does not allow them to be read.

The embedding/reading distinction means that while I can, for example, include an image from a different origin on this page and your browser will display it for you, the page—and any JavaScript running on it—shouldn’t be allowed to learn the image’s contents. Unfortunately, there are side channels for the page to learn the existence or non-existence of an image at a URL, its size and dimensions, but if the contents of an image served from a well-known URL contain an authenticated user’s private information, the contents should remain private because any third-party page will only be able to display it back to the user—not read them.

Canvas is an HTML feature that makes implementing this distinction tricky. A `<canvas>` element is kind of like an image, but it can be manipulated from JavaScript—you can draw shapes, text, and other images onto it and, importantly, access individual pixels of the canvas for both reading and writing. Using the function `CanvasRenderingContext2D.drawImage`, any image that can be embedded onto a page can also be drawn onto a canvas, and this includes cross-origin images as well. In order to prevent an attacker from drawing a cross-origin image onto a canvas then reading the canvas to learn what the image looks like, a canvas that has ever had a cross-origin image drawn to it is marked as [tainted](https://developer.mozilla.org/en-US/docs/Web/HTML/CORS_enabled_image#Security_and_tainted_canvases) and can no longer be read from.

### Some pixels are drawn faster than others

Pixels in an image can have varying levels of transparency, and a well-optimized image drawing library might use different code to draw an image depending on what kind of pixels it contains. There are three cases that arise naturally:

- If a pixel is fully transparent, drawing it onto a canvas is very easy: you don’t need to do anything at all.
- If a pixel is fully opaque, it’s pretty easy too: overwrite the destination pixel with it.
- If the pixel is semi-transparent, you actually need to do some math to figure out the resulting color after blending it onto whatever’s already drawn on the canvas.

Conveniently for us, `drawImage` lets us both crop and resize the image that will be drawn, so we can make a call like `drawImage(img, x, y, 1, 1, 0, 0, 1024, 1024)` to take just the one pixel located at coordinates `(x, y)` in img, scale it up to `1024×1024`, and draw that onto our canvas.

If there’s any difference in performance between the three cases above, doing this will take a different amount of time depending on whether the given pixel is transparent, opaque, or semi-transparent, so by measuring the time the operation takes we can figure out which one it is. Repeat this for every pixel in img and you’ll have a “silhouette” of the image, which might be enough to figure out what the image contains, especially if it’s an image of text or a drawing on a transparent background.

Indeed, both Firefox and Chromium use Google’s [Skia Graphics Library](https://skia.org/), which used to handle the three cases separately in the function `blit_row_s32a_opaque`. A quick [benchmark](https://github.com/aleksejspopovs/cve-2020-16012/blob/main/benchmark/benchmark.html) I wrote confirmed that the performance of `drawImage` varied depending on the alpha-value of the pixels being drawn1:

![A bar chart titled “drawImage performance when drawing a pixel scaled to 1024×1024”, subtitled “Firefox 78.0.2 on Linux x86_64”. The x-axis is “Time to perform 1000 iterations, ms, average of 10 runs”. Five rows are visible: #000000, 0%: 46 ms 1%: 404 ms 50%: 401 ms 99%: 402 ms 100%: 252 ms](images/bench.png)

### Caveat: GPU acceleration

All of the above only applies when image's operations are performed using the CPU. When they are performed using GPU acceleration, no observable differences in performance seem to exist.

Firefox does not support GPU acceleration in all configurations yet: the majority of Windows users are accelerated in the common case, but other platforms are likely affected. Chromium will use GPU acceleration on most hardware, so it is only affected when GPU rendering is disabled, either because the user is using a known-bad GPU or driver, or because GPU acceleration has been explicitly disabled2.

### Exploitation

To exploit this, let’s write a function that takes one pixel of an image and measures how long it takes to draw it, blown up to 1024×1024 pixels, a hundred times:

```
const Iters = 100
let ScratchContext = document.createElement('canvas').getContext('2d')

function zeroDelay() {
  return new Promise(resolve => setTimeout(resolve, 0))
}

async function timePixel(image, x, y) {
  let startTime = performance.now()
  for (let j = 0; j < Iters; j++) {
    ScratchContext.drawImage(image, x, y, 1, 1, 0, 0, 1024, 1024)
  }
  /* in Chromium, the draw operations aren't actually performed
     immediately, but only after the JavaScript thread stops. we wait
     on a timeout with a duration of zero to give the browser a chance
     to do the drawing, as otherwise we'd just be measuring the time
     taken to enqueue all of the draw operations. */
  await zeroDelay()
  let endTime = performance.now()

  return endTime - startTime
}
```

Looking at the performance chart above, we know that, in an image where all pixels are either fully transparent or fully opaque, `timePixel` will return a much higher value for the opaque ones. Now we can measure all of the pixels one-by-one and render the timings on a heatmap, obtaining the Hollywood hacker movie-worthy reconstruction we saw in the video at the beginning of the post.

The full source code for the exploit is available [on my Github](https://github.com/aleksejspopovs/cve-2020-16012).

### Disclosure and vendor response

I reported this bug to Mozilla on May 29th, 2020 through the [Mozilla Security Bug Bounty program](https://www.mozilla.org/en-US/security/bug-bounty/) and to Google through the [Chrome Vulnerability Reward](https://www.google.com/about/appsecurity/chrome-rewards/) the next day. It took some time to figure out which graphics backend is used in Firefox by default these days. With the help of a Google engineer and some profiling tools, we identified that the same piece of Skia code was responsible for this behavior in both browsers.

Google [updated](https://skia.googlesource.com/skia/+/5d3314c53ce5c966591f0b02349103f51f986e6e) Skia to remove branching on alpha value in `blit_row_s32a_opaque` completely on August 29th, 2020 and [merged](https://chromium.googlesource.com/chromium/src.git/+/c000744b0540f5a5617f8c986fac70d026e03008) that change into Chromium on the same day. Mozilla [merged](https://hg.mozilla.org/mozilla-central/rev/48c0f5033c28) the change on October 6th, 2020.

Google has issued `CVE-2020-16012` to notify users about this bug.

Both vendors offered very generous bounties for my reports. It’s been a pleasure working with Mozilla and Google to get this fixed, and I would like to take this opportunity to thank Mike Klein from Google and Lee Salzman from Mozilla for their work on diagnosing and fixing the bug. I would also like to thank Tom Ritter and Lee Salzman from Mozilla for their helpful feedback on drafts of this blog post.

1 Although the behavior in Chromium is nearly identical, if you’re not careful about how you set up the benchmark, you might get a surprising result there: while the performance for fully and semi-transparent pixels is the same as in Firefox, fully solid pixels are drawn much faster. This is caused by an additional optimization that is only hit in Chromium, where Skia detects that an opaque 1024×1024 image is being drawn onto a canvas that’s exactly the same size, so instead of doing it pixel-by-pixel it just moves the whole buffer over, which turns out to be even faster than doing nothing 10242 times. It appears that this optimization is only triggered when the entire source image is opaque, not just the pixel we crop out of it. If you use a source image that has pixels with different alpha-values, Chromium performs exactly the same as Firefox.

2/ Though my Intel GPU is not on the blocklist, during my experiments I found that hammering it with thousands of draw commands will crash the driver and force Chromium to revert to CPU rendering, so this might be one way of exploiting the vulnerability on some hardware/OS combinations.
