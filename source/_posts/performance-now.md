---
title: performance.now() - stuff I learned
date: 2018-11-20 07:47:57
tags: performance
---

Last week I was at the first edition of [performance.now()](https://perfnow.nl), a conference in Amsterdam focussed on web performance and organized by the same good folks as [CSS Day](https://cssday.nl) and [Mobilism](https://mobilism.nl). I learnt a ton about resource loading, rendering performance, protocols (hello, HTTP/3!), and much more, and I wanted to share some of it.

<!-- more -->

This isn't a comprehensive summary of the whole conference though. If you're interested in talk-by-talk summary you might want to read the excellent [write-up by Hidde de Vries](https://hiddedevries.nl/en/blog/). This is a collection of stuff I personally learned, in no particular order. Enjoy!

## Networks

As web developers we work at such a high level of abstraction we sometimes forget the layers of complexities. We should build our software as robust as possible, because networks _will_ fail your users, regularly. As [Tim Kadlec](https://twitter.com/tkadlec) put it in his talk, “we shouldn’t be surprised if anything fails, we should be surprised that anything ever works at all”.

Also nice to remember: even our abstract world of software is still bound to the plain ol' laws of physics. This slide by [Natasha Rooney](https://twitter.com/thisNatasha) really drove this home for me:

![Image of the United States, with an arrow pointing from San Francisco to Chicago](./5g.jpg)

Can you gues what it is? It's the distance light can travel in 10ms. And since data can't travel faster than the speed of light (at least until [quantum networks](https://en.wikipedia.org/wiki/Quantum_network) become a thing), this means that _even in theoretically perfect circumstances_ you're going to have latencies greater than 10ms if your data has to travel any further than that. '5G' is pictured because 10ms is the target latency for 5G networks between 2 points anywhere in the world (woops!).

## HTTP/3

"HTTP/3? I've just started to use HTTP/2 and now I have to learn a new thing already?" Well, yes and no. The things you know and love about HTTP aren't changing, but the way _they're transported_ will get an upgrade.

You might know that HTTP is built on top of TCP as its transport layer. You might also know that HTTP/2 has a shiney thing called _multiplexing_ which allows you to send multiple resources concurrently over a single connection. This is where TCP is slowing HTTP/2 down though, as TCP packets are always received in order and if a single packet is lost, the entire connection is blocked while that packet is being re-requested -- a problem called "head of line blocking". Even though all your styles and scripts can be sent simultanuously, they'll all be blocked if a single packet of your favicon is dropped.

Enter QUIC. QUIC is a new transport layer protocol, which solves this problem by establishing
multiplexed _connections_ between point A and B. This means that 2 resources being sent concurrently can be sent over 2 different connections, both unable to block the other. HTTP can run over QUIC instead of TCP, and they've decided to call HTTP-over-QUIC 'HTTP/3'. So there you have it.

QUIC was originally invented at Google and is available in Chrome right now. If you visit any Google property in Chrome your content will likely be served over QUIC:

![A screenshot of the Chrome DevTools network panel on the Google homepage](./quic.png)

## Preloading assets

I must have been living under a rock, because I completely missed [`link[rel=preload]`](https://developer.mozilla.org/en-US/docs/Web/HTML/Preloading_content) before. It's very useful to give hints to the browser to make sure it loads critical assets a quickly as possible. You can even use it with the [`Link` HTTP Header](https://www.w3.org/wiki/LinkHeader) to give hints before sending over the actual content. Use it to preload your critical CSS and fonts (and critical JS if you have any). Don't overuse it though, as that will mess up your browser's resource priorities.

## HTTP headers

Speaking of headers, here are some useful other ones I learnt about:

* [`Referrer-Policy`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy): a header to indicate what information gets leaked in the `Referer` header when linking to external properties. Kind of bummed they didn't stick with the 'referer' misspelling on this one though.
* [`Server-Timing`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Server-Timing): can be used to show backend timing metrics directly in the network panel of your browser's developer tools.
* [`Clear-Site-Data`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Clear-Site-Data): it's like the 'clear site data' button in your browser, but as an HTTP header. If sent, the browser will clear all relevant data for that website. Requires a fairly recent browser, though.
* [`Expect-CT`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Expect-CT): makes you less vulnerable to mistakenly issued certificates by letting browsers audit (and possibly report) your certificates using [Certficate Transparency](https://www.certificate-transparency.org/what-is-ct).
* [`Accept-CH`](https://httpwg.org/http-extensions/client-hints.html`): tells the browser to send along client hints (like `Viewport-Width` or `DPR`) with following requests, which you can use to serve appropriate -- optimized -- assets.
* [`Feature-Policy`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Feature-Policy): indicates what features (like autoplay, speaker, vibration) is the page is allowed to use. Make sure no script on your page can go rogue and start autoplaying video or requesting permissions it shouldn't need.
* [`Save-Data`](http://webconcepts.info/concepts/http-header/Save-Data): can be used to detect browser Data Saver mode, and to serve lighter or less assets accordingly.

We also got a list of headers that we should drop from our HTTP responses:

* [`Expires`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Expires): A caching header. It's almost always used in conjunction with `Cache-Control: max-age`, in which case it'll be completely ignored as per specification. Wasted bytes.
* [`X-Cache`](`https://anothersysadmin.wordpress.com/2008/04/22/x-cache-and-x-cache-lookup-headers-explained/`): A header typically sent by CDNs, but which can also be seen being sent by other servers. This is debugging information, only relevant to developers. We don't need to send it to our users.
* [`Via`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Via): used by proxies to track request forwards. Necessary on the backend to detect request loops, but again, we don't have to send it to our end users.
* [`X-Frame-Options`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options): used to indicate whether or not a page can be embed in an iframe. In itself it seems very useful, everything you want from this header can be achieved with a [Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy), which you should have anyway.
* [`P3P`](https://en.wikipedia.org/wiki/P3P): a header to indicate communicate a privacy policy using the P3P protocol. It was never supported outside of Internet Explorer and older versions of Edge, and even in those browsers the policy was never actually validated. Fun fact: the most frequently occuring value for this header is "This is not a P3P policy"; Internet Explorer required the header to be set for some purposes, but never actually validated the contents.

## Image optimizations

Use [progressive JPEGs](https://www.liquidweb.com/kb/what-is-a-progressive-jpeg/) to make images feel like they're loading faster. This works best when combined when you can actually send multiple images at the same time (like with HTTP/2 multiplexing).

You could also consider newer image formats. WebP has long been supported in Chrome and is now [slowly being adopted by browsers](https://www.zdnet.com/article/firefox-and-edge-add-support-for-googles-webp-image-format/), and it will shave rougly 30% off your images on average. [AVIF](https://en.wikipedia.org/wiki/AV1#AV1_Still_Image_File_Format_%28AVIF%29) is on the horizon though: a new open, royalty-free image format that promises to be about 50% smaller than JPEG. There's no actual browser support for it yet, but if you _really, really_ want it you can embed it in a `<video autoplay muted playsinline>` tag because it's technically a single frame of [AV1](https://en.wikipedia.org/wiki/AV1) video, which is supported by Firefox and Chromium-based browsers. But you really shouldn't.

## Font loading optimizations

Some quick tips for getting your text to render quickly:

* avoid web fonts where possible -- no font renders faster than a system font. Only load fonts where necessary to give your page personality; this is often not the case with body text.
* self host your fonts to reduce connection overhead and reliance on a third party.
* use [`font-display: swap`](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display) to prevent a flash of invisible text (FOIT)
* make sure your fallback font resembles your web font as much as possible to minimize the reflow when the font it loaded. Use [Font Style Matcher](https://meowni.ca/font-style-matcher/) to match up the font settings.
* use [`font-synthesis`](https://developer.mozilla.org/en-US/docs/Web/CSS/font-synthesis) to minimize the reflow of loading additional variants (weights, styles) of your fonts.

## Performance monitoring tools

Some performance tools worth mentioning:

* [WebPageTest](https://www.webpagetest.org/): very well known and mentioned in almost every single talk at the conference, but I don't use it often enough. This free tool lets you test the performance of your website from different places in the world, using real browsers and real connection speeds.
* [Request Map Generator](http://requestmap.webperf.tools/): takes WebPageTest output and turns it into a request map to get a better sense of how your CDNs and third parties are behaving. [This is an example of the output](http://requestmap.webperf.tools/render/150930_6C_8e18a8699be0287083cc5f121a0b18f4/).

## Conclusion

There are still lots of little things I learnt. Like, did you know `requestAnimationFrame()` is currently throttled at 60fps in browsers? Or that non-CORS-enabled ('opaque') external assets take up 7 _megabytes_(!) of storage in your ServiceWorker cache in Chrome for security reasons? All useful things to keep in mind when you're writing code.

All in all I had a great time at performance.now(). If you liked this post you might want to check out [the conference videos](https://www.youtube.com/playlist?list=PLjnstNlepBvMnKuNFvWeQWzlRp8Cbiw0X). Or better yet, maybe attend next year?
