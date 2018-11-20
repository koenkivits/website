---
title: performance.now() - stuff I learned
date: 2018-11-20 07:47:57
tags: performance
---

Earlier this month I attended the first edition of [performance.now()](https://perfnow.nl): a top-notch conference in Amsterdam focussed on web performance and organized by the same good folks as [CSS Day](https://cssday.nl) and [Mobilism](https://mobilism.nl). I learned a lot about resource loading, rendering performance, protocols _(hello, HTTP/3!)_, and much more, and here's my attempt at capturing some of it. Enjoy!

<!-- more -->

## Networks

When working on either frontend or backend you sometimes forget all the layers of complexity inbetween over which you have zero control. As [Tim Kadlec](https://twitter.com/tkadlec) put it in his talk, “we shouldn’t be surprised if anything fails, we should be surprised that anything ever works at all”. Build your applications and websites as robust as possible, as you can safely assume networks will fail your users all the time.

Also good to remember: even the fastest networks in the world are still bound to the laws of physics. This slide by [Natasha Rooney](https://twitter.com/thisNatasha) really drove this home for:

![Image of the United States, with an arrow pointing from San Francisco to Chicago](./5g.jpg)

Can you gues what it is? It's the distance light can travel in 10ms. Data can't travel faster than the speed of light, so that means that _even in theoretically perfect circumstances_ you're going to have latencies greater than 10ms if your data has to travel any further than that. '5G' is pictured because 10ms is the target latency for 5G networks between any two points anywhere in the world (woops!).

## HTTP/3

"HTTP/3? I've just started to use HTTP/2 and now I have to learn a new thing already?" Well, yes and no. The things you know and love about HTTP aren't changing, but the way _they're transported_ will get an upgrade.

You might know that HTTP is built on top of TCP as its transport layer. You might also know that HTTP/2 has a fancy thing called _multiplexing_ which allows you to send multiple resources concurrently over a single connection. This is where TCP is slowing HTTP/2 down though, as TCP packets are always received in-order and if a single packet is lost, the entire connection is blocked while that packet is being re-requested — a problem called "head of line blocking". Even though all your styles and scripts can be sent simultanuously, they'll all be blocked if a single packet of your favicon is dropped.

Enter QUIC. QUIC is a new transport layer protocol, which solves this problem by establishing several multiplexed _connections_ between point A and B. This means that 2 resources being sent concurrently can be sent over 2 different connections, both unable to block the other. HTTP can run over QUIC instead of TCP, and they've decided to call HTTP-over-QUIC 'HTTP/3'. So there you have it.

QUIC was originally invented at Google and is available in Chrome right now. If you visit any Google property in Chrome your content will likely be served over QUIC:

![A screenshot of the Chrome DevTools network panel on the Google homepage](./quic.png)

## Preloading assets

Make sure your important assets load faster by preloading them using [`link[rel=preload]`](https://developer.mozilla.org/en-US/docs/Web/HTML/Preloading_content). You can even use it with the [`Link` HTTP Header](https://www.w3.org/wiki/LinkHeader) to give hints before sending over the actual content. Use it to preload your critical CSS, JS, and fonts. Don't overuse it though, as that will mess up your browser's resource priorities.

## HTTP headers

Speaking of headers, here are some useful other ones to take a look at:

* [`Referrer-Policy`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy): used to indicate what information gets leaked in the `Referer` header when linking to external properties. I'm kind of bummed they didn't stick with the 'referer' misspelling though.
* [`Server-Timing`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Server-Timing): can be used to show backend timing metrics directly in the network panel of your browser's developer tools.
* [`Clear-Site-Data`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Clear-Site-Data): it's like the 'clear site data' button in your browser, but as an HTTP header. If sent, the browser will clear all relevant data for that website.
* [`Expect-CT`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Expect-CT): makes you less vulnerable to wrongly issued certificates by letting browsers audit your certificates using [Certficate Transparency](https://www.certificate-transparency.org/what-is-ct).
* [`Accept-CH`](https://httpwg.org/http-extensions/client-hints.html`): can be used to tell the browser to send along client hints (like `Viewport-Width` or `DPR`) with subsequent requests, which you can use to serve optimized assets.
* [`Feature-Policy`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Feature-Policy): indicates what features (like autoplay, speaker, vibration) the page is allowed to use. Use this to make sure no script on your page can go rogue and start autoplaying video or requesting permissions it shouldn't need.
* [`Save-Data`](http://webconcepts.info/concepts/http-header/Save-Data): a request header sent by browsers when Data Saver mode is active. You can use it to serve lighter assets.

We should avoid using the following headers, as they are of no use to your visitors and are therefore simply wasted bytes:

* [`Expires`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Expires): a caching header. It's almost always used in conjunction with `Cache-Control: max-age`, in which case it doesn't do anything.
* [`X-Frame-Options`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options): used to indicate whether or not a page can be embed in an iframe. Very useful, but this can also be achieved with a [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy), which you should have anyway.
* [`X-Cache`](`https://anothersysadmin.wordpress.com/2008/04/22/x-cache-and-x-cache-lookup-headers-explained/`): typically sent by CDNs. This is debugging information, only relevant to developers. We don't need to bother our users with it.
* [`Via`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Via): used by proxies to track request forwards. Necessary on the backend to detect request loops, but again, we don't have to send it to our end users.
* [`P3P`](https://en.wikipedia.org/wiki/P3P): a header to indicate communicate a privacy policy using the P3P protocol, which was never supported outside of older Microsoft browsers. Fun fact: the most frequently occuring value for this header is "This is not a P3P policy"; Internet Explorer required the header to be set for some security purposes, but never actually validated the contents.

## Image optimizations

Use [progressive JPEGs](https://www.liquidweb.com/kb/what-is-a-progressive-jpeg/) to make images feel like they're loading faster. This works best when combined when you can actually send multiple images at the same time (like with HTTP/2 multiplexing). [MozJPEG](https://github.com/mozilla/mozjpeg) outputs progressive JPEGs by default.

You can also consider using newer image formats. WebP has long been supported in Chrome and is now [slowly being adopted by browsers](https://www.zdnet.com/article/firefox-and-edge-add-support-for-googles-webp-image-format/), and it will shave rougly 30% off your images on average. [AVIF](https://en.wikipedia.org/wiki/AV1#AV1_Still_Image_File_Format_%28AVIF%29) is on the horizon though: a new open, royalty-free image format that promises to be about 50% smaller than JPEG. There's no actual browser support for it yet, but if you _really, really_ want it you can embed it in a `<video autoplay muted playsinline>` tag because it's technically a single frame of [AV1](https://en.wikipedia.org/wiki/AV1) video, which is supported by Firefox and Chromium-based browsers. But you really shouldn't.

## Font loading optimizations

Text is still the most important part of the web, so it's important to make sure your visitors have a nice reading experience.  Here are some quick tips for getting your text to render quickly and smoothly:

* use `link[rel=preload]` to preload your critical fonts
* avoid web fonts whenever possible — no font renders faster than a system font. More often than not your body text will look just fine using system fonts.
* self host your fonts to reduce connection overhead and reliance on a third party.
* use [`font-display: swap`](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display) to prevent a flash of invisible text _(FOIT)_
* make sure your fallback font resembles your web font as much as possible to minimize the reflow when the font it loaded. Use [Font Style Matcher](https://meowni.ca/font-style-matcher/) to match up the text CSS.
* use [`font-synthesis`](https://developer.mozilla.org/en-US/docs/Web/CSS/font-synthesis) to minimize the reflow of loading additional variants (weights, styles) of your fonts.

## Measuring performance

It's very hard to optimize when you don't know why your page is slow. Here are some tools to help you discover pain points:

* [WebPageTest](https://www.webpagetest.org/): very well known and mentioned in almost every single talk at the conference, but I don't use it often enough. This free tool lets you test the performance of your website from different places in the world, using real browsers and real connection speeds.
* [Request Map Generator](http://requestmap.webperf.tools/): takes WebPageTest output and turns it into a request map to get a better sense of how your CDNs and third parties are behaving. [This is an example of the output](http://requestmap.webperf.tools/render/150930_6C_8e18a8699be0287083cc5f121a0b18f4/).

There is also a wealth of information available in the [Performance API](https://developer.mozilla.org/en-US/docs/Web/API/Performance) that you can log to track your performance in the wild.

## Conclusion

There are still lots of other little things I learned. Like, did you know `requestAnimationFrame()` is currently throttled at 60fps in browsers? Or that non-CORS-enabled ('opaque') external assets take up 7 _megabytes_(!) of storage in your ServiceWorker cache in Chrome for security reasons? If you're interested in this sort of thing I can really recommend you to watch [the conference videos](https://www.youtube.com/playlist?list=PLjnstNlepBvMnKuNFvWeQWzlRp8Cbiw0X). Or even better: attend next year?
